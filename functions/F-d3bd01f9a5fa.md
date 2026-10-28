<!-- Trusted provenance: F-d3bd01f9a5fa; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d3bd01f9a5fa

## Purpose

F-d3bd01f9a5fa performs count-controlled dispatch over records connected by relative links. It obtains an opaque context value once, captures a signed record count from the initial record, resolves a dispatch target, and then repeats this sequence:

1. Pass the captured context value to F-d0b41dd4354d.
2. Invoke F-f3fa9f888021 with the resolved target, the supplied base value, the current record’s payload address, a caller-supplied opaque value, and zero.
3. Read the current record’s relative link and advance to the resulting address.

The function has no explicit return value. Its observable behavior consists of dependency calls and memory reads. It contains no explicit stores to caller-provided memory, but its dependencies may modify memory or external state.

Calling the records a “list” and the opaque context value a “saved context” are inferred interpretations. The established behavior is relative-address traversal and repeated use of a captured value; neither a conventional list format nor context-restoration semantics is established.

## Interface

The interface has four inputs, described here by their roles:

| Input | Representation | Use |
|---|---|---|
| Base | 32-bit integer used as an address-bearing value | Passed unchanged to each dispatch. When the target input is zero, also supplies the address of a default-target field. |
| Initial record address | Pointer to 32-bit integer storage | Supplies the initial count, first payload address, and first relative link. |
| Target | 32-bit integer | Used directly when nonzero. Zero selects a value read from the base structure. |
| Opaque argument | Uninterpreted 32-bit value | Passed unchanged to every dispatch. |

These widths follow the supplied representation: integer values and addressed record words are 32 bits, and the opaque argument occupies four bytes. Exact platform addressing, byte order, alignment requirements, and behavior on address overflow are not independently established.

The return interface is void-like: normal completion produces no explicit result value. Return values from the two dependencies called inside the loop are not consumed.

### Addressed layout

The initial record and each subsequently selected record have these relevant offsets:

| Byte offset | Interpretation |
|---|---|
| +0 | Signed 32-bit count, read only from the initial record |
| +4 | 32-bit relative displacement, read when advancing from that record |
| +8 | Beginning of the payload region passed to F-f3fa9f888021 |

No payload length or payload field layout is available. Passing the address at offset +8 does not establish how much accessible storage the dispatch dependency requires.

For a current record beginning at address \(R\), its next address is derived from:

\[
R_{\text{next}} = (R + 4) + D
\]

where \(D\) is the 32-bit displacement read at \(R + 4\). The displacement is relative to its own field, not to the record beginning, payload beginning, or initial record.

The initial count does not become a per-record count. After traversal advances, the word at offset +0 of the new record is not read as a count by this function.

When the target input is zero, a 32-bit target is read from byte offset +56, equivalently +0x38, relative to the base.

## Behavior

### Initialization and read order

The function first calls F-7b103854e61a with no arguments and captures its four-byte result.

Only after that call returns does it read the signed 32-bit count from the initial record address. Consequently, changes made by F-7b103854e61a to that memory can affect the count observed by this invocation.

Next, it resolves the target:

- A nonzero target input is retained without a direct read of the base’s default-target field.
- A zero target input causes one 32-bit read at base +56. The resulting value becomes the target for all dispatches in this invocation.
- A default-target value of zero remains zero. There is no second fallback, rejection, or retry.

This ordering applies even if the captured count is zero or negative. A nonpositive count prevents loop-body calls, but it does not prevent the initial dependency call, count read, or conditional default-target read.

The count and resolved target are captured once. Later changes to their original memory locations do not cause them to be reread.

### Per-record processing

For each admitted iteration, the function performs these operations in order:

1. Calls F-d0b41dd4354d with the context value captured during initialization.
2. Calls F-f3fa9f888021 with these five arguments, in order:
   - The resolved 32-bit target.
   - The original base value.
   - The address eight bytes beyond the current record beginning.
   - The original opaque 32-bit argument.
   - A zero-valued final argument.
3. Reads the 32-bit relative displacement at current record +4.
4. Computes the next record address relative to the displacement field.
5. Updates the loop counter and evaluates whether another iteration is admitted.

