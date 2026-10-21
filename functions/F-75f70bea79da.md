<!-- Trusted provenance: F-75f70bea79da; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-75f70bea79da

## Purpose

F-75f70bea79da computes an updated integer from an initial state and three scalar factors, then publishes the updated and original values through three supplied destinations.

The mathematical update is the original state plus the product of the three factors. The function first writes the updated value to the state destination, then writes the same value to a second destination, and finally writes the original state to a third destination.

The destinations are not required by the supplied body to be distinct. Exact aliasing changes the final contents because the final write restores the original state wherever the third destination overlaps a preceding destination.

This contract describes established static behavior under valid-storage and representable-arithmetic conditions. It does not establish a particular machine representation, overflow result, ownership convention, or machine-level access sequence. No runtime tests have been performed.

## Interface

The function takes six positional inputs and returns no value.

| Position | Contract role | Input or output behavior |
|---|---|---|
| 1 | State destination | Pointer to an integer object. Its initial value is read, and the computed value is subsequently written to it. |
| 2 | Updated-value destination | Pointer to an integer object receiving the computed value. Its previous contents are not consumed. |
| 3 | Original-value destination | Pointer to an integer object receiving the captured initial state. Its previous contents are not consumed. |
| 4 | First factor | Signed integer scalar used in the update product. |
| 5 | Second factor | Signed integer scalar used in the update product. |
| 6 | Third factor | Signed integer scalar used in the update product. |

These role names describe accesses, not ownership or distinctness requirements.

### Integer widths and layout

The supplied declarations use the same signed integer type for all three scalar factors, both intermediate values, and all destination elements. The body contains no explicit conversion to a wider arithmetic type.

The concrete width is not independently established. Let **W** denote the applicable signed integer width. If the environment uses a conventional W-bit two’s-complement representation without padding, its value range is:

−2^(W−1) through 2^(W−1)−1.

A 32-bit range is therefore conditional, not an established target property. Storage size, alignment, byte order, padding, and concrete object representation remain unresolved.

Each pointer is used to access one integer object. There is no indexing, traversal, length argument, capacity argument, or aggregate layout.

### Successful-behavior preconditions

The ordinary behavior described below requires:

- The state destination designates a live, suitably aligned integer object whose initial value can be read and whose storage can be written.
- The updated-value and original-value destinations designate live, suitably aligned writable integer objects.
- Each object remains valid through every access made to it.
- The initial state is a valid value of the applicable signed integer type.
- Both multiplication results and the final addition result are representable under the applicable arithmetic semantics.
- Concurrent or external activity does not violate the execution environment’s access and memory-model requirements.

The second and third destinations need not have meaningful initial values merely for this operation, because their previous contents are not read. Their storage must nevertheless be valid for the required writes.

No ownership precondition is established. Supplied pointers may refer to owned or borrowed storage; the initial ownership arrangements are unknown.

## Behavior

### Value computation

The function captures the initial integer from the state destination before writing any destination. That captured value supplies both the starting state for the calculation and the value later written to the original-value destination.

For arithmetic grouping, the function:

1. Forms the product of the third and second factors.
2. Multiplies that intermediate product by the first factor.
3. Adds the captured initial state.

This grouping constrains representability even though mathematical multiplication is associative. A final mathematical result within range does not make an out-of-range intermediate product acceptable.

There is no explicit widening accumulator, overflow check, saturation, clamping, or modular reduction. Under representable arithmetic, ordinary signed integer arithmetic determines the updated value.

Negative states and negative factors require no separate branch. A zero factor also introduces no branch. In particular, a zero first factor does not remove the representability requirement for the preceding multiplication of the third and second factors.

### Logical access order

The supplied body establishes this logical sequence:

| Order | Access |
|---|---|
| 1 | Read the initial integer from the state destination. |
| 2 | Write the computed value to the state destination. |
| 3 | Write that same computed value to the updated-value destination. |
| 4 | Write the captured initial value to the original-value destination. |

The computation uses the captured initial value, not a later reread of the state destination. The original-value write likewise uses that captured value.

The prior contents of the updated-value and original-value destinations are not read. All three destination writes appear unconditionally in the supplied body, including when the computed value equals the initial state.

These are logical operations, not guaranteed counts of machine transactions. The evidence supplies no volatile or atomic access qualification, memory barrier, synchronization primitive, or instruction-level ordering guarantee. An implementation may realize ordinary accesses differently while preserving the behavior required by its execution model.

### Exact aliasing

When two or more destinations designate the same valid integer object, the last logical write to that object determines its final value.

| Destination relationship | Final contents |
|---|---|
| All three are distinct | State and updated-value destinations contain the computed value; original-value destination contains the initial state. |
| State and updated-value destinations coincide; original-value destination is distinct | The shared object contains the computed value; the distinct object contains the initial state. |
| State and original-value destinations coincide; updated-value destination is distinct | The shared object ends with the initial state; the distinct updated-value destination contains the computed value. |
| Updated-value and original-value destinations coincide; state destination is distinct | The shared object ends with the initial state; the distinct state destination contains the computed value. |
| All three coincide | The sole object ends with its initial state. |

When all three destinations coincide, the final state is unchanged, but the supplied operation sequence still includes writes of the computed value before the write of the initial value. This does not guarantee that each write is separately observable at machine level.

Partial byte overlap between different purported integer objects is not established as supported. It must not be treated as an exact-alias case without additional object-layout and access-legality information.

### Invalid inputs and arithmetic failures

The body contains no pointer validation, explicit error return, recovery branch, or rollback mechanism.

