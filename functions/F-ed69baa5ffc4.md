<!-- Trusted provenance: F-ed69baa5ffc4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ed69baa5ffc4

## Purpose

F-ed69baa5ffc4 rebuilds a selected collection of 48-byte records from linked nodes. Before processing the linked structures, it overwrites two working-coordinate fields in every node reached by a separate flat-array traversal.

Records describe runs selected by direction magnitude and bounded by changes in signed direction. Each record retains its starting direction, starting node, closing node, and originating list-entry address. Its first-coordinate summary incorporates every visited node from the start through the closing node. Its second-coordinate bounds use only the two endpoints.

“Run,” “coordinate,” and “collection” describe the observable organization. A geometric interpretation is inferred intent, not an established domain contract.

This specification describes static evidence only. No runtime tests have been performed.

## Interface

### Arguments and result

The function receives:

- A pointer to a mutable context.
- A signed 32-bit selector used both to select context fields and to choose coordinate orientation.

It returns a 32-bit status:

| Result | Established condition |
|---|---|
| 0 | All list entries have been processed, possibly without producing records. |
| 64 | A matching node requires a slot, the selected count is not less than capacity, and capacity is not less than the first global-limit value read for that acquisition. |
| Dependency-reported nonzero status | F-6dd003795db7 reports an error during growth. |

An available slot bypasses the global-limit checks. Capacity equal to or greater than the global limit therefore does not, by itself, cause failure.

No explicit validation establishes safe handling of null pointers, invalid selectors, malformed links, overlapping storage, or arithmetic overflow.

### Context layout

Offsets are in bytes. Pointers occupy 32 bits in the supplied representation.

| Offset | Width | Meaning |
|---|---:|---|
| 0 | 32 bits | Value captured for growth-dependency calls |
| 28 | 32 bits | Flat-array count used in end-address arithmetic |
| 32 | 32-bit pointer | Flat-array base |
| 40 | 32 bits | List-entry count used in end-address arithmetic |
| 44 | 32-bit pointer | List-entry base |

For selector \(s\), the selected collection uses:

| Offset | Width | Meaning |
|---|---:|---|
| \(48+28s\) | signed 32 bits | Record count |
| \(52+28s\) | signed 32 bits | Record capacity |
| \(56+28s\) | 32-bit pointer | Record storage |
| \(72+28s\) | signed 8 bits | Selected direction |

Selector zero uses one coordinate orientation; every nonzero selector uses the other. This distinction does not establish that every nonzero selector is a valid collection index.

### Node layout

Flat-array nodes have a 40-byte stride. Linked-node accesses use:

| Offset | Width | Meaning |
|---|---:|---|
| 0 | unsigned 16 bits | Flags |
| 3 | signed 8 bits | Direction |
| 12 | signed 16 bits | First source coordinate |
| 14 | signed 16 bits | Second source coordinate |
| 24 | signed 32 bits | First working coordinate |
| 28 | signed 32 bits | Second working coordinate |
| 32 | 32-bit pointer | Main traversal link |
| 36 | 32-bit pointer | Anchor-search link |

Only the low two flag bits participate in endpoint flag propagation. The function does not validate reciprocal links or require linked nodes to belong to the prepared flat array.

### Record layout

Records have a 48-byte stride.

| Offset | Width | Explicit assignment |
|---|---:|---|
| 0 | 8 bits | Zero at creation; bit 0 may be set at closure |
| 1 | 8 bits | Captured starting direction |
| 2 | 16 bits | Narrowed midpoint of first-coordinate extrema |
| 4 | 16 bits | Narrowed lower endpoint second coordinate |
| 6 | 16 bits | Narrowed upper endpoint second coordinate |
| 16 | 4 bytes | All zero |
| 28 | 4 bytes | `00 7D 00 00`, in increasing address order |
| 32 | 4 bytes | All zero |
| 36 | 32-bit pointer | Starting node |
| 40 | 32-bit pointer | Closing node, initially the starting node |
| 44 | 32-bit pointer | Address of the originating list entry |

The bytes at offset 28 represent 32000 under little-endian interpretation. Unlisted bytes depend on F-unmapped-initializer; zero initialization is not independently established.

## Behavior

### Initial capture and coordinate preparation