The relative-link read occurs after both dependency calls return. A dependency that modifies the current record’s link can therefore change which record is selected next.

Conversely, the payload address is determined from the current record before dispatch. This function does not first follow the link and then dispatch the next record’s payload.

### Completion and the final link read

Traversal is controlled by the captured count and loop counter, not by a null-link convention.

For an ordinary finite positive count, the function reads and processes the link of the final dispatched record before discovering that no further dispatch is required. The final link field must therefore remain readable even when its computed destination will not be used.

The destination computed from that final link is not itself dereferenced by another iteration after normal count exhaustion.

A zero displacement is not an end marker. It selects the address of the current link field, four bytes beyond the current record beginning. Likewise, a repeated record address or a cycle is not detected or rejected.

### Counter width and large counts

Although the count is a signed 32-bit value, the supplied counter update has signed 16-bit narrowing behavior. Under the represented 32-bit arithmetic with arithmetic right shift, the counter follows this sequence:

- Starts at zero.
- Advances through the nonnegative values up to 32,767.
- Wraps from 32,767 to −32,768.
- Continues through the signed 16-bit range.

This creates three materially different cases:

| Captured count | Consequence under the represented counter arithmetic |
|---|---|
| Zero or negative | No iterations |
| 1 through 32,767 | Exactly that many iterations, provided all required calls and accesses complete |
| 32,768 through the maximum signed 32-bit value | The loop cannot terminate through its counter comparison |

For the last case, every reachable counter value is less than the captured count. This does not guarantee an indefinitely successful execution: a dependency may not return, or an access or address calculation may fail. It establishes that the loop’s own count comparison does not provide termination.

The narrowing behavior is significant and must not be silently replaced by an unrestricted 32-bit increment. The precise machine-level interpretation of the supplied arithmetic remains an evidence limitation described below.

## Side effects

### Direct effects

The function directly performs:

- One call to F-7b103854e61a.
- One initial count read.
- One conditional default-target read.
- One F-d0b41dd4354d call per admitted iteration.
- One F-f3fa9f888021 call per admitted iteration.
- One relative-link read per completed loop body.

It has no explicit direct writes to record storage, the base structure, or payload storage. Local traversal and counter updates are not caller-visible stores.

The function exposes the base value and payload address to F-f3fa9f888021. Any reads, writes, ownership transfers, or external effects performed by dependencies remain part of their unknown contracts, not established direct effects of this function.

### Aliasing and mutation

No independence between the base structure, record storage, and payload storage is established.

If dependency activity changes the current record’s link before the subsequent link read, traversal uses the changed link. If it changes the original count or default-target field after those values have been captured, this function continues using the captured values.

The repeated call to F-d0b41dd4354d does not prove that all global state is restored before dispatch. It receives the same captured value on every iteration, but its effects and the effects of the dispatch dependency are unknown.

There is no additional context-related call after traversal completes.

### Ownership and lifetime

No allocation, explicit deallocation, or explicit ownership transfer appears in this function.

That does not establish that dependencies preserve or release any supplied memory. In particular, after dispatch returns, the function still reads the current record’s link field. Safe execution requires that field to remain valid through that read, regardless of what dispatch does with the payload or enclosing allocation.

The initial record must be readable for the count read even when no iteration occurs. When fallback target resolution is selected, base +56 must also be readable even for a nonpositive count.

### Errors and abnormal completion

There is no explicit input validation, error result, retry, cycle check, or recovery path.

Null addresses, inaccessible fields, invalid alignment, arithmetic overflow, and dependency failures have no defined local recovery behavior in the supplied evidence. Dependency return values inside the loop do not control continuation.

If a dependency fails to return normally, the subsequent operations in that iteration are not reached through normal execution.

## Dependencies

| Opaque function ID | Established call contract | Unknown behavior |
|---|---|---|
| F-7b103854e61a | Called once, before the initial count read; takes no arguments; supplies a four-byte value retained for the invocation | Meaning of the value; memory and global-state effects; ownership; failure behavior |
| F-d0b41dd4354d | Called before every dispatch with the retained four-byte value | Whether it selects, restores, validates, or otherwise changes context; memory effects; failure behavior |
| F-f3fa9f888021 | Called with resolved target, original base, current payload address, original opaque argument, and zero | Payload interpretation; target interpretation; memory access extent; mutation; ownership; failure behavior |

