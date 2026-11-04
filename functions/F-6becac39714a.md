<!-- Trusted provenance: F-6becac39714a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6becac39714a

## Purpose

F-6becac39714a conditionally replaces two control references, performs up to two link-field writes, conditionally advances a tracked reference through a byte-filtered chain, and clears the supplied record if execution reaches the final stage.

The apparent intent is removal of a record from a linked structure, including maintenance of endpoint and traversal references. That intent is inferred. The established behavior is the ordered memory access contract below; it does not require a particular container interpretation.

Completion is not guaranteed. In particular, the reconnection writes occur before traversal and can themselves create a cycle that prevents the final record clear.

Two address interpretations must remain distinct:

- Reconnection and traversal-link accesses use addresses relative to the supplied record address.
- Traversal flags use the current traversal value directly as an address, with a byte displacement of 30.

No supplied invariant makes these address interpretations equivalent.

## Interface

The function accepts three inputs and returns no value.

| Input | Representation and role |
|---|---|
| Control address | Address of mutable storage containing three consecutive 32-bit unsigned fields |
| Record address | Address of mutable storage containing two consecutive 32-bit unsigned fields |
| Designated value | A 32-bit unsigned value used for comparisons and address displacement |

The control fields are:

- **Control A:** byte offset 0.
- **Control B:** byte offset 4.
- **Tracked reference:** byte offset 8.

The record fields are:

- **Record A:** byte offset 0.
- **Record B:** byte offset 4.

These names designate storage locations, not snapshots of entry values. A later read can observe an earlier write through overlapping storage.

Additional accesses use computed addresses. The represented field width is 32 bits; traversal flags are unsigned 8-bit quantities. Byte order is not independently established, and the function does not define a portable serialized representation.

Successful execution requires all reached reads and writes to address valid storage with suitable permissions and alignment. This requirement includes computed locations, not just the two input storage regions. No validation is performed.

No non-aliasing restriction is established. Control storage, record storage, and computed targets may overlap, with consequences determined by execution order. Concurrency, atomicity, and synchronization guarantees are not established.

## Behavior

### Address interpretation and arithmetic

For a stored 32-bit value \(v\), define its **relative address** as the supplied record address plus \(v\), minus the designated value. Its **companion address** is four bytes beyond that relative address.

The represented calculations use 32-bit address arithmetic. Unsigned displacement arithmetic wraps modulo \(2^{32}\); no range rejection or overflow check is present. The conversion of the supplied record address into that arithmetic is an execution-model dependency, not proof that every computed address is valid.

For a nonzero traversal value \(v\), its **flag address** is the direct numerical address \(v + 30\), using the represented 32-bit arithmetic. This calculation does not incorporate the supplied record address or subtract the designated value.

A nonzero stored value can select a zero, wrapped, inaccessible, or otherwise invalid address. Conversely, the function tests stored values for zero, not computed addresses.

### Conditional control replacements

The first control comparison reads Control A.

- If it equals the designated value, the function reads Record A and writes that value to Control A.
- Otherwise, this stage performs no Control A write.

The next comparison reads Control B.

- If it equals the designated value, the function reads Record B and writes that value to Control B.
- Otherwise, this stage performs no Control B write.

The Control B comparison follows the possible Control A write. Overlap can therefore change the value compared or copied in the second operation.

Neither record field is unconditionally captured at entry. These replacements use the record values read when their corresponding assignments are reached.

### Reconnection writes

After the control replacements, the function tests Record B for zero.

If Record B is nonzero, it writes the current Record A value to the relative address selected by Record B. This is a 32-bit write. A zero Record B suppresses this write.

The function then tests Record A for zero.

If Record A is nonzero, it writes the current Record B value to the companion address selected by Record A. This is also a 32-bit write. A zero Record A suppresses this write.

The first reconnection write precedes the second test and write. If its target overlaps relevant storage, it can affect the second operation. The guard reads do not establish that the record fields remain cached for all subsequent uses.

The source value and computed destination within an individual assignment have no independently established operand-evaluation order. The contract specifies the order between these operations, not a finer bus-level order of their constituent accesses.

These writes modify the memory that subsequent traversal may read. Traversal is therefore over the resulting memory state, not necessarily the chain that existed at entry.

### Conditional tracked-reference traversal

Only after both reconnection stages does the function read the tracked reference.

If it differs from the designated value, the function skips traversal and performs no tracked-reference write at this stage.

