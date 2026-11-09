<!-- Trusted provenance: F-04595dfeaf6e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-04595dfeaf6e

## Purpose

F-04595dfeaf6e performs accounting, repeatedly removes entries from a context-linked structure, updates entry-associated state, and selects either a fixed-width transfer or a dependency call. It returns a 32-bit zero after observing an empty pending-entry head and completing the terminal calls required by the observed state.

Established behavior includes several distinctions from a conventional queue drain:

- An initial dependency call occurs before accounting or inspection of the pending head, including when the width is zero.
- Accounting occurs before every pending-head inspection, including the inspection that terminates processing.
- Removing an entry increments the context count rather than decrementing it.
- Clearing flag bit `0x80` causes insertion into a separate chain only if the entire resulting flag word is zero.
- The initial destination calculation multiplies the width by the numeric context address. Subsequent dependency-based transfers multiply it by the updated index.
- A width of zero suppresses the per-entry transfer, not entry processing or the initial dependency call.

“Queue,” “recycling,” “capacity,” and “high-water mark” are inferred interpretations, not established object types, ownership rules, or bounds guarantees.

## Interface

### Explicit inputs and result

The supplied representation exposes two explicit inputs:

| Input | Contract |
|---|---|
| Context address | Addresses the context fields described below. |
| Initial descriptor address | Its 32-bit field at byte offset +16 supplies an argument to the initial dependency call. |

The normal result is a 32-bit zero. No alternative returned status, validation error, or recovery path is visible. Return requires the selected dependencies to return normally.

The initial descriptor is not necessarily the descriptor used after an entry is removed. Each removal obtains a replacement descriptor through the entry’s associated object.

### Implicit operands

The function also depends on machine-state operands whose initialization is not supplied:

| Operand | Observed use |
|---|---|
| Width/multiplier | Initial address calculation, transfer selection, and later destination calculations |
| Chain-state address | A 32-bit field at +8 and a byte at +1 |
| Comparison-state address | Two 32-bit fields at +8 and +12 |
| Reset value | Replacement value for entry, descriptor, and sometimes index fields |
| Wrap threshold | Equality comparison against the incremented index |

The reset value is not established to be zero. The width is not established to remain constant across dependency calls. No dependency preservation rule for these operands is supplied.

### Widths and layouts

Offsets are in bytes. Ordinary fields and address-bearing values are represented as 32-bit quantities. The exceptional byte access and the direct transfer widths are identified separately.

**Context**

| Offset | Meaning established by use |
|---|---|
| +0 | Destination base value |
| +12 | Index, incremented per removed entry and conditionally replaced |
| +16 | Count, sampled during accounting and incremented per removed entry |
| +20 | Pending-entry head |
| +32 | Condition field, read when the sampled count equals one |
| +44 | Accounting counter, incremented before each pending-head inspection |
| +48 | Stored maximum, compared with the count as unsigned |

**Entry**

| Offset | Meaning established by use |
|---|---|
| +0 | Forward link, also reused for insertion into the separate chain |
| +4 | Successor backlink location written during removal |
| +8 | Condition field, reset after a dependency call when nonzero |
| +16 | Associated-object address |
| +28 | Flags; bit `0x80` is cleared |

The removed entry’s own +4 field is not directly updated by the removal sequence. The write targets +4 of the successor addressed by a reread of the removed entry’s forward link.

**Associated state**

- Entry +16 addresses an object containing a descriptor address at +4.
- Descriptor +4 receives the reset value.
- Descriptor +16 supplies either a source address for direct transfer or the second argument to F-2489d48d501c.

**Other implicit state**

- Chain-state +8 is a 32-bit chain head.
- Chain-state +1 is an independently accessed byte.
- Comparison-state +8 and +12 are 32-bit comparison operands.

These accessed offsets do not establish complete structure sizes, allocation extents, alignment, or disjointness.

## Behavior

### Initial dispatch

Before any accounting or pending-head inspection, F-2489d48d501c is called with:

1. The address-like expression formed from context +0 plus the width/multiplier times the numeric context address.
2. The 32-bit value at initial descriptor +16.

The first multiplication uses the context address, not the value at context +12. This distinction remains part of the observed contract even though its intended meaning is unresolved.

No returned value from this call is visibly consumed. Its effects on memory, implicit operands, ownership, and control flow are unknown.

The supplied representation does not establish a total evaluation order between the reads used to prepare the two arguments.

### Accounting and pending-head inspection

Before each pending-head inspection, including the final empty observation, the caller performs the following sequence:

1. Read, increment, and write context +44.
2. Read context +16 as the accounting count.
3. Read context +48 as the stored maximum.
4. If the stored maximum is less than the sampled count under an unsigned 32-bit comparison, write the sampled count to context +48.
5. If the sampled count equals one, read context +32. If that field is nonzero, call F-6ad030b1b8cc.
6. Read context +20 as the pending head.

The count governing step 5 is the earlier sampled value. It is not reread for that decision.

F-6ad030b1b8cc exposes an auxiliary machine result. The caller does not directly store that result into the context. It does not determine the terminal comparison: on the terminal branch that compares two words, both operands are loaded from comparison-state.

Accounting precedes the count increment performed during entry processing. With relevant state preserved by dependencies and no arithmetic overflow, a count reached by removing an entry is considered for the stored maximum during the next accounting phase.

### Empty-head termination

If the observed pending head is zero, no entry-processing or direct-transfer branch is taken. Instead:

1. Read chain-state +8.
2. If that word is nonzero, call F-unresolved-01 with that word.
3. If that word is zero:
   - Read comparison-state +8.
   - Read comparison-state +12.
   - If the words differ, call F-unresolved-01 with the word read from comparison-state +8.
   - If the words are equal, read the byte at chain-state +1. Call F-unresolved-01 with its zero-extended value only when that byte is nonzero.
4. Call F-unresolved-02.
5. Return 32-bit zero if the calls return normally.

The reads are short-circuited. A nonzero chain head avoids both comparison-state reads and the byte read. Unequal comparison words avoid the byte read.

The conditional dependency can receive zero: comparison-state +8 may be zero while comparison-state +12 is nonzero. Invocation on that branch depends on inequality, not on the argument’s truth value.

No semantic role such as release, unlock, notification, or scheduling is established for either terminal dependency.

### Removing an entry

For a nonzero pending head, the observed entry address is retained for entry processing.

First, the caller reads entry +0 and writes that value to context +20. If the value just read was nonzero, it then reads entry +0 again and writes the address of context +20 into the second link value’s +4 field.

The nonzero test uses the first link read; the successor address uses the second. The representation does not justify collapsing them into one read. Aliasing or external mutation could make them differ, and no second nonzero check is visible.

Next, the caller reads entry +28, clears bit `0x80`, and writes the masked word back. The exact mask is `0xFFFFFF7F`.

If that masked value is zero, the caller:

1. Reads chain-state +8.
2. Writes the value into entry +0.
3. Writes the entry address into chain-state +8.

If the masked value is nonzero, these chain-insertion operations are skipped. The branch uses the masked value derived from the flag read, not a fresh read of the stored flags.

The caller then reads entry +8. If it is nonzero, the caller invokes F-f46257aeeddc and, after that call returns, writes the reset value into entry +8. No explicit argument is shown for the call. The post-call reset does not establish that the dependency operates only on this field or on this entry.

### Associated-object, count, and index updates

After the preceding entry operations, the caller:

1. Reads the associated-object address from entry +16.
2. Reads the descriptor address from associated-object +4.
3. Writes the reset value into descriptor +4.
4. Reads, increments, and writes context +16.
5. Reads context +12, increments that value, and writes the incremented value to context +12.
6. If the incremented value equals the wrap threshold, writes the reset value to context +12.
7. Reads context +12 again for the subsequent destination calculation.