Before resetting the selected count, the function captures the context value at offset 0, the list base, the list end derived from the initial list count, and the selected direction.

A negative selected direction is negated and assigned back through an 8-bit character representation before widening. Node directions are instead widened before negation. Consequently, mathematical absolute-value behavior is not established for the extreme selected byte value.

The selected count is then written to zero. Capacity and record storage are not explicitly reset or released.

The flat-array base and count are read after that reset. Preparation precedes every list-entry check, including the empty-list case.

For selector zero, each flat-array node receives:

- Its sign-extended source coordinate at offset 12 into working offset 24.
- Its sign-extended source coordinate at offset 14 into working offset 28.

For a nonzero selector, the source coordinates are exchanged.

Within each node, the first source read and destination write precede the second source read and destination write. Nodes advance in 40-byte address steps. The loop compares unsigned addresses against a computed end address; it is not independently count-controlled. Ordinary complete-array traversal requires suitable nonwrapping arithmetic.

### List entries and anchor selection

List entries advance in four-byte address steps within the initially captured bounds. Each entry’s node pointer is read when the entry is processed.

If the listed node’s offset-36 link points to itself, that entry is skipped without first inspecting its direction or main link.

Otherwise, the listed node is the initial anchor candidate. Anchor selection first examines the direction magnitude of its offset-36 neighbor. Only if that matches the selected comparison value does it examine the listed node’s magnitude.

If both match, the search follows offset-36 links:

- Encountering a nonmatching node selects that node’s offset-32 successor as the anchor.
- Returning to the originally listed node without a nonmatch retains the original node as anchor.

If either initial magnitude comparison fails, the listed node remains the anchor.

This search compares magnitudes, not signed directions. Avoiding a split inside a matching region is inferred intent.

### Main traversal and boundaries

Main traversal follows offset-32 links. It processes the anchor on entry and recognizes a later encounter with that same anchor as the end of the list entry.

When no record is active, a node starts a record only if its widened direction magnitude matches the selected comparison value. The record retains the original signed direction byte.

While a record is active, the current node’s first working coordinate updates both extrema before any closure decision. Closure occurs when:

- The current signed direction differs from the record’s captured starting direction; or
- The current node is the anchor.

Thus a direction-changing closing node contributes to the first-coordinate extrema even though its direction does not belong to the preceding run.

After closure at a non-anchor node, that same node is immediately eligible to start another record. Opposite matching directions can therefore share a boundary node.

On the second encounter with the anchor, an active record closes first. Processing then advances to the next list entry without starting another record there.

### Slot acquisition and growth

The candidate node’s direction byte is captured before any growth or initialization call.

The function reads the selected capacity and compares the selected count against it using signed 32-bit comparisons.

If count is less than capacity, the function acquires an existing slot without reading the global capacity limit. For acquisition, it rereads count, increments count, computes the slot address from the selected storage pointer and the old count with a 48-byte stride, and calls F-unmapped-initializer with that address and 48.

If count is not less than capacity, global-limit evaluation follows this conditional order:

1. **Admission read:** Read the signed 32-bit global limit and compare the old capacity against that value. If capacity is not less, return 64 without calling F-6dd003795db7.
2. **Proposal calculation:** If growth is admitted, calculate
   \[
   p=c+(c\mathbin{\text{ arithmetic-right-shift }}2)+4,
   \]
   where \(c\) is the previously read capacity.
3. **Below-old-capacity branch:** If \(p<c\), skip the proposal-versus-limit comparison. Read the global limit again and use this newly read value as the proposed capacity.
4. **Other proposals:** If \(p\geq c\), read the global limit again for the signed comparison between that value and \(p\).
   - If this comparison value is less than \(p\), read the global limit once more and use that latest value as the proposed capacity.
   - Otherwise retain \(p\), with no replacement-value read.

These are distinct source-level evaluations. The value that admits growth, the value used to decide whether to clamp, and the value assigned by clamping are not established to be identical.

Accordingly, an insufficient-capacity acquisition uses:

- One global read if growth is rejected.
- Two if a below-old-capacity proposal takes the short-circuit replacement branch.
- Two if an ordinary proposal is retained.
- Three if an ordinary proposal requires replacement.

There is no revalidation that a replacement value satisfies earlier comparisons.

