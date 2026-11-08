<!-- Trusted provenance: F-21aefc29be9e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-21aefc29be9e

## Purpose

F-21aefc29be9e traverses storage associated with an input object. Each successful traversal attempt triggers two 32-bit derivations, three element-directed calls, and an update to a carried 64-bit state.

The three element-directed calls independently read a shared 32-bit yielded-element slot when forming their arguments. The function does not capture one element reference for use by all three calls. They target the originally yielded element only if the slot retains that value through their respective argument-evaluation points.

The derivations depend on the carried state, an input-object field, and two initialization-result slots. The final element-directed call receives two differences between the derived values and values extracted from a per-element record.

An arrangement or positioning operation is plausible inferred intent. Coordinate meanings, units, numeric representations, and the arithmetic performed by dependencies are not established.

The function returns no explicit value. Its observable behavior consists of dependency calls, object-field and local-slot accesses, carried-state transitions, and normal-exit traversal cleanup.

## Interface

### Input and object accesses

The function accepts one 32-bit argument used as an object reference.

| Storage | Width | Established access |
|---|---:|---|
| Input object at offset 0xE8 | 32 bits | Read once in each successful iteration, before the two derivation paths. |
| Input object at offset 0xF0 | 64 bits | Read once during initialization to establish the initial carried state. |
| Associated object at offset 0x04 | 32 bits | Read after F-583dc5fa4d66 returns its reference. |

F-583dc5fa4d66 receives the input reference and returns the associated-object reference. That returned reference is also passed to traversal initialization.

The evidence establishes neither a complete object layout nor valid address ranges, alignment, null handling, or reference ownership. No explicit null check precedes these field reads.

### Output and local storage

There is no explicit return value, status flag, or error result.

The function provides dependencies with addresses into these local storage regions:

- A 20-byte initialization record.
- A 16-byte per-element record.
- A 20-byte traversal record.
- Two adjacent 32-bit initialization-result slots.
- Two adjacent 32-bit per-element-result slots.
- One 32-bit yielded-element slot.

F-473cdebbc402 receives the address of the first slot in each result pair. The caller subsequently consumes both adjacent slots. The supplied evidence does not independently establish the dependency’s write extent or initialization guarantees.

F-6e4651b532dc receives the yielded-element slot’s address. Whether it retains that address, or makes it accessible to other dependencies, is unknown. Consequently, later calls must not be assumed to leave that slot unchanged.

### Width and argument conventions

Several dependencies return 64-bit values. When this function forwards such a value as two arguments, it supplies the low 32-bit word first and the high 32-bit word second.

This establishes word selection and argument position. It does not establish floating-point interpretation, signedness of the complete 64-bit quantity, or a general memory-endianness contract.

The final two differences use 32-bit operands represented as signed integers in the supplied representation. No widening, saturation, range validation, or explicit overflow handling is visible. Results outside the signed 32-bit range remain unresolved rather than being specified as wrapping.

### Preconditions

For normal completion:

- All explicitly accessed object fields must be readable.
- Dependency calls must return normally.
- Records and output slots must contain usable values when consumed.
- Each reference passed to a dependency must satisfy that dependency’s requirements at that call.

These are requirements for the described execution, not conditions visibly checked by the function. If dependencies retain local-storage addresses, their retention and lifetime rules must also be satisfied; those rules are not supplied.

## Behavior

### Initialization

The following order is established:

1. F-1e77f5f5600b is called without explicit arguments.
2. F-960363c07b32 receives the input reference and the 20-byte initialization record.
3. F-473cdebbc402 receives that record and the address of the first initialization-result slot.
4. F-583dc5fa4d66 receives the input reference and returns the associated-object reference.
5. The 32-bit field at offset 0x04 of that associated object is read.
6. F-b3a54d881220 receives that field and returns a 64-bit value.
7. F-596e18a33aec receives two fixed-address 32-bit data operands, followed by the low and high words of the preceding result. Its 64-bit result supplies the retained recurrence operand.
8. The input object’s 64-bit field at offset 0xF0 is read as the initial carried state.
9. F-538044b67754 receives the traversal record, the associated-object reference, and the 32-bit pattern 0xFFFFFFFE.

The final traversal-initialization argument corresponds to −2 under signed 32-bit interpretation. Its semantic meaning is unknown.

The contents and meanings of the two fixed-address operands are not supplied. They must not be treated as known numeric constants.

The offset-0xF0 read occurs after F-596e18a33aec returns. Therefore, an earlier dependency can affect the value observed there if it modifies that field. The function does not snapshot this field at entry.

### Traversal attempts and yielded-slot reads

Before each traversal attempt, the current carried state is preserved as low and high 32-bit words. F-6e4651b532dc then receives the traversal record and the address of the yielded-element slot.

A zero return terminates the loop. Any nonzero return enters the body; continuation is not limited to a return value of one.