The equality comparison uses the incremented index value. It is not a greater-than-or-equal bounds check. An index beyond the threshold is not reset merely because it is beyond the threshold.

The final index read is distinct from the earlier index read and writes. Aliasing between accessed fields can therefore affect the value used for transfer.

### Transfer selection and repetition

After entry processing, the width/multiplier selects the following behavior:

| Observed width | Caller behavior |
|---|---|
| 0 | No per-entry transfer; return to accounting. |
| 1 | Read one byte from the source addressed by descriptor +16 and write one byte to destination base plus index. |
| 2 | Read two bytes from that source and write two bytes to destination base plus twice the index. |
| 4 | Read four bytes from that source and write four bytes to destination base plus four times the index. |
| Any other value | Call F-2489d48d501c with destination base plus width times index, and the value at descriptor +16. |

The descriptor in this phase is the one obtained through the removed entry. The destination base is obtained from context +0.

After a direct transfer or the dependency-based path, processing returns to accounting. There is no separate caller-side direct transfer for a nonstandard width.

A direct transfer preserves the selected-width bit pattern. No byte-order conversion, wider destination store, or repeated element copy is shown. The source value must be obtained before its corresponding destination write, but the representation does not establish a complete order among all source-pointer, destination-base, and address-expression reads.

## Side effects

Established caller-side effects are:

- Accounting-counter increments and conditional maximum updates.
- Pending-head replacement and conditional successor-backlink replacement.
- Entry flag-bit clearing.
- Conditional linkage into the chain rooted at chain-state +8.
- Conditional replacement of entry +8.
- Replacement of descriptor +4.
- Count and index updates.
- One-, two-, or four-byte destination writes on direct-transfer paths.
- Dependency calls under the conditions described above.

No caller-side allocation, explicit release, rollback, or synchronization primitive is visible. Linking an entry into another chain does not establish allocator ownership transfer or permission for the caller’s client to release it.

No null, range, alignment, list-integrity, or destination-capacity validation is shown beyond the specific field tests. A nonzero address is not thereby established to be accessible.

Partial mutation is possible if execution fails or a dependency does not return. Pending-head removal and flag changes precede access to the associated descriptor; a later access failure need not leave the entry pending.

The recovered ordinary operations use 32-bit values. Direct transfers use 8, 16, or 32 bits, and the terminal byte is zero-extended. The maximum comparison is explicitly unsigned. Exact overflow behavior for increments, multiplication, and address addition is **documented_only**: the supplied representation alone does not establish a portable arithmetic or address-wrapping guarantee.

Termination is not guaranteed. It requires an eventual empty-head observation and normal return from every dependency reached along the path.

## Dependencies

| Opaque ID | Established caller-visible invocation |
|---|---|
| F-2489d48d501c | Called initially without a width restriction. Called again after entry processing when the selected width is neither zero, one, two, nor four. Receives a computed address-like value and descriptor +16 value. No returned value is visibly consumed. |
| F-6ad030b1b8cc | Called when the sampled accounting count is one and context +32 is nonzero. No explicit arguments are shown. Exposes an auxiliary machine result. |
| F-f46257aeeddc | Called when removed entry +8 is nonzero. No explicit arguments are shown. Followed, if it returns, by the caller’s reset write to entry +8. |
| F-unresolved-01 | Conditional terminal dependency receiving a chain-head word, a comparison-state word, or a zero-extended byte. |
| F-unresolved-02 | Terminal dependency called without explicit arguments after the conditional terminal handling. |

The unresolved IDs distinguish the two terminal targets without asserting identities absent from the dependency map.

No dependency is assumed to preserve globals, implicit operands, pending-list state, associated objects, or ownership. The caller’s observed operations describe what happens after each call returns using the state then available, not a guarantee that pre-call state survives.

## Acceptance examples

These are conditional static-contract examples, not executed tests. All accessed locations must be valid for their stated widths, and invoked dependencies must return normally for a return-value claim.

