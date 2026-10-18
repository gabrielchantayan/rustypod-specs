<!-- Trusted provenance: F-adc440f62ed1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-adc440f62ed1

## Purpose

F-adc440f62ed1 invokes an operation selected through the supplied object, then writes a supplied byte to a location derived from the original object reference.

The central requirement is sequencing: the selected operation must return normally before this function performs its byte write. The write is not conditional on a success indication from that operation.

The evidence establishes dispatch followed by an update. It does not establish the application-level purpose of either action. Interpreting the call as a notification, or the byte as a status or mode, would be inferred intent rather than a supported semantic guarantee.

## Interface

### Inputs

| Input | Established role |
|---|---|
| Object reference | Identifies the initial integer-valued dispatch information, is passed to the selected operation, and is retained as the basis of the final destination calculation. |
| Supplied byte | Provides the eight-bit representation written after the selected operation returns normally. |

The supplied byte is not visibly validated, compared, normalized, or transformed. Its signedness and application-level meaning are unspecified. It is not an explicit argument to the selected operation.

### Relevant storage

Only three locations in the object-related storage are established:

| Location | Use |
|---|---|
| Beginning of the referenced object | Supplies an integer value used to calculate the dispatch-entry address. |
| Address derived from that integer value with a displacement of 20 bytes | Supplies the callable target. |
| Address derived from the integer-converted original object reference with a displacement of 102 bytes | Receives the supplied byte after normal return from the call. |

These are byte displacements, not table indices. In particular, the dispatch displacement does not require multiplication by a pointer width.

The evidence does not establish a complete object layout, the extent of a dispatch table, or the purpose of surrounding storage. It also does not establish that the dispatch storage and the final destination occupy separate allocations.

### Widths and arithmetic

The supplied value and final write are one byte wide, representing eight bits.

The initial object value is read using the integer type represented in the supplied body. Both address calculations involve that integer type: the dispatch calculation adds 20 to the value read from the object; the destination calculation converts the original object reference to an integer and adds 102.

The supplied body does not independently define the integer type’s bit width or the execution environment’s pointer width. A 32-bit interpretation must therefore remain conditional rather than becoming a portability guarantee.

The callable-target read uses the execution environment’s callable-pointer representation. Its width is not established merely by the integer type used in the preceding calculation.

Neither calculation includes a visible bounds or overflow check. The contract does not prescribe modulo arithmetic, lossless pointer conversion, or a particular outcome for an unrepresentable result. Defined completion requires the actual conversion and arithmetic rules to produce valid addresses.

### Result

No explicit return value is supplied.

Normal completion establishes that the selected operation returned normally and that this function subsequently performed its byte assignment. It does not establish that the selected operation accomplished an application-level objective.

### Preconditions for defined completion

The following conditions are required but not checked locally:

- The original object reference permits the initial integer-width read.
- The dispatch address calculation produces readable callable-target storage.
- The selected target supports the actual three-argument invocation.
- The selected operation’s own preconditions hold.
- The original-reference conversion and destination arithmetic produce a valid address.
- The destination remains writable for one byte after the selected operation returns.

No null-reference check, target validation, or lifetime validation is visible.

## Behavior

### Selection and call

The runtime-selected callable is designated **F-opaque-01** in this contract. This is a local dependency label, not a claim that every execution selects the same implementation.

Selection depends on the integer value read from the beginning of the object. That value determines where the callable target is read, using the 20-byte displacement described above.

There is one visible call path. The supplied byte does not select the target or control whether the call occurs. No alternative target or fallback behavior is present.

The selected operation receives three explicit arguments:

| Position | Argument |
|---|---|
| First | The original object reference. |
| Second | A value supplied by a fixed global datum. |
| Third | Integer zero. |

These positions describe the call interface, not the order in which argument expressions are evaluated.

The global-derived argument’s representation is not available in the supplied material. It must not be classified as a pointer, handle, scalar status, or ownership token without additional evidence. Its width is likewise unresolved.

