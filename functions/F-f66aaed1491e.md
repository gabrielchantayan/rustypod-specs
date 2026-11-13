<!-- Trusted provenance: F-f66aaed1491e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f66aaed1491e

## Purpose

F-f66aaed1491e rearranges fixed-width records in caller-supplied storage. It uses callback comparisons, preparation involving three selected record positions, inward scans, record exchanges, and an explicit stack of deferred regions.

**Established behavior:** the initial region enters partition processing unconditionally. Regions subsequently selected for further processing are partitioned only when their recorded counts, interpreted as signed 32-bit integers, exceed ten.

**Inferred intent:** this is a partitioning stage of a comparison-based sorting operation. With a consistent, content-preserving comparator and ordinary region geometry, its preparation resembles median-of-three pivot selection. The supplied evidence does not establish a complete sort, a later finishing stage, or globally ordered output.

The function has no specified return value and no explicit error-reporting mechanism. It performs no direct allocation or release operation.

## Interface

### Inputs and result

The interface supplies four 32-bit quantities:

| Input | Established use |
|---|---|
| Region address | Starting address of the initial record region. |
| Record count | Region size used in address calculations and subsequent scheduling. Some operations interpret its bits as signed. |
| Record width | Byte spacing between records and byte extent transferred by each record exchange. |
| Comparator reference | Indirect callable reference, with its two least significant bits cleared before invocation. |

The function returns without an explicit result value. Observable outputs are memory changes, dependency calls, and eventual return or failure to return.

### Record layout and arithmetic

For ordinary positive counts and nonwrapping geometry, records occupy adjacent, equal-width byte ranges. The selected first, middle, and last positions have zero-based indices zero, the count divided by two rounded down, and the count minus one.

The actual middle calculation uses an arithmetic right shift of the count interpreted as signed 32-bit. Multiplication by the width and address calculations retain 32-bit results. Thus, the ordinary middle-index interpretation does not apply across the entire unsigned count range.

More precisely, the middle address is obtained by adding the width multiplied by the signed, shifted count to the region address. The last address adds that same displacement to the middle address, subtracting one width from the added displacement when the count is even. This selects the conventional last position only under suitable count and arithmetic conditions.

Every exchange treats a record as uninterpreted storage, including padding and payload bytes.

### Comparator contract

Each comparator invocation receives two 32-bit record addresses. Its 32-bit result is interpreted as signed:

- Positive results select greater-than branches.
- Negative results continue the forward scan.
- Positive results continue the backward scan.
- Zero stops either scan.

The callable target is the supplied comparator reference with both low bits cleared. References differing only in those bits select the same target.

There is no validation of the target, its ordering consistency, or its effects on memory.

**Additional conditions for value-level sorting interpretations:** comparison results must describe a consistent ordering, and callbacks must preserve all record contents relevant to the claimed outcome. Preserving only the two explicit arguments is insufficient: a callback could modify another selected record or the pivot record through another reference.

### Conditions for ordinary bounded processing

The implementation does not enforce these conditions:

- A positive record width.
- Sufficient readable and writable storage for every accessed record.
- Nonwrapping region arithmetic and byte-offset products.
- An initial count supporting the preparation and scans; three is the minimum for the ordinary three-position interpretation.
- A callable comparator target.
- Comparator behavior and record preservation sufficient for the intended scan stopping conditions.
- Results from F-d7be9cfb4000 that support valid subsequent region geometry and eventual progress.
- A deferred-region depth within the fixed internal capacity.

These conditions are not a complete safety proof for arbitrary dependency behavior. In particular, a dependency that changes storage validity can invalidate otherwise suitable input geometry.

## Behavior

### Region selection

The function maintains two parallel internal arrays, each containing 32 entries: one array of 32-bit addresses and one of 32-bit counts.

It stores the initial region in the first entry and immediately removes it for processing. Deferred regions are subsequently removed in last-in, first-out order.

There is no initial count check. An initially supplied count of ten or less does not bypass preparation or scanning. An empty initial region is not handled as an explicit no-op.

### Three-position preparation

For the current region, the function computes the first, middle, and last addresses described above.

The observable dependency and exchange sequence is:

1. Compare first against middle; exchange those records if the result is positive.
2. Compare middle against last.
3. If that result is positive, compare first against last, exchange first and last if this additional result is positive, and then exchange middle and last.
4. Exchange middle with the record one width before last.

The last exchange always occurs, including when its two addresses coincide. Other selected exchanges likewise are not omitted merely because their addresses coincide.

Under nonoverlapping, ordinary record geometry, **and provided all preparation callbacks preserve the relevant record contents and return a consistent ascending ordering**, the first three stages put the three selected values into nondecreasing order. Under those same conditions, the final exchange moves the selected median value to the position immediately before last.