1. **Observed empty state with no conditional terminal call.**  
   If the first pending-head inspection observes zero, the ensuing chain-head read yields zero, the comparison reads yield equal words, and the byte read yields zero, the caller skips F-unresolved-01, calls F-unresolved-02, and returns zero. The initial dispatch and one accounting phase precede this inspection.

2. **Observed nonzero terminal chain head.**  
   If an empty-head path reads a nonzero chain-state +8 value, F-unresolved-01 receives that value. The caller does not read the comparison-state fields or chain-state +1 on that terminal path.

3. **Unequal comparison words with a zero first word.**  
   If the terminal chain head is observed as zero and the subsequent comparison reads yield zero and seven, F-unresolved-01 is called with zero. The terminal byte is not read.

4. **Flag-controlled insertion.**  
   If the flag read yields `0x80`, the caller writes zero and performs the chain-insertion sequence. If it yields `0x81`, the caller writes one and skips that insertion sequence. Later dependency effects are not constrained by this example.

5. **Zero-width processing with explicit preservation conditions.**  
   Suppose the initial width is zero and the initial pending structure is a finite, valid, noncyclic list. Suppose every dependency preserves all implicit operands and does not alter the context, pending links, entries, associated objects, descriptors, or transfer-relevant state used by this scenario. Assume accessed regions do not alias in ways that disrupt the described list progression; the caller’s documented mutations remain permitted. Then each pending entry is removed and its associated descriptor, count, and index processing occurs without a per-entry direct transfer or per-entry F-2489d48d501c call. The initial F-2489d48d501c call still occurs. Merely forbidding dependencies from adding entries would not be sufficient: changing the width could change transfer behavior.

6. **Two-byte transfer after equality wrapping.**  
   If the incremented index equals the wrap threshold, the reset write occurs, and the subsequent index read yields that reset value, an observed width-two branch uses twice that value in its destination offset. No zero reset is implied. Nonaliasing, valid source and destination storage establishes the ordinary two-byte transfer case.

7. **Accounting includes the empty observation.**  
   Suppose exactly three entries are processed, dependencies preserve relevant state, caller mutations do not alias the accounting counter, and arithmetic does not overflow. Context +44 increases by four: three entry-bearing inspections and one terminating empty inspection.

8. **Nonstandard width with explicit preservation conditions.**  
   Suppose the initial width is three and exactly one valid pending entry is present. Suppose every dependency preserves all implicit operands and the context, list, entry, associated-object, and descriptor state needed for this scenario, while allowing the caller’s documented mutations. Assume relevant regions do not alias disruptively and address arithmetic is valid. The initial F-2489d48d501c call cannot then clear the pending head or change the width. After removing the entry and updating its bookkeeping, the caller makes a second F-2489d48d501c call using the updated index and entry-derived descriptor. It performs no direct three-byte transfer.

## Uncertainties

The following remain **documented_only**:

- Initialization, calling convention, and preservation rules for implicit operands.
- Complete structure layouts, allocation boundaries, and permitted aliasing.
- Dependency memory effects, ownership effects, failure behavior, and nonreturning behavior.
- Exact overflowing arithmetic and address-calculation semantics.
- Alignment requirements and invalid-access behavior.
- Concurrency, atomicity, interrupt interaction, and memory ordering.
- Whether the initial multiplication by the context address is intentional.
- Whether the count and index represent occupancy, positions, sequence values, or another abstraction.
- Semantic identities of the terminal dependencies.

A queue-drain operation with fixed-width storage and entry reuse is inferred intent only. It does not replace the established field operations, branch conditions, or dependency uncertainty.

## Evidence

This contract is based on the supplied static representation and opaque dependency mapping. Established claims concern visible accesses, call sites, data widths, predicates, and control-flow relationships. Semantic interpretations are identified separately; unavailable behavior remains documented rather than assumed.

No runtime execution, implementation inspection beyond the supplied representation, smoke run, or tests have been performed. Acceptance examples state conditional expected behavior and are not verification results.
