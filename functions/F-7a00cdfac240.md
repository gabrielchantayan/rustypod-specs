<!-- Trusted provenance: F-7a00cdfac240; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7a00cdfac240

## Purpose

F-7a00cdfac240 processes signed-integer observations until it reads zero. Each nonzero observation is passed to F-076cc2c81536 together with a caller-supplied scalar. The observation’s numerical value also determines where the next observation will be read.

The operation maintains reporting storage relative to its root input. It marks entry before reading the first observation and publishes completion after reading zero. The reported result is the scalar increased by 3000; it does not depend on how many observations were processed.

These are behavioral descriptions, not application-level interpretations. The evidence does not establish that processing destroys objects, releases resources, empties a container, or confirms successful work by the dependency.

## Interface

### Inputs and output

| Item | Contract |
|---|---|
| Root address | Identifies the first observation and anchors the reporting locations. |
| Scalar | A signed-integer value passed unchanged to every dependency invocation and used to calculate the completion result. |
| Return | No value. Completion is represented through memory writes and ordinary return. |

The dependency receives an observation’s captured integer value, not necessarily the address from which that value was read. Whether this integer denotes an address, handle, resource identifier, or another application value is **documented_only**.

### Storage widths and layout

Let **W** be the storage size, in addressable bytes, of the signed-integer type used for observations, the scalar, and the result.

| Storage role | Starting offset from root | Access width |
|---|---:|---:|
| Initial observation | 0 | W bytes |
| State | 3W | One byte |
| Completion result | 6W | W bytes |
| Completion marker | 7W | One byte |

The state and marker writes affect one byte each. They are not assignments to entire integer-sized regions.

The supplied representation does not independently establish W, pointer width, byte order, signed-integer representation, or the number of bits in an addressable byte. If W is four bytes, the reporting offsets are 12, 24, and 28 bytes respectively. Those concrete offsets are conditional, not target-independent facts.

This is an accessed-storage layout, not a complete object definition. Allocation boundaries, other fields, padding, and unaccessed contents remain unspecified.

### Preconditions for ordinary execution

Every reached access requires suitable storage, lifetime, alignment, and permission under the applicable execution model. Root-relative address calculations must also be valid; numerical accessibility alone is insufficient.

For each nonzero observation whose dependency invocation returns normally:

- Adding 308 to the captured observation must be representable in the signed-integer type.
- Conversion of that sum to an address must be supported.
- The resulting address must permit a signed-integer read when continuation reaches it.

Completion requires a representable scalar adjustment by 3000 and valid reporting destinations.

Validity is required at the time of use. In particular, a dependency may invalidate storage that was valid at entry. Its normal return does not establish continued validity.

## Behavior

### Processing and termination

Zero is the sole established termination observation. A zero observation is not submitted to the dependency.

Every nonzero observation, including a negative value, requires one invocation of F-076cc2c81536. The arguments are the captured observation and the original scalar. No dependency return value is used.

The first observation comes from the root. For later observations, the address is obtained by adding 308 to the preceding nonzero observation’s numerical value and interpreting the result as an address. The displacement is not scaled by W and is not relative to the root.

The dependency invocation associated with an observation precedes the next observation read. This makes the next read sensitive to dependency changes to successor storage. Reading successor contents before the invocation would not preserve the established behavior.

Both the dependency argument and successor selection use the already captured observation. If the dependency changes the location that supplied that observation, the changed contents do not replace the captured value. They can affect a later read that revisits that location.

There is no established duplicate suppression, visited-location tracking, iteration bound, or cycle detection. Repeated values and repeated addresses remain eligible for processing. Termination depends on an actual observation of zero, not on repetition.

### Reporting lifecycle

The state byte is assigned 1 before the initial observation is read. Previous state, marker, and result contents do not control whether processing begins. Entry does not directly clear the marker or result.

Once zero is observed, the completion writes have a fixed order:

| Reporting event | Ordering requirement |
|---|---|
| Marker becomes 1 | Before the result write |
| Result receives scalar plus 3000 | After the marker write and before the final state write |
| State becomes 2 | After the result write |

Completion reporting also occurs when the initial observation is zero. The scalar adjustment is performed once per completed invocation, not once per nonzero observation.

No further observation is read after the terminating zero. Reporting may subsequently overwrite storage that supplied that zero, but the replacement does not restart processing.

