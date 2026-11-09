<!-- Trusted provenance: F-a51e4d3e8ef4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a51e4d3e8ef4

## Purpose

F-a51e4d3e8ef4 calculates a boundary between two memory regions, conditionally performs a sequence of operations across that boundary, and places a requested item on one side or the other.

“Source,” “destination,” “reservation,” “insertion,” and “transfer” describe inferred intent. The established behavior consists of memory accesses, arithmetic, dependency calls, output writes, and status returns. The supplied evidence does not establish the complete record format or the dependencies’ internal behavior.

The function has two main phases:

1. Plan a count of destination-side operations using measurements and a reservation for the requested item.
2. If the plan is accepted, execute that count, inserting the item at its requested position or inserting it into the source afterward.

The return status alone does not distinguish successful execution from planning rejection. Both return zero. A zero result from either of two execution dependencies instead produces status 0x40. Rejection and explicit failure clear all four outputs, but neither establishes that dependency side effects were absent or reversed.

## Interface

The function receives eleven arguments in this order:

| Argument | Representation | Established role |
|---|---|---|
| Context | 32-bit address | Supplies flags and a capacity-like field; passed to most dependencies. |
| Destination | 32-bit address | Region used for destination-side operations. |
| Source | 32-bit address | Region measured and used for source-side operations. |
| Requested position | Unsigned 32-bit integer | Selects where the requested insertion falls among operations. |
| Item descriptor | Opaque 32-bit value | Passed to sizing and insertion dependencies. |
| Auxiliary descriptor | Opaque 32-bit value | Passed to the insertion dependency. |
| Extra size | Signed 32-bit integer | Participates in reservation arithmetic and is passed unchanged to insertion. |
| Position output | Address of a writable 16-bit object | Receives an insertion position. |
| Reference output | Address of a writable 32-bit object | Receives a dependency-produced value. |
| Flag output | Address of a writable 8-bit object | Explicitly receives zero or one. |
| Planned-count output | Address of a writable 16-bit object | Receives the accepted planning count. |

The explicit return values are 32-bit zero and 0x40.

The function accesses these fields:

| Region and offset | Access |
|---|---|
| Context + 0x1c | Unsigned 16-bit capacity-like field. |
| Context + 0x30 | 32-bit flags; bit 1 controls reservation overhead. |
| Source + 8 | One byte, tested for the all-ones byte value. |
| Destination + 10 | 16-bit field supplied to execution dependencies. |
| Source + current capacity − 2 | Signed 16-bit displacement used during ordinary transfers. |

For one reference-producing call, the argument is the address **source + 4**. The function does not load a pointer from that address. On the other reference-producing path, the argument is the destination address itself.

The caller must supply memory valid for every access reached, including dependency accesses under their separate contracts. No explicit null, bounds, alignment, overlap, or position-range checks appear.

The source-relative displacement can identify memory before or after the source address. This function does not establish the extent of either region. Native byte order and external aliasing rules are not supplied.

## Behavior

### Initial sizing and reservation

Before planning, the retained planning stride is initialized to zero. It remains zero if no planning iteration executes.

The function reads the byte at source + 8, tests whether it is 0xff, and passes that Boolean result with the context and item descriptor to F-f45584ca1b1e. Its 32-bit result is retained and later supplied to insertion calls.

After that call returns, the context flags are read. Bit 1 selects an overhead of two when set and one when clear. A flags mutation by F-f45584ca1b1e can therefore affect the reservation.

For ordinary non-overflowing arithmetic, the reservation is the retained sizing result, plus the selected overhead, plus extra size, plus two, rounded upward to an even value. The parity adjustment adds one only when the preliminary reservation is odd.

The capacity field is read and zero-extended after the preliminary reservation is formed and before the conditional parity adjustment. This is the planning capacity snapshot. It is not refreshed during planning.

Negative extra sizes, negative sizing results, and overflowing totals are not explicitly rejected. The ordinary “round upward” interpretation is not a complete contract for every possible 32-bit operand pattern; the supplied source representation does not fully establish signed-overflow semantics.

### Initial planning coordinates

F-8cb9e658d14d is called with context and source, then with context and destination.

Two signed 32-bit coordinates are formed:

- The source-side boundary is planning capacity minus source measurement, plus reservation.
- The destination-side coordinate is planning capacity minus destination measurement.

The source-side boundary is formed before the destination measurement call. Neither measurement call causes the saved capacity or reservation to be recomputed.

The planning count starts at zero. The retained stride also starts at zero, independently of whether the loop is entered.

### Planning iterations

Planning continues while the destination-side coordinate is strictly less than the source-side boundary, using a signed comparison.

Each iteration selects a stride:

- If the planning count equals the full unsigned 32-bit requested position, the reservation is selected.
- Otherwise, F-c4c1afdb3fb3 is called with context and source, and the selected stride is its result plus two.

The planning invocation visibly supplies two arguments. A later execution invocation supplies three. The complete calling contract is unresolved.

The selected stride replaces the retained stride. The planning count is then incremented and masked with 0xfffeffff, the stride is subtracted from the source-side boundary, and the stride is added to the destination-side coordinate before the next comparison.