On a successful attempt, the function does not immediately copy the yielded-element slot into a stable element reference. The slot is read separately when forming arguments for:

1. F-39509ed7bd5a.
2. F-960363c07b32.
3. F-5cf6facacac5.

Many dependency calls occur between the traversal call and these reads. There are also intervening calls between the three reads themselves. Because address retention and dependency side effects are unknown, none of the three slot values is guaranteed to equal the value supplied by the successful traversal call or either of the other two slot values.

On a terminating attempt, the loop body does not consume the yielded-element slot. Traversal cleanup still occurs.

### First derived value

For a successful iteration, the function first reads the input object’s 32-bit field at offset 0xE8 and passes it to F-b3a54d881220. The resulting 64-bit value is retained for both derivation paths in that iteration.

The first path then:

1. Passes the saved carried state, low word first, to F-b0e125d76081.
2. Passes that 64-bit result first and the converted offset-0xE8 value second to F-c4b34006a13c.
3. Reads the first initialization-result slot and passes it to F-b3a54d881220.
4. Passes the converted initialization value first and the preceding F-c4b34006a13c result second to F-1b22d7e17791.
5. Calls F-baa332e06e32 without explicit arguments and retains its 32-bit return as the first derived value.

The function does not explicitly consume a direct return from F-1b22d7e17791. Its relationship to the following no-argument call is unresolved. An implicit result channel is possible, but its mechanism and numeric semantics are not established.

### Second derived value

The second path:

1. Passes the same saved carried state to F-f7c21c23a4e8.
2. Passes that result first and the same converted offset-0xE8 value second to F-c4b34006a13c.
3. Reads the second initialization-result slot and passes it to F-b3a54d881220.
4. Passes the converted initialization value first and the second F-c4b34006a13c result second to F-8a693c103d1c.
5. Calls F-baa332e06e32 without explicit arguments and retains its 32-bit return as the second derived value.

The initialization-result slots are read at their respective conversion calls, not captured together at initialization. Their addresses have been exposed to a dependency, so their continued stability is not independently guaranteed.

Neither path establishes that F-baa332e06e32 rounds, truncates, or performs any particular numeric conversion.

### Element-directed calls and differences

After both derived values have been obtained:

1. The yielded-element slot is read to form the argument to F-39509ed7bd5a.
2. After that call returns, the slot is read again to form the object argument to F-960363c07b32. Its other argument is the 16-byte per-element record.
3. F-473cdebbc402 receives that record and the address of the first per-element-result slot.
4. The yielded-element slot is read again to form the first argument to F-5cf6facacac5. Its second argument is the first derived value minus the first per-element-result value. Its third argument is the second derived value minus the second per-element-result value.

These are distinct slot reads, not repeated uses of a captured reference. In particular:

- F-960363c07b32 need not receive the reference previously passed to F-39509ed7bd5a.
- F-5cf6facacac5 need not receive the reference used to obtain the per-element record.
- The two differences use the extracted per-element-result slots even if the final element reference differs from the record-production reference.

The evidence establishes call order and argument positions, but not a finer ordering among reads used to form different arguments of the same call.

### Carried-state update

After F-5cf6facacac5 returns, F-8a693c103d1c receives:

- The saved carried state as its first 64-bit operand.
- The retained recurrence operand as its second 64-bit operand.

Its 64-bit result becomes the state for the next traversal attempt.

Both derivation paths and this recurrence call use the state saved before the current traversal call. Intermediate derivation results do not replace that state.

The offset-0xE8 field is reread for every successful iteration. The offset-0xF0 field is not reread. Later modifications to offset 0xF0 therefore do not directly replace the carried state.

### Normal termination and errors

When a traversal attempt returns zero, F-944b595e25e9 receives the traversal record once, and the function returns.

There is no visible early-return branch or explicit error-handling path. The traversal result distinguishes zero from nonzero; no separate error classification is established.

Cleanup following an abnormal dependency exit, invalid access, or arithmetic failure is not established.

## Side effects

The function visibly writes local working state but performs no direct writes to the input-object fields. Dependencies receive object references and local-storage addresses and may modify objects, globals, records, or retained slots according to contracts not supplied here.

Established ordering includes:

- Both derived values precede all three element-directed calls.
- F-39509ed7bd5a precedes per-element record production.
- Record extraction precedes the difference-based call.
- The difference-based call precedes recurrence.
- Cleanup follows the terminating traversal call.

If F-39509ed7bd5a modifies an object subsequently used for record production, that modification can affect the extracted values. This observation is conditional: record production may instead target a different object because the yielded-element slot is reread.

No explicit allocation, deallocation, reference-count operation, logging, or external I/O appears in the body. Such effects inside dependencies remain unknown. F-944b595e25e9 has a cleanup-shaped position, but its precise release behavior is not established.

There is no visible locking or isolation mechanism. No dependency is assumed to preserve globals, referenced objects, or address-exposed local storage. Exposing local storage does not itself establish ownership transfer or permission to retain it beyond its lifetime.