F-6dd003795db7 receives the captured context value, element size 48, old capacity, final proposed capacity, current selected storage pointer, and writable error-result storage initialized to zero.

Its returned pointer is assigned to the selected storage field before the error result is inspected.

- A nonzero error is returned without restoring the previous pointer.
- On zero error, the final proposed capacity is written to the selected capacity field and slot acquisition proceeds.

After successful growth, count is reread and incremented without another count-versus-capacity check. Dependency interference or changing limit values can therefore invalidate ordinary capacity assumptions.

### Record creation

After F-unmapped-initializer returns, explicit operations occur in this order:

1. Store the previously captured direction at offset 1.
2. Store zero at offset 0.
3. Read the starting node’s first working coordinate to seed both extrema.
4. Store the starting-node pointer at offset 36.
5. Write zero bytes at offsets 32–35.
6. Store the same node at offset 40.
7. Store the current list-entry address at offset 44.
8. Write `00 7D 00 00` at offsets 28–31.
9. Write zero bytes at offsets 16–19.

The active signed direction comes from the pre-call capture. The starting coordinate is read after initialization. A dependency-induced node mutation can therefore affect these two values differently.

### Record closure

Closure performs these operations in order:

1. Store the current node at record offset 40.
2. Add the accumulated first-coordinate extrema, shift the sum right by one, narrow to 16 bits, and store at offset 2.
3. Read the starting and closing nodes’ 16-bit flags. If their combined low two bits are nonzero, set record flag bit 0 by read-modify-write.
4. Read the closing node’s second working coordinate.
5. Read the second working coordinate through the record’s starting-node pointer.
6. Store the smaller endpoint value at offset 4 and the larger at offset 6, narrowing each to 16 bits.

The relative order of the two endpoint flag reads is not established by the supplied expression.

Interior nodes affect first-coordinate extrema, but not endpoint second-coordinate bounds or flag propagation. A run covering the entire circle closes at its starting anchor and consequently uses that node twice for endpoint bounds.

With unchanged working coordinates prepared from signed 16-bit sources, the midpoint sum fits signed 32 bits. Arithmetic right shift yields the lower integer midpoint for a negative odd sum.

## Side effects

Direct effects include:

- Resetting the selected count.
- Overwriting flat-array working coordinates.
- Incrementing count before slot initialization.
- Writing records incrementally.
- Replacing the storage pointer before checking growth status.
- Updating capacity after reported growth success.
- Retaining pointers to caller-supplied nodes and list-entry addresses.
- Conditionally reading the global limit multiple times within one acquisition.

There is no explicit rollback, record destruction, node release, list release, or independent copy of referenced nodes.

Absence of rollback is not a preservation guarantee. F-6dd003795db7 may modify or release storage or interfere with context, nodes, or globals. An error return does not establish continued accessibility of earlier records. The same noninterference limitation applies to F-unmapped-initializer.

Some values are captured while others are reread later. No snapshot, synchronization, reentrancy, or isolation guarantee is established. In particular, repeated global-limit evaluations within one acquisition do not constitute a stable limit snapshot.

## Dependencies

### F-6dd003795db7

Established arguments are the captured 32-bit context value, element size 48, old and final proposed signed 32-bit capacities, current storage pointer, and initialized writable error-result storage.

The pointer-sized return is stored before status inspection.

Resizing is inferred intent. Allocation policy, copying, ownership transfer, preservation on failure, permitted errors, and interference with other memory remain unknown.

### F-unmapped-initializer

This document-local opaque ID identifies the dependency for which no published mapping was supplied.

It receives the acquired slot address and 48. No return value is consumed.

Initialization or clearing is inferred intent. Its write coverage, ownership effects, and preservation of surrounding state are unknown.

### Global capacity-limit state

The limit is read as a signed 32-bit value only after insufficient capacity is found. Available-slot acquisition does not consult it.

Within one growth attempt, separate evaluations serve admission, conditional proposal comparison, and conditional replacement. The below-old-capacity condition short-circuits the proposal comparison but still causes a replacement-value read. Otherwise, a replacement read occurs only when the proposal exceeds the separately read comparison value.

Neither immutability nor equality among these read values is established. The replacement value is not rechecked against the old capacity, admission value, comparison value, or required count. No fixed maximum record count follows from this dependency.