Starting from zero, this masked increment cycles through 0–65535: the increment after 65535 sets bit 16, which the mask clears. The requested position is not truncated for equality testing. Requested positions above 65535 cannot match the planning count.

No local progress or iteration-limit check exists. Dependency results and arithmetic can produce zero or adverse strides, repeated count cycles, or nontermination.

### Overshoot correction and acceptance

After the loop, the destination-side coordinate is compared with the saved capacity. If it is strictly greater:

1. The planning count is decremented and masked with 0xffff.
2. The retained stride is added back to the source-side boundary.
3. The destination-side coordinate is left unchanged.

The retained stride is the last selected stride **only if an iteration occurred**. When the loop executes zero times, it is still zero. Consequently, a zero-iteration overshoot correction changes the count from zero to 65535 but leaves the source-side boundary unchanged.

The adjusted source-side boundary is then compared with the saved capacity, using a signed comparison:

- A boundary strictly greater than capacity rejects the plan.
- A boundary equal to or less than capacity accepts it.

The zero-iteration correction is not excluded by the available dependency contracts. In particular, negative destination measurements are not locally rejected.

### Planning rejection

On rejection, the function writes these outputs in order:

1. Position: 16-bit zero.
2. Reference: 32-bit zero.
3. Flag: 8-bit zero.
4. Planned count: 16-bit zero.

It then returns zero.

No execution-phase insertion or transfer is called on this path. Sizing and planning dependencies have already executed, so rejection is not necessarily side-effect-free.

### Accepted-plan initialization

For an accepted plan, the function first writes the adjusted count to the 16-bit planned-count output, then writes one to the 8-bit flag output. It does not initialize the position or reference output at this point.

An execution counter begins at zero. Execution continues while that counter is less than the adjusted planning count. The counter uses the same masked increment as planning.

The adjusted count is in the range 0–65535. Thus, assuming dependencies return normally, the execution counter can reach it without wrapping first. In particular, an accepted count of 65535 runs counters 0 through 65534.

### Destination-side insertion

When the execution counter equals the full requested position:

1. The function reads and retains the 16-bit field at destination + 10.
2. F-dfe52f60212b receives context, destination, the retained field, item descriptor, original sizing result, auxiliary descriptor, and extra size.
3. A zero result takes the explicit failure path.
4. Otherwise, the retained destination field is written to the position output.
5. F-4055be9c3f18 is called with the address source + 4.
6. Its 32-bit result is written to the reference output.

The position output uses the destination field read before insertion. It is not refreshed afterward, even if insertion changes that field.

The reference dependency receives an address within the source region, not a value fetched from source + 4.

### Ordinary transfer operation

When the execution counter differs from the requested position:

1. F-c4c1afdb3fb3 is called with context, source, and zero.
2. The context capacity is read again for this operation.
3. A signed 16-bit displacement is read at source + current capacity − 2.
4. The sign-extended displacement is added to the source address.
5. F-7bb443a5224b receives context, destination, the current 16-bit field at destination + 10, the constructed source-relative pointer, and the result of the preceding F-c4c1afdb3fb3 call.
6. A zero result takes the explicit failure path.
7. Otherwise, F-76b5978a0554 is called with context, source, and zero.

The return value of F-76b5978a0554 is not examined.

The destination-field read and displacement-related reads occur after F-c4c1afdb3fb3 and before F-7bb443a5224b. Their relative order within argument evaluation is not established.

The execution capacity need not equal the saved planning capacity. The function permits earlier dependencies to change what later reads observe; no preservation guarantee is supplied.

### Source-side insertion after execution

After the loop, if the adjusted planning count is less than or equal to the full requested position, insertion targets the source.

The insertion position is the low 16 bits of requested position minus completed execution count. F-dfe52f60212b receives context, source, that position, item descriptor, original sizing result, auxiliary descriptor, and extra size.

A zero result takes the explicit failure path. Otherwise:

1. The same low-16-bit relative position is written to the position output.
2. F-4055be9c3f18 is called with the destination address itself.
3. Its result is written to the reference output.

For a normally terminating accepted execution, the requested insertion therefore occurs either within the destination-side sequence or after that sequence on the source side. Large requested positions are compared at full width but truncated when used as source-side insertion positions.

### Explicit failure and normal completion

A zero result from F-dfe52f60212b or F-7bb443a5224b clears position, reference, flag, and planned count, in that order, then returns 0x40.

No rollback is visible. Earlier operations and partial mutations by the failing dependency can remain observable.

All normally completed accepted paths return zero. No additional final flag write occurs.

## Side effects

Direct writes by this function target the four output objects. Dependencies receive region addresses and opaque arguments and may mutate reachable memory or global state. Their preservation, allocation, release, and ownership behavior must not be inferred from this caller.

The flag becomes one before execution finishes. It is not a transactional completion marker. A later explicit failure clears it after earlier operations may already have changed state.

Output stores are ordered, not atomic as a group. If outputs overlap one another or input memory, store order can affect final bytes and later field reads. The available evidence does not establish whether such aliasing is permitted.