Under valid ordinary execution without interfering effects, completion leaves state 2, marker 1, and the adjusted scalar in the result location. These values do not establish that dependency work succeeded or that any resource was released.

The reporting sequence is not an established atomic publication or synchronization protocol. In particular, an old marker and result can remain present while state is 1. Marker 1 alone therefore does not distinguish completion of the current invocation from stale completion data.

### Arithmetic boundaries

The successor and result adjustments are signed-integer arithmetic. The evidence establishes no explicit overflow detection, saturation, wrapping rule, or arithmetic-error response.

For a greatest representable signed integer **M**:

- A continued nonzero observation must be no greater than **M − 308** for its successor addition to be representable.
- The scalar must be no greater than **M − 3000** for completion arithmetic to be representable.

Because both constants are positive, adding them to an already representable operand cannot cause lower-bound underflow. A representable successor sum still need not identify usable storage.

Conditionally, if the signed-integer maximum is 2,147,483,647, the respective upper bounds are 2,147,483,339 and 2,147,480,647.

The successor calculation belongs to continuation after the corresponding dependency invocation returns normally. Completion arithmetic follows the terminating observation and marker assignment. This ordering does not establish recoverable partial effects for invalid arithmetic. If signed overflow invalidates execution under the applicable model, earlier writes are not thereby guaranteed to remain observable.

### Failure and nontermination

There is no established error return, invalid-address sentinel, rollback, recovery action, or direct cleanup path.

An unusable successor address does not behave as a zero observation. It prevents the ordinary continuation described by this contract.

If a dependency invocation does not return normally, the corresponding successor read is not reached through the ordinary path. Consequences of exceptions, nonlocal control transfer, or process termination are **documented_only**.

If observations remain nonzero indefinitely and execution continues, completion reporting is never reached. No progress or termination guarantee is established.

## Side effects

### Direct activity

For an ordinarily completed invocation with **N** nonzero observations:

| Activity | Count |
|---|---:|
| Dependency invocations | N |
| Signed-integer observation reads | N + 1 |
| State-byte assignments | 2 |
| Marker-byte assignments | 1 |
| Signed-integer result assignments | 1 |

These are semantic counts. They exclude dependency activity and do not prescribe machine instructions, physical memory transactions, or atomic accesses.

Previous reporting values are not directly consulted as control information. They may nevertheless become observations if a computed successor validly designates their storage.

### Aliasing and mutation

The observations need not occupy distinct allocations. Subject to the execution model’s access rules, successor storage may coincide with the root, previously visited storage, reporting storage, or other memory.

A byte assignment can change a legally overlapping integer representation. Assigning that byte the value 1 does not generally make the overlapping integer equal to 1; the effect depends on byte order and the rest of the representation.

The initial observation and state byte occupy distinct regions in the stated root-relative layout. Their overlap is not implied by that layout.

Observation reads do not directly modify their source storage. This is not a preservation guarantee: dependency effects and reporting writes may modify storage later encountered by the operation.

### Ownership and external state

No direct allocation, deallocation, reference-count adjustment, or ownership transfer is established. Submission of an integer to the dependency does not establish borrowing, retention, consumption, or release of an associated resource.

No separately identified global object is directly accessed. Computed addresses may still designate global storage, and the dependency may change globals, lifetimes, allocation state, or external resources.

There is no established locking, exclusion, reentrancy guard, or concurrency guarantee. A normally returning dependency must not be assumed to preserve the root storage, reporting destinations, or successor storage.

## Dependencies

### F-076cc2c81536

This is the sole explicit function dependency.

Its established relationship to F-7a00cdfac240 is:

- One invocation for each nonzero observation, in observation order.
- The captured observation as the first argument.
- The unchanged scalar as the second argument.
- Normal return before the associated successor read.
- No use of a returned value.

The following dependency properties are **documented_only**:

- Application meanings and additional requirements of its arguments.
- Memory and global-state reads or writes.
- Allocation, retention, release, and ownership effects.
- External I/O, callbacks, and reentrant execution.
- Synchronization behavior.
- Failure mechanisms and termination guarantees.

Neither the existence of the call nor its normal return establishes purity, successful processing, storage preservation, or resource release.

## Acceptance examples