No dependency is assumed to preserve globals, preserve input storage, or retain or release memory. Those properties require separate evidence.

## Acceptance examples

These examples are conditional expectations derived from the supplied representation, not executed tests. Unless stated otherwise, dependencies return normally and all required accesses and address calculations are valid.

### Zero count with an explicit target

Given an initial count of zero and a nonzero target:

- F-7b103854e61a is called before the count read.
- The base’s default-target field is not directly read.
- Neither loop dependency is called.
- The initial relative link is not read.
- The function returns without dispatch.

### Negative count with fallback target selection

Given a negative initial count and a zero target input:

- The initial context call and count read occur.
- The default target at base +56 is still read.
- No dispatch occurs.

An invalid fallback address is therefore not made harmless by the negative count.

### One record

Given a captured count of one and current record address \(R\):

- F-d0b41dd4354d receives the captured context value.
- F-f3fa9f888021 receives the payload address \(R + 8\), along with the other four prescribed arguments.
- The link at \(R + 4\) is read afterward.
- No second record is dispatched.

The computed next address need not name a readable record for this invocation, but the link field itself must be readable.

### Two records with a forward displacement

Given a captured count of two, first record address \(R\), and first displacement 28:

- The first payload address is \(R + 8\).
- The second record address is \(R + 32\), because the displacement is based at \(R + 4\).
- The second payload address is \(R + 40\).
- The second record’s link is read before normal completion.

### Link mutation during dispatch

Given a count admitting another iteration, suppose the first dispatch changes its record’s displacement from one valid value to another.

The next record is selected using the replacement value, because the link read follows dispatch. Capturing the old displacement before the dependency calls would change observable behavior.

### Captured fields remain fixed

Suppose initialization captures count two and resolves a fallback target. During the first iteration, a dependency changes both backing fields.

The second iteration still uses the originally captured count for admission and the originally resolved target for dispatch. Its record link, however, is read from current memory after the dependency calls.

### Counter boundary

With count 32,767, exactly 32,767 dispatches occur if all operations complete.

With count 32,768, the represented signed 16-bit counter wraps and never reaches a value that fails the loop comparison. An implementation that simply performs 32,768 dispatches and returns would not preserve that behavior.

## Uncertainties

The following items are **documented_only**:

- **Machine arithmetic details:** The supplied counter expression indicates signed 16-bit narrowing through 32-bit arithmetic and signed shifting. No independent machine evidence establishes signed-overflow and right-shift behavior. The counter-boundary conclusions are conditional on that represented arithmetic.
- **Address arithmetic:** Valid relative addressing is established structurally; overflow, wrapping, address provenance, and invalid-address behavior are not.
- **Platform memory rules:** Byte order, alignment requirements, and fault behavior are unspecified.
- **Record format beyond accessed fields:** Payload size, payload schema, record allocation boundaries, and any broader format invariants are unknown.
- **Dependency semantics:** Context meaning, target meaning, mutations, global-state preservation, ownership, and failure mechanisms are unknown.
- **Concurrency:** No synchronization or atomic access contract is established. The behavior of unsynchronized concurrent mutation cannot be supplied from this evidence.
- **Intended count range:** The function does not validate the count against 32,767. An external requirement restricting counts to the terminating range is possible but unproven.
- **Resource lifetime after return:** No cleanup obligation or retained-pointer behavior can be assigned without dependency contracts.

## Evidence

Established static evidence consists of the supplied representation of F-d3bd01f9a5fa and the supplied opaque dependency mapping. It supports the call order, field offsets, argument order, one-time count and target capture, post-dispatch link reads, lack of explicit stores, and counter-update analysis described above.

Inferred intent is limited to describing the operation as count-controlled relative-record traversal with a context-related action before each dispatch. Those descriptions do not establish undocumented callee semantics or ownership rules.

No runtime tests have been performed. The acceptance examples are analytical conditions, not observed execution results.