## Acceptance examples

These are conditional examples, not execution results. Unless stated otherwise, assume valid stable memory, consistent circular links, nonoverflowing arithmetic, prepared linked-node coordinates, and successful noninterfering dependencies that preserve required storage.

1. **Empty list:** Count becomes zero and flat-array coordinates are prepared before status 0 is returned.

2. **Orientation:** Source coordinates −7 and 12 become working coordinates −7 and 12 for selector zero, or 12 and −7 for a valid nonzero selector.

3. **Singleton:** A node whose offset-36 link points to itself produces no record, regardless of direction.

4. **No match:** A valid nonsingleton circle without a matching magnitude produces no records.

5. **Closing-node contribution:** With selected magnitude 2, directions +2, +2, +1 close the matching run at the +1 node. That node contributes to first-coordinate extrema and endpoint bounds.

6. **Shared boundary:** A +2 to −2 transition closes the positive record at the negative node and can start a negative record at that same node.

7. **Interior extremum:** First coordinates 10, 30, and 14 from start through closure yield midpoint 20, not the endpoint-only midpoint 12.

8. **Endpoint-only summaries:** Endpoint second coordinates 8 and 12 yield bounds 8 and 12 despite an interior value of 100. An interior flag alone does not set record bit 0; endpoint flag value 2 does.

9. **Existing capacity bypasses the limit:** Count 0 and capacity 5 permit slot acquisition even if the global limit is 3. No global-limit read occurs on that path.

10. **Admission rejection:** Count 5, capacity 5, and an admission-read value of 3 produce status 64 without a growth call for that acquisition.

11. **Unclamped proposal:** With count and capacity 8, an admission value of 20 permits proposal 14. A comparison value of 14 or greater retains 14. There is no replacement-value read.

12. **Distinct comparison and replacement values:** With count and capacity 8, admission value 20 permits proposal 14. Comparison value 12 requires replacement. If the replacement read yields 9, the growth dependency receives proposed capacity 9, not 12 or 14. On reported success, capacity becomes 9 before slot acquisition.

13. **Short-circuit replacement:** If the calculated proposal is below the old capacity under the applicable arithmetic behavior, no proposal-versus-limit comparison occurs. A fresh global read supplies the replacement capacity directly. This is conditional on arithmetic behavior, not a claim that ordinary nonnegative capacities reach this branch without overflow.

14. **Growth error:** A nonzero dependency error is returned after its returned pointer replaces selected storage. No rollback or preservation of earlier records is guaranteed.

## Uncertainties

The following remain **documented_only**:

- **Extreme direction:** Under low-eight-bit narrowing, selected −128 remains −128 after normalization, whereas node −128 widens to magnitude 128. They do not match under that interpretation.
- **Arithmetic boundaries:** Invalid selectors, negative capacities, signed overflow, wrapped addresses, and overflowing slot offsets have no established safe outcome. Growth clamping does not prove overflow safety.
- **Shifts and narrowing:** Negative midpoint rounding depends on arithmetic signed right shift. Out-of-range 16-bit conversion is not independently specified.
- **Global changes:** The source establishes conditional repeated evaluations, not stable values, a synchronization mechanism, or which external changes can occur between evaluations.
- **Dependencies and ownership:** Allocation preservation, initialization coverage, ownership transfer, and interference remain unknown.
- **Linked-node membership:** Linked nodes are not checked against the prepared array; their existing working-coordinate fields may be read without preparation.
- **Termination:** Malformed links or cycles excluding the anchor can prevent termination or cause invalid accesses.
- **Aliasing:** Overlap among context, nodes, list entries, and records can affect later reads and invalidate ordinary structural interpretations.

## Evidence

The supplied static representation establishes field widths and offsets, coordinate preparation, anchor selection, signed-direction boundaries, endpoint treatment, acquisition branches, dependency arguments, explicit record writes, and pointer-before-error ordering.

The nested growth conditions establish separate global-limit evaluations: admission first; then either short-circuit replacement or proposal comparison; then replacement only if that comparison requires it. They do not establish equality between comparison and replacement values.

Geometric purpose, resizing intent, and clearing intent are interpretations rather than verified dependency contracts.

No runtime execution, tests, dependency implementation inspection, or external validation has been performed.