These are conditional criteria derived from static evidence, not executed tests. Ordinary-completion examples require valid accesses and address conversions, representable arithmetic, normally returning dependencies, and no interference beyond that expressly described.

### Immediate zero

If the initial observation is zero when read after the entry-state assignment, scalar 25 causes no dependency invocation. Completion publishes marker 1, result 3025, and state 2 in the required order.

Skipping reporting because there were no nonzero observations would violate the contract.

### Two nonzero observations

If the actual observation sequence consists of two nonzero values followed by zero, exactly two dependency invocations occur. Each receives its corresponding captured value and the same scalar.

With scalar 10, the result is 3010. Observation count and magnitude do not change that adjustment.

### Value-directed addressing

If a captured observation is 4096, its successor address is numerically 4404, provided the arithmetic and address conversion are valid. It is not the root plus 308 and is not an advance of 308 integer elements.

The successor contents are read only after the invocation receiving 4096 returns normally.

### Dependency changes successor contents

Suppose the selected successor contains a nonzero integer before the dependency invocation. If the dependency replaces it with zero before returning, the next observation terminates processing.

Conversely, replacing a successor zero with a nonzero value requires that value to be processed. A pre-invocation snapshot of successor contents would fail these criteria.

### Dependency changes the current source

Suppose the dependency changes the storage from which its first argument was read, while successor storage lies elsewhere. The argument and selected successor still derive from the captured observation.

The changed source contents affect processing only if a subsequent relevant access reaches them.

### Repeated observations

Three separately read occurrences of the same nonzero value require three dependency invocations and select the same successor address.

Dependency mutation may make a later read at that address obtain zero. Equal values therefore justify neither deduplication nor an unconditional claim of nontermination.

### Stale reporting data

An invocation beginning with state 2, marker 1, and an older result still starts processing by assigning state 1. The marker and result are not directly cleared.

Their continued presence during processing is insufficient evidence that this invocation has completed.

### Result storage supplies zero

If a valid successor read obtains zero from the completion-result location, processing terminates. Completion may then replace that zero with a nonzero adjusted scalar.

That replacement causes no additional observation or dependency invocation.

### Negative values and upper boundary

An initial zero with scalar −3000 produces result zero. The reported zero is not another observation.

Conditionally, with signed maximum 2,147,483,647, scalar 2,147,480,647 produces that maximum. Increasing the scalar by one has no established wrapped result or explicit error response.

A negative nonzero observation still requires a dependency invocation. Whether its adjusted value converts to usable storage remains a separate precondition.

### Invalidated successor storage

If the dependency invalidates selected successor storage and returns normally, the subsequent read lacks its required validity. Ordinary completion is not established.

Neither invalidation nor abnormal dependency exit establishes rollback, cleanup, or preservation of reporting contents.

## Uncertainties

The following remain **documented_only**:

- Concrete integer and pointer widths, byte order, signed representation, and addressable-byte width.
- Target-specific integer-to-address conversion and calling convention.
- Complete object layout, allocation boundaries, lifetime, alignment, and permitted aliasing.
- Consequences of invalid accesses, unrepresentable arithmetic, and abnormal control transfer.
- Dependency effects on memory, globals, ownership, resources, and reentrant execution.
- Concurrent-use guarantees and application meanings of values and constants.

**Inferred intent:** state 1 plausibly denotes work in progress; state 2 and marker 1 plausibly denote completion. These interpretations do not establish successful dependency work, ownership disposition, or synchronization.

“Observation,” “successor,” and “reporting” identify behavioral roles. They do not establish declared field names or a particular container design.

## Evidence

**Established static evidence:** the supplied representation establishes two inputs, no returned value, signed-integer observations, integer-scaled reporting locations, byte-wide state and marker assignments, and one explicit dependency. It supports zero termination, value-directed successor addressing with adjustment 308, unchanged scalar forwarding, completion adjustment 3000, and the ordering constraints described above.

No explicit validation, traversal bound, duplicate suppression, allocation, release, rollback, or synchronization appears in this operation’s direct behavior. These absences do not constrain hidden dependency effects.

**Conditional deductions:** concrete byte offsets, numerical range limits, completed-execution access counts, and final reporting contents depend on the stated representation and execution conditions.

**Evidence limits:** no independent target-platform definition or dependency implementation is available. No runtime tests have been performed.