The initial object read must precede the dependent callable-target read. The relative order between dispatch evaluation and obtaining the global-derived argument is not established. Both must be resolved for the invocation to occur.

### Post-call assignment

Once F-opaque-01 returns normally, the function assigns the supplied byte to the destination derived from the original object reference.

No visible condition intervenes. In particular, the assignment does not depend on:

- A result returned by F-opaque-01.
- The current value of the destination byte.
- A comparison involving the supplied byte.
- A second evaluation of the global datum.
- A second dispatch lookup.

The final destination is not taken from a dependency result. Changing the object’s dispatch information during the call therefore does not redirect this function’s destination calculation.

The function does not explicitly read the previous destination byte. An already-equal value does not eliminate either the call or the represented assignment.

### Observable sequencing

The selected operation executes before this function’s byte assignment. The assignment precedes normal return from this function.

This ordering does not guarantee that the dependency sees the destination’s entry value. The dependency may change that location itself, and another actor may also modify it.

If F-opaque-01 writes a different byte to the destination and then returns normally, this function subsequently writes the supplied byte, provided the original destination remains valid. The dependency’s earlier write does not suppress or replace the later assignment.

The established order is local execution order. No synchronization primitive, atomic protocol, or inter-thread visibility guarantee is shown.

### Noncompletion and invalid execution

If F-opaque-01 does not return normally, this function does not reach its subsequent assignment through the visible continuation. The dependency may nevertheless have produced effects before control stopped or transferred elsewhere.

If the dependency invalidates the original object and then returns, the function still proceeds toward the destination write. There is no visible mechanism to detect that invalidation, obtain replacement storage, or restore the object’s lifetime.

Invalid reads, incompatible calls, invalid conversions, arithmetic outside the applicable rules, and inaccessible destinations are outside the preconditions for defined completion. The supplied evidence does not specify a uniform fault, exception, error code, or recovery outcome for such cases.

## Side effects

### Direct effects

The represented body directly performs:

- An integer-width read from the beginning of the object.
- A callable-target read at the derived dispatch location.
- Evaluation of a fixed-global-derived argument.
- One indirect invocation when the invocation is reached.
- One one-byte assignment when the post-call continuation is reached.

These are source-level access effects. They do not specify a machine-instruction count, bus transaction width, atomicity, or device-memory behavior.

No direct allocation, deallocation, reference-count update, or ownership-transfer operation is visible. No direct assignment to the fixed global datum is shown.

### Dependency effects

F-opaque-01’s effects are not constrained by the absence of corresponding operations in this function. It may affect object storage, globals, external resources, or other reachable state.

In particular, the evidence supplies no preservation guarantee for:

- The object’s dispatch information.
- The destination byte.
- The fixed global datum.
- Storage reachable through the arguments.
- The lifetime of the original object.

The second argument has already been obtained for the current invocation when the dependency begins executing. A mutation of the global datum during that invocation does not cause this function to evaluate the argument again afterward.

That fact does not imply that storage potentially referenced by the argument is immutable.

### Ownership and lifetime

Passing the object reference establishes use of the reference, not acquisition or transfer of ownership.

The function itself shows no release operation, but this does not prove that the dependency leaves the object alive. Retention, release, replacement, and destruction behavior remain dependency-level unknowns.

The original destination must remain valid through the final write. This is a prerequisite for safe continuation, not a lifetime guarantee provided by this function.

## Dependencies

### F-opaque-01

| Aspect | Established contract |
|---|---|
| Identity | Selected at runtime; concrete implementation is unresolved. |
| Selection | Callable target obtained from the dispatch address derived through the object’s initial integer value. |
| Inputs | Original object reference, fixed-global-derived value, and integer zero, in that argument order. |
| Invocation count | One on an execution that reaches the invocation. |
| Result handling | No result is visibly inspected or used to decide whether to assign the byte. |
| Continuation | Normal return is necessary for this function to reach its assignment. |
| Additional requirement | The original destination must remain valid for defined completion. |
| Unknowns | Complete signature, calling convention, side effects, ownership rules, termination behavior, and error behavior. |