Those value-level conclusions do not follow from consistent comparison results alone. A callback may return a correct comparison of its arguments while modifying another selected record. The unconditional contract is the call-and-exchange sequence, not preservation of the original selected values or their eventual ordering.

### Inward scans

The address one record before last becomes the fixed pivot-reference address for the scans. “Fixed” describes the address, not immutable contents. The function does not copy the pivot record into separate stable storage.

The forward cursor starts at the first address. Before each comparison, it advances by one width. It compares that record against the pivot-reference address and continues advancing while the result is negative.

The backward cursor starts at the pivot-reference address. Before each comparison, it retreats by one width. It compares that record against the same pivot-reference address and continues retreating while the result is positive.

Both scans perform their initial movement before comparing. The backward scan runs after the forward scan, even if the forward cursor has already reached the pivot-reference address.

After both stop:

- If the backward address is above the forward address, the two records are exchanged. Scanning then resumes, again moving each cursor before its next comparison.
- Otherwise, scanning ends.

The function then exchanges the forward record with the record at the pivot-reference address. This exchange is performed even for coincident addresses.

No explicit record-bound checks constrain either scan. Under conventional, content-preserving comparison semantics, the prepared records serve the inferred sentinel purpose. If callbacks mutate relevant contents, invalidate storage, or return unsuitable results, neither that purpose nor scan termination is established.

### Record exchange behavior

Each exchange independently chooses its transfer granularity:

- Four-byte units when the width and the first exchange address are both divisible by four.
- One-byte units otherwise.

The second address is not separately checked for alignment. Its alignment follows from common record spacing only when the addresses actually belong to appropriately formed records.

For each unit, in increasing address order, the observable accesses are:

1. Read the unit from the first address into temporary storage.
2. Read the corresponding unit from the second address.
3. Write the second unit to the first address.
4. Write the saved first unit to the second address.

An exchange is not atomic and does not buffer two complete records. With overlapping storage, later reads may observe earlier writes. With coincident addresses, reads and writes still occur; the function does not replace the exchange with a no-op.

The transfer loops execute at least once. A zero width therefore does not mean zero accesses. Decrementing the remaining-width quantity wraps, potentially causing extensive unintended access before a fault or eventual counter exhaustion.

### Post-partition bookkeeping

After the final scan-related exchange, the function calls F-d7be9cfb4000 with:

- The 32-bit difference between the final forward address and the current region’s first address.
- The record width.

The first argument is an address displacement, not an established element index. The dependency’s implementation is unavailable.

Its 32-bit result determines two recorded counts:

- The left count is the dependency result minus one.
- The right count is the current count minus the left count, minus one.

These calculations retain 32-bit results. Scheduling interprets both resulting counts as signed.

The left region retains the current first address. The right region starts at the final forward address itself—the location used for the final pivot exchange—not one width beyond it.

These details must not be replaced with conventional disjoint quicksort subdivisions. Even assuming that the dependency performs division would not justify changing either the subtraction or the right-region address.

### Scheduling and termination

The function compares the two derived counts as signed 32-bit integers.

When the left count is at most the right count:

- If the left count is at least eleven, defer the right region and immediately process the left.
- Otherwise, discard the left from further processing and select the right as the continuation. Partition that continuation only if its signed count exceeds ten.

When the left count exceeds the right count:

- If the right count is at least eleven, defer the left region and immediately process the right.
- Otherwise, discard the right from further processing. Continue with the left only if its signed count exceeds ten.

When no continuation qualifies, remove the most recently deferred region. Return when no deferred region remains.

No deferred-stack capacity check is present.

**Inferred intent:** processing the smaller substantial region first limits deferred depth. This is not an unconditional bound when dependency results or arithmetic produce unsuitable counts.

## Side effects

Direct effects are reads and writes of record storage, writes to fixed internal bookkeeping storage, and dependency invocations.

For valid, disjoint records, an individual completed exchange preserves the exchanged bytes as a pair. This does not establish that the function as a whole preserves the input records: callbacks or F-d7be9cfb4000 may modify them, and their memory effects are unavailable.

The function directly performs no heap allocation, release, ownership transfer, or explicit retention of the caller’s region after return. Dependency behavior must not be inferred from that absence. A dependency’s effects on globals, storage, allocation, ownership, or externally visible state remain unknown.

There is no rollback. A fault, nonreturning dependency, or invalid memory access may occur after earlier writes have completed.

Invalid counts, zero width, arithmetic wrap, invalid callable references, inconsistent comparisons, and deferred-stack exhaustion have no explicit error return or recovery path.

## Dependencies

### Dynamically selected comparator

No fixed opaque function ID is supplied for this caller-selected dependency.

Established interaction:

- Invoke the supplied reference after clearing its two low bits.
- Pass two record addresses.
- Interpret the returned 32-bit value as signed.
- Use it during preparation and both scans.