If it equals the designated value, the function reads Record A as the initial traversal value. Traversal has the following externally relevant rules:

- A zero traversal value terminates immediately, without reading a flag or link for that value.
- A nonzero traversal value causes an unsigned one-byte read at its flag address.
- A zero flag terminates traversal at that nonzero value, without reading its relative link field.
- A nonzero flag causes a 32-bit read at that value’s relative address. The fetched value becomes the next traversal value.

All flag values from 1 through 255 have the same continuation behavior.

When traversal terminates, the function writes the terminating value to the tracked reference. This is either zero or the first encountered nonzero value whose directly addressed flag byte is zero.

There is no length bound, visited-record tracking, or cycle detection. A reachable cycle of nonzero values whose flag bytes remain nonzero prevents termination. Such a cycle may already exist or may result from the preceding reconnection writes.

No final tracked-reference value is written while traversal continues. Earlier control and reconnection writes have already occurred.

### Final record clear and return

The final stage is reached only if traversal was skipped or terminated, and no earlier access prevented further execution.

The function then:

1. Writes a 32-bit zero to Record B.
2. Writes a 32-bit zero to Record A.
3. Returns without a result.

This is not an atomic two-field clear. An interruption between the writes can leave only Record B cleared.

Under stable single-threaded execution, successful return leaves both record fields zero. This remains true when either field also occupies a control location: the final record writes can overwrite earlier control results.

No earlier stage guarantees that the final stage will be reached.

## Side effects

The function may write to:

| Location | Condition |
|---|---|
| Control A | Its comparison equals the designated value |
| Control B | Its comparison equals the designated value |
| Relative address selected by Record B | Record B is nonzero at its guard |
| Companion address selected by Record A | Record A is nonzero at its guard |
| Tracked reference | Its comparison matches and traversal terminates |
| Record B, then Record A | Execution reaches the final clear |

These logical locations need not be physically distinct. Consequently, entry-state summaries are insufficient for arbitrary aliasing. A computed write can change control storage, the supplied record, traversal links, or flags that will be inspected later.

The body contains no explicit allocation, release, ownership transfer, destructor invocation, or memory reclamation. Clearing the record does not establish that its backing storage has been freed or may safely be reclaimed.

There is no explicit error result, diagnostic, rollback, or exception handler. An invalid access may prevent completion after earlier writes have taken effect; the precise fault behavior belongs to the execution environment.

Nonterminating traversal prevents the tracked-reference assignment and both final clearing writes. It does not undo earlier mutations.

## Dependencies

No other function invocation is present in the supplied body. There are no opaque callee dependencies to enumerate.

Execution depends on:

- Readable and writable control and record storage for the accesses reached.
- Writable computed targets for reconnection writes that are enabled.
- Readable direct-address flag bytes for each nonzero traversal candidate reached.
- Readable relative link fields for candidates whose flags are nonzero.
- The execution environment’s address conversion, 32-bit arithmetic, alignment, and fault rules.

No allocator, ownership manager, synchronization service, or external validation mechanism is established.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed test results. Unless an example states otherwise, assume stable single-threaded memory, valid reached accesses, and no overlap among control storage, supplied record storage, and computed fields.

### No control reference matches

Assume Control A, Control B, and the tracked reference all differ from the designated value.

Expected behavior:

- The function performs neither conditional control replacement.
- Each reconnection write is governed by its corresponding record-field zero test.
- No traversal flag or traversal link is read.
- Record B is cleared before Record A, and the function returns.

### Both endpoint controls match, with traversal skipped

Assume Control A and Control B equal the designated value, the tracked reference differs from it, and Record A and Record B initially hold \(a\) and \(b\), respectively.

Expected behavior:

- Control A receives \(a\).
- Control B receives \(b\).
- If \(b\) is nonzero, the relative field selected by \(b\) receives \(a\).
- If \(a\) is nonzero, the companion field selected by \(a\) receives \(b\).
- Traversal is skipped.
- Record B and then Record A are cleared.

The tracked-reference assumption is material. Matching endpoints and valid, non-overlapping storage alone do not guarantee the final clear.

### Reconnection creates a nonterminating self-cycle

Assume both record fields contain the same nonzero value \(a\). Let the relative field and companion field selected by \(a\) be valid, separate from the control and supplied record storage. Assume the tracked reference equals the designated value and the byte at the direct flag address for \(a\) is nonzero.