A null pointer is not checked before its corresponding access. Invalid, expired, insufficiently aligned, or nonwritable destination storage falls outside the established successful-behavior preconditions. The evidence does not establish a required trap, diagnostic, partial result, or other outcome.

Signed overflow has no specified result in this contract. The body does not establish wrapping arithmetic. Neither wrapped destination values nor a particular subset of completed writes is required when an arithmetic intermediate or the final sum is out of range.

Because there is no rollback mechanism, the contract also provides no transactional guarantee if an access cannot complete. It does not promise that earlier effects are undone, nor does it specify fault-time destination contents.

## Side effects

The directly evidenced memory effects are one logical read through the state destination and three ordered logical writes through the supplied destinations.

The body contains no direct global-variable access, allocation, deallocation, I/O, callback, or external function invocation. Supplied destinations may nevertheless refer to global or shared storage, so their contents can change through these pointer accesses.

No allocation, deallocation, or explicit persistent retention of a supplied pointer occurs. The function does not establish or transfer ownership, and it does not extend the lifetime of referenced storage. Initial ownership, borrowing arrangements, and external ownership conventions remain unknown.

The function returns no value and supplies no explicit status channel. Its ordinary results are the values written to the destinations.

No atomicity, synchronization, cross-thread visibility, or all-or-nothing update guarantee is established.

## Dependencies

There are no callee dependencies in the supplied body, so there are no additional opaque function IDs to list.

Successful execution depends on environmental rules for:

- Signed integer representation, width, and arithmetic.
- Pointer validity, object lifetime, and alignment.
- Legal reads and writes through the supplied pointers.
- Aliasing and overlapping storage.
- Optimization, concurrency, and fault handling.

These are environmental requirements and unresolved semantics, not evidence of hidden helper calls.

## Acceptance examples

The following examples are conditional consequences of the static evidence, not observed executions. Unless stated otherwise, they assume valid storage, compatible object layout, no interfering activity, and representable intermediate arithmetic.

### Distinct destinations

For an initial state of 10 and first, second, and third factors of 2, 3, and 4:

- The first product is 4 × 3 = 12.
- The second product is 12 × 2 = 24.
- The computed value is 24 + 10 = 34.
- The state destination ends with 34.
- The updated-value destination ends with 34.
- The original-value destination ends with 10.

The previous contents of the latter two destinations do not affect these results.

### Negative increment

For an initial state of 7 and factors of −2, 3, and 4, the increment is −24 and the computed value is −17.

With distinct destinations, the final values are −17 in the state destination, −17 in the updated-value destination, and 7 in the original-value destination.

### Zero increment with valid intermediate arithmetic

For an initial state of 9 and factors of 0, 3, and 4, the multiplication sequence produces 12 and then 0. The computed value is 9.

All three distinct destinations end with 9. The supplied body still contains all three writes; it does not select a no-write path.

### Exact aliasing

For an initial state of 10 and factors of 2, 3, and 4, the computed value is 34:

- If only the state and updated-value destinations coincide, their shared object ends with 34; the distinct original-value destination ends with 10.
- If only the state and original-value destinations coincide, their shared object ends with 10; the distinct updated-value destination ends with 34.
- If only the updated-value and original-value destinations coincide, their shared object ends with 10; the distinct state destination ends with 34.
- If all three coincide, the sole object ends with 10.

These outcomes depend on the final write carrying the captured initial state rather than rereading a destination after its update.

### Conditional 32-bit arithmetic boundaries

If the applicable signed integer type has the conventional 32-bit range:

- An initial state of 2,147,483,646 with all factors equal to 1 produces 2,147,483,647. With distinct destinations, both updated destinations receive that value and the original-value destination receives 2,147,483,646.
- An initial state of 2,147,483,647 with all factors equal to 1 requires an out-of-range addition. No computed value or final destination contents are required by this contract.
- Factors of 0, 50,000, and 50,000 require the first multiplication to produce 2,500,000,000, outside that signed range. The zero first factor does not make this an accepted zero-increment case.

### Invalid destination

If the original-value destination is null, the successful-behavior preconditions are not met. The absence of a null check does not establish a particular failure signal or guarantee that preceding destination writes are observable.

## Uncertainties

### Established static evidence

The supplied body establishes:

- Six positional inputs and no returned value.
- One initial state read.
- The stated arithmetic grouping.
- Three ordered logical writes.
- No reads of the second or third destination’s prior contents.
- No explicit validation, error handling, allocation, deallocation, persistent pointer retention, or callees.

### Inferred intent

Advancing an integer state while publishing both its updated and original values is a plausible intent. No domain meaning is established for the state or factors. They are not evidenced as byte counts, dimensions, addresses, units, or capacities.

### documented_only

The following remain unresolved:

- Concrete integer width, storage size, representation, alignment, byte order, and calling convention.
- Any environment-specific definition of overflow behavior.
- Machine-level access counts, optimization effects, atomicity, and cross-thread visibility.
- Outcomes for invalid pointers, unsupported overlap, access faults, or asynchronous interference.
- Initial storage ownership, borrowing rights, and external ownership conventions.
- Any caller-side rule requiring distinct destinations.

These unknowns do not justify assuming a particular platform, assigning caller ownership, or strengthening the function’s guarantees.

## Evidence

This contract is based solely on the supplied static function body and declarations for F-75f70bea79da. The prior draft and reviewer feedback are not independent evidence of behavior.

No runtime execution, binary inspection, dependency implementation, caller behavior, or external documentation has been examined. The acceptance examples are derived from the established logical operations under their stated conditions; they are not test results.