## Dependencies

| Opaque ID | Established role |
|---|---|
| F-1e77f5f5600b | Initial no-argument call. |
| F-960363c07b32 | Receives an object reference and a record address; used for initialization and per-element record production. |
| F-473cdebbc402 | Receives a record and the first address of a subsequently consumed pair of 32-bit slots. |
| F-583dc5fa4d66 | Returns the reference used for the offset-0x04 read and traversal initialization. |
| F-b3a54d881220 | Maps one 32-bit operand to a 64-bit result. |
| F-596e18a33aec | Produces the recurrence operand from four 32-bit arguments. |
| F-538044b67754 | Receives traversal storage, the associated-object reference, and 0xFFFFFFFE. |
| F-6e4651b532dc | Receives traversal storage and the yielded-slot address; returns the continuation indicator. |
| F-b0e125d76081 | First transformation of the saved carried state. |
| F-f7c21c23a4e8 | Second transformation of the saved carried state. |
| F-c4b34006a13c | Combines each transformation result with the converted per-iteration field. |
| F-1b22d7e17791 | First derived-value operation, preceding a no-argument result call. |
| F-8a693c103d1c | Second derived-value operation and, separately, recurrence update. |
| F-baa332e06e32 | Supplies each retained 32-bit derived value without explicit arguments. |
| F-39509ed7bd5a | Receives the yielded-slot contents at its own argument-evaluation point. |
| F-5cf6facacac5 | Receives the yielded-slot contents at its own argument-evaluation point and two differences. |
| F-944b595e25e9 | Receives traversal storage on normal termination. |

These roles do not establish independent dependency semantics, ownership rules, or global-state preservation.

## Acceptance examples

These are conditional static-contract examples, not executed tests.

### Empty traversal

If the first traversal call returns zero, initialization still occurs, including the offset-0x04 and offset-0xF0 reads. Cleanup is called once.

There is no offset-0xE8 read, derived-value computation, element-directed body call, or recurrence update.

### One successful iteration

If traversal returns nonzero and then zero, one complete body execution occurs. The recurrence update occurs before the terminating traversal attempt even though no further body executes. Cleanup follows termination.

### Stable yielded-element slot

Suppose a successful traversal supplies reference A, and the slot remains A through all three element-directed argument-evaluation points. Then F-39509ed7bd5a, per-element F-960363c07b32, and F-5cf6facacac5 all receive A.

This conclusion requires slot stability; traversal success alone is insufficient.

### Changed yielded-element slot

Suppose a successful traversal supplies A. Assume a dependency with access to the retained slot address changes it to B before the F-39509ed7bd5a argument is formed, to C before the per-element F-960363c07b32 argument is formed, and to D before the F-5cf6facacac5 argument is formed.

The three calls receive B, C, and D respectively. The final differences still use the values extracted from the record produced with C. The function does not enforce a common target or reacquire a record for D.

This example does not assert that any dependency actually retains or modifies the slot.

### Difference direction and argument order

Suppose the retained derived values are 100 and 70, and the per-element-result slots contain 12 and 9 when consumed. F-5cf6facacac5 receives its independently read reference, then 88, then 61.

This establishes subtraction direction and argument placement, not how the derived values were obtained.

### Changing fields and recurrence

If an intervening dependency changes offset 0xE8 between two successful iterations, the second iteration reads the changed field. Within each iteration, both derivation paths use that iteration’s single converted field value.

If the first recurrence call returns a new 64-bit pair, the next iteration’s transformations use that pair rather than rereading offset 0xF0.

## Uncertainties

The following remain **documented_only**:

- Object types, coordinate meanings, units, and intended traversal order.
- Numeric representation and arithmetic semantics of the 64-bit values.
- The result-transfer relationship preceding F-baa332e06e32.
- Contents and mutability of the fixed-address operands.
- Complete record layouts and output-initialization guarantees.
- Retention of the yielded-slot address and possible changes between its separate reads.
- Whether the three element-directed calls actually target one object.
- Stability of address-exposed initialization and per-element result slots.
- Signed-overflow behavior of the final differences.
- Null handling, alignment, aliasing, reference validity, and dependency errors.
- Ownership, allocation, release, global-state effects, reentrancy, concurrency, and abnormal-exit cleanup.

An arrangement interpretation remains inferred intent, not an established guarantee.

## Evidence

This contract uses only the supplied static representation and opaque dependency mapping. Established observations include storage widths and extents, field offsets, call order, argument positions, separate yielded-slot reads, value reuse, loop branching, recurrence placement, and normal-exit cleanup placement.

The supplied body independently shows a yielded-slot read at each of the three element-directed calls and no intervening snapshot that unifies their references. Whether those reads observe equal values depends on behavior outside the established local operations.

No dependency implementations, independent dependency contracts, or runtime traces establish the unresolved semantics. No runtime tests have been performed.