Expected behavior:

- The first reconnection write stores \(a\) in the relative field selected by \(a\).
- The second reconnection write stores \(a\) in its companion field.
- Traversal starts at \(a\).
- Each traversal step reads a nonzero flag and then retrieves \(a\) again from the relative link.
- In the absence of external interruption or mutation, traversal does not terminate.
- No terminating tracked-reference write occurs.
- Neither final record-clearing write occurs.

Control A and Control B may also equal the designated value in this example; their replacements do not eliminate the cycle. This demonstrates why completion cannot be inferred merely from valid, stable, non-overlapping storage.

### Matching tracked reference with no continuation

Assume the tracked reference equals the designated value and Record A is zero when traversal begins.

Expected behavior:

- No traversal flag byte or traversal link field is read.
- The tracked reference receives zero.
- Record B is then cleared, followed by Record A.

### First nonzero candidate is retained

Assume traversal begins at nonzero \(a\), and the byte at direct address \(a + 30\) is zero.

Expected behavior:

- That byte is read.
- No traversal link read for \(a\) occurs.
- The tracked reference receives \(a\).
- The final record clear follows.

A zero byte at the relative address plus 30 does not satisfy this condition unless it is also the required direct-address location.

### Traversal skips a candidate

Assume traversal begins at nonzero \(a\). Its directly addressed flag is nonzero, its relative link contains nonzero \(b\), and the directly addressed flag for \(b\) is zero. These conditions apply to memory after reconnection.

Expected behavior:

- The flag for \(a\) is read.
- The relative link for \(a\) is read.
- The flag for \(b\) is read.
- No traversal link for \(b\) is read.
- The tracked reference receives \(b\), followed by the final record clear.

### A relative link yields zero

Assume a reached nonzero candidate has a nonzero flag and its relative link contains zero.

Expected behavior:

- The link read produces the terminating value.
- No flag byte at address 30 is read on behalf of that zero value.
- The tracked reference receives zero.
- The final record clear follows.

### Zero designated value

Assume the designated value, all three control fields, and both record fields are zero.

Expected behavior:

- Both control comparisons match, and both conditional replacements write zero.
- Neither reconnection write occurs.
- The tracked-reference comparison matches.
- Traversal terminates immediately without a flag or link read.
- The tracked reference receives zero, and both final record writes occur.

Zero is not prohibited as the designated value. Its role in equality comparisons is separate from the zero checks that suppress reconnection and terminate traversal.

## Uncertainties

### Inferred intent

The paired reconnection writes and endpoint replacements suggest unlinking a record. The traversal suggests skipping records marked by a nonzero byte.

Neither interpretation establishes a concrete record type, flag meaning, structural invariant, or ownership rule.

### documented_only: execution representation

The supplied representation supports 32-bit fields and address calculations, but does not independently establish the platform ABI, native byte order, pointer provenance rules, alignment enforcement, or exact invalid-access behavior.

### documented_only: relationship between address forms

No invariant establishes equality between the numerical supplied record address and the designated value. Relative link access and direct flag access must therefore remain separate in any implementation or acceptance model.

### documented_only: aliasing and concurrency

No disjointness, thread-safety, reentrancy, or atomicity guarantee is provided. Sequential overlap effects can be described using the established operation order. Supported semantics for concurrent mutation are not established.

### documented_only: lifetime and structural validity

The body does not establish allocation extents, permission to reclaim storage, consistent reciprocal links, acyclic traversal, or eventual discovery of a zero flag.

Even initially acyclic traversal data is not by itself a termination guarantee: reconnection changes link memory before traversal starts.

## Evidence

Established static evidence from the supplied body of F-6becac39714a comprises:

- Two ordered conditional control replacements.
- Two ordered, zero-guarded 32-bit reconnection writes.
- A tracked-reference comparison performed after those writes.
- Optional traversal using direct-address one-byte flags and relative-address 32-bit links.
- A tracked-reference assignment only after traversal terminates.
- Final zero writes to record offset 4 and then record offset 0.
- No explicit calls, validation, allocation, release, rollback, traversal bound, or cycle detection.

The self-cycle acceptance example follows from the first reconnection write storing the shared nonzero record value into that same value’s relative link. The later traversal reads the modified link and can repeat indefinitely.

Structural purpose is inferred; ordered memory behavior is statically established subject to the representation limits above. No runtime tests have been performed.