The planned-count output is written before execution, but execution uses the retained internal count rather than rereading that output.

No local allocation, release, ownership transfer, or reference-count operation is established. The reference dependency’s result is merely stored as 32 bits; its lifetime, encoding, and ownership are unknown.

## Dependencies

| Opaque ID | Established use | Unresolved behavior |
|---|---|---|
| F-f45584ca1b1e | Supplies the retained sizing result from context, item descriptor, and source-byte predicate. | Units, signed range, errors, mutations. |
| F-8cb9e658d14d | Supplies source measurement first, destination measurement second. | Measurement meaning, permitted range, side effects. |
| F-c4c1afdb3fb3 | Supplies ordinary planning stride before adding two; supplies an execution argument without that addition. | Differing visible argument counts, result stability, mutations. |
| F-dfe52f60212b | Performs the insertion-like operation; zero triggers status 0x40. | Partial effects, error causes, ownership, nonzero-result meaning. |
| F-7bb443a5224b | Receives destination information, a source-relative pointer, and a sizing value; zero triggers status 0x40. | Copying versus moving, consumption, partial failure. |
| F-76b5978a0554 | Called after a nonzero transfer result; return value ignored. | Removal or release semantics, failures, mutations. |
| F-4055be9c3f18 | Produces the reference output from source + 4 or destination. | Encoding, nullability, ownership, lifetime, side effects. |

No dependency is assumed to preserve context fields, region contents, or globals without a separate contract.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Assume valid disjoint memory, non-overflowing arithmetic, and normally returning dependencies unless stated otherwise.

1. **Zero operations without correction.**  
   Capacity 100, reservation 20, source measurement 40, and destination measurement 20 produce source boundary 80 and destination coordinate 80. Planning skips the loop and accepts count zero. The count output becomes zero and the flag one. Insertion targets the source at the requested position’s low 16 bits. On success, reference acquisition receives the destination address.

2. **Zero iterations with correction and acceptance.**  
   Capacity 100, reservation 20, source measurement 20, and destination measurement −10 produce source boundary 100 and destination coordinate 110. The loop is skipped because 110 is not less than 100. Overshoot correction changes count zero to 65535 and adds the initialized stride of zero to the boundary. The unchanged boundary of 100 passes acceptance. The outputs initially receive count 65535 and flag one. These measurements are not excluded by a supplied dependency contract.

3. **Execution following that corrected count.**  
   Under the preceding conditions, choose requested position 65535. If memory remains valid and every execution dependency succeeds, all 65535 loop operations are ordinary transfers, each followed by F-76b5978a0554. Source-side insertion then receives position zero. The planning count does not establish that the source actually contains enough material; that requirement depends on missing region and dependency contracts.

4. **Insertion inside the destination sequence.**  
   Capacity 100, reservation 20, source measurement 40, destination measurement 80, requested position one, and ordinary planning stride ten produce initial coordinates 20 and 80. One ordinary stride produces 30 and 70; the reservation stride produces 50 and 50. Count two is accepted. Execution transfers once, then inserts into the destination. Its position output uses the destination field captured before insertion. Reference acquisition receives source + 4.

5. **Planning rejection with status zero.**  
   Capacity 100, reservation 200, both measurements 100, and requested position zero produce initial coordinates 0 and 200. The reservation stride produces 200 and 0. Correction restores the source boundary to 200 and reduces the count to zero. The plan is rejected, all outputs are cleared, and status zero is returned.

6. **Transfer failure after earlier mutation.**  
   If one transfer and its following F-76b5978a0554 call complete, then a later transfer returns zero, all outputs are cleared and status 0x40 is returned. Restoration of earlier state is not promised.

7. **Full-width position with truncation.**  
   For an accepted zero-count plan and requested position 65536, source-side insertion receives position zero. No local rejection precedes this truncation.

8. **Negative displacement.**  
   An ordinary transfer that reads displacement −6 supplies source minus six to F-7bb443a5224b. No local range check protects that address.

## Uncertainties

The following remain **documented_only**:

- Complete region layouts, valid sizes, field meanings, and structural invariants.
- Exact overflow semantics for signed intermediate expressions and address arithmetic beyond valid memory.
- Valid input ranges for sizing and measurement dependencies, including whether negative measurements occur in supported use.
- Dependency behavior on malformed records, invalid addresses, and unsupported descriptors.
- The complete argument contract of F-c4c1afdb3fb3.
- Output aliasing permissions and concurrency guarantees.
- Allocation, ownership, lifetime, partial-mutation, and rollback semantics.
- Preconditions sufficient to guarantee planning termination and safe execution.

The zero-initialized retained stride and the resulting zero-iteration correction are established, not unresolved. What remains unknown is whether external contracts exclude inputs that reach that case. No such exclusion is supplied.

## Evidence

The supplied static function body establishes the widths, offsets, dependency-call arguments, retained-stride initialization, count masks, signed coordinate comparisons, output-store ordering, and explicit status returns described here.

The acceptance calculations follow from those operations under their stated assumptions. Redistribution, insertion, transfer, measurement, and removal-like terminology express inferred intent rather than independently established dependency semantics.

No runtime tests have been performed.