**documented_only:** calling-convention details beyond the visible arguments and result, mutation permissions, global-state effects, reentrancy, ownership effects, and failure behavior are unavailable.

In particular, the implementation does not enforce the content preservation required by the conditional median and sentinel interpretations.

### F-d7be9cfb4000

Established interaction:

- Called once after each completed partition.
- Receives a 32-bit byte-address displacement and the record width.
- Returns the quantity used to form both derived counts.

**Inferred intent:** convert a byte displacement into an element-related quantity.

**documented_only:** exact arithmetic, signedness, behavior for zero width, memory and global effects, ownership effects, and failure behavior are unavailable. Ordinary division and dependency purity are not assumed.

## Acceptance examples

These are conditional static expectations, not executed tests.

### Preparation with preserved contents

Assume five valid, nonoverlapping records, a positive width, and callbacks that preserve the entire region and consistently compare ascending keys.

If the initial first, middle, and last keys are 3, 2, and 1, preparation first exchanges first with middle. The next two comparisons are positive, causing first-to-last and middle-to-last exchanges. Before the final preparation exchange, the three selected positions contain 1, 2, and 3.

The final preparation exchange moves key 2 from the middle position to the position immediately before last. This value-level expectation requires the stated preservation condition.

### Consistent results with mutation do not establish ordering

Use the same three selected keys, 3, 2, and 1, in a five-record region. Let the first two comparisons leave storage unchanged.

After the first exchange, the selected positions contain 2, 3, and 1. During the first-versus-last callback, let the callback correctly return a positive comparison for 2 versus 1, but change the middle record’s key to 0.

The ensuing first-to-last and middle-to-last exchanges leave the selected positions containing 1, 2, and 0 before the final preparation exchange. They are not nondecreasing.

This behavior is compatible with the visible call sequence and correct ascending results for those compared arguments. It refutes an unconditional ordering postcondition when mutation is permitted.

### Equal-key scans

Assume valid geometry, preserved record contents, and a comparator returning zero for every comparison.

Each directional scan stops at its first comparison after moving once. Exchanges still occur while the backward address is above the forward address. Equal-key records therefore need not retain their relative order, and their payloads need not remain at the same addresses.

### Conditional count derivation

If a completed partition has current count twenty and F-d7be9cfb4000 returns nine, the recorded left and right counts become eight and eleven.

The left receives no further partition processing. The right continuation begins at the final forward address and has count eleven. This example does not assert which displacement causes the dependency to return nine.

### Deferred order

If the current count is forty and the dependency returns thirteen, the derived counts are twelve and twenty-seven.

The right region is deferred; the left is processed immediately. Any region deferred later is removed before that earlier right region.

### Initial cutoff and transfer width

An initially supplied count of ten still enters preparation and scanning, assuming storage and dependencies permit progress. Ten is not an entry shortcut.

For an eight-byte width and a first exchange address divisible by four, an exchange uses two ordered four-byte units. For a three-byte width, it uses three ordered one-byte units.

Neither example establishes complete sorting of the output.

## Uncertainties

- **documented_only — helper semantics:** F-d7be9cfb4000 is unavailable, preventing a complete mathematical interpretation of derived regions.
- **documented_only — dependency mutation:** callbacks and the helper may affect records, globals, storage validity, or ownership. Their preservation properties are not established.
- **documented_only — broader operation:** no caller or finishing stage establishes a complete sorting pipeline.
- **documented_only — invalid geometry:** small counts, zero width, wrapped arithmetic, and inaccessible storage have no established safe outcome.
- **documented_only — large inputs:** signed shifting and signed count decisions prevent treating every unsigned count as ordinary array geometry.
- **documented_only — capacity and termination:** no unconditional deferred-depth bound or progress proof follows for arbitrary dependency results.
- **Inferred intent:** median selection, sentinel-bounded partitioning, and smaller-region-first scheduling require additional conventional assumptions.
- Stability is not guaranteed, including when comparison-equivalent records have different payloads.
- Concurrent access and device-backed memory have no established synchronization guarantees. The described transfer order does not make exchanges atomic.

## Evidence

This contract rests solely on static inspection of the supplied function body and the supplied opaque dependency mapping.

Established evidence includes four 32-bit inputs, masked indirect comparator calls, signed result tests, 32-bit count and address arithmetic, ordered unit exchanges, two 32-entry deferred arrays, unconditional initial processing, the derived-region cutoff, and the call to F-d7be9cfb4000.

Value-level preparation claims are explicitly conditional on consistent ordering and preservation of relevant record contents by callbacks. Fixed pivot-address use is distinguished from preservation of the pivot value. Unknown dependency behavior remains documented rather than assumed.

No runtime tests, executions, or dynamic observations have been performed.