No other callable dependency is visible.

The fixed global datum is a separate data dependency. Its value contributes to the invocation, but its declaration, width, meaning, and initialization requirements are not supplied.

## Acceptance examples

These examples are conditional deductions from static evidence. They are not executed tests.

### Ordinary completion

Assume valid dispatch storage, compatible invocation rules, valid address calculations, and a dependency that returns normally without invalidating the destination.

The dependency receives the original object reference, the global-derived value, and zero. After the dependency returns, the supplied byte is written to the derived destination. This function then returns without an explicit result.

### Boundary byte representations

For supplied representations of `0x00` and `0xFF`, the same dispatch behavior applies. On normal continuation, the respective eight-bit representation is written unchanged. Neither representation selects a special path.

### Destination already contains the supplied value

Assume both the existing destination and the supplied byte contain `0x5A`.

The dependency is still invoked. The represented assignment still follows normal return; equality is not a condition for skipping either operation.

### Dependency writes the destination first

Assume the dependency writes `0x12` to the destination, leaves it valid, and returns normally. If the supplied byte is `0xA5`, this function subsequently writes `0xA5`.

Without subsequent or interfering writes, the resulting destination byte is `0xA5`.

### Dependency changes dispatch information

Assume the dependency replaces the integer value at the beginning of the object and returns normally.

This function does not select another target afterward. Its final destination remains derived from the original object reference, subject to valid conversion, arithmetic, and writable storage.

### Dependency changes the global datum

Assume the dependency changes the global datum during execution and returns normally.

This function does not fetch another global-derived argument after the call. The mutation does not introduce a condition that suppresses the byte write.

### Dependency returns an unfavorable result

If the actual selected operation has a result convention and returns a value regarded elsewhere as failure, normal return still leads to the assignment.

This example does not establish any particular dependency return type or failure convention.

### Dependency does not return normally

If the dependency never returns, or transfers control without normal return, this function’s later assignment is not reached through the visible continuation. Prior dependency effects are not rolled back by this function.

### Object or address validity is lost

If the dependency invalidates the destination, or the object-reference conversion loses necessary address information, ordinary completion cannot be required. The body contains no validation that repairs either condition.

## Uncertainties

The following remain **documented_only**:

- Integer width and the execution environment’s integer-arithmetic rules.
- Object-pointer and callable-pointer widths and representations.
- Pointer-to-integer conversion behavior, including representability.
- Alignment, endianness, and access requirements.
- The fixed global datum’s declaration, width, value, and meaning.
- Relative evaluation order between the global-derived argument and dispatch selection.
- The concrete target and its complete invocation requirements.
- Dependency mutations, ownership effects, failure modes, and termination.
- The object’s full size, layout, and invariants.
- The semantic meaning of the final byte.
- Concurrency, volatile-access, and hardware-memory requirements.
- Platform-specific consequences of invalid accesses or calls.

An operation-before-update interpretation is reasonable inferred intent. It does not establish successful notification, validation, resource management, or a particular state-machine transition.

## Evidence

The supplied static body establishes an indirect three-argument invocation followed by a one-byte assignment and return without an explicit value.

The dispatch selection uses the object’s initial integer value and a 20-byte displacement. The assignment uses an integer conversion of the original object reference and a 102-byte displacement.

The call receives the original object reference, a fixed-global-derived argument, and integer zero. The supplied byte participates in the later assignment rather than the explicit call arguments.

No conditional branch, result inspection, destination-value comparison, validation, recovery, allocation, or deallocation is visible. These observations describe this function’s direct behavior; they do not constrain the selected dependency.

No runtime tests have been performed. Acceptance examples are static deductions under their stated preconditions, and unresolved execution-model and dependency behavior remains documented rather than assumed.
