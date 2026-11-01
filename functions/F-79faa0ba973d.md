<!-- Trusted provenance: F-79faa0ba973d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-79faa0ba973d

## Purpose

F-79faa0ba973d produces the exact mathematical magnitude of a signed-integer argument when that magnitude is representable in the result type.

**Established static evidence:** The interface accepts one signed integer by value and returns an integer of the same declared type. The supplied definition identifies no external-storage access, resource operation, or callable dependency.

**Inferred intent:** An absolute-value utility.

The distinction between a representable argument and a representable magnitude is essential. A signed type may contain a negative value whose positive magnitude lies outside its range. This function supplies neither a wider result type nor an overflow-recovery contract.

## Interface

### Values and types

| Element | Contract |
|---|---|
| Argument | One signed-integer value |
| Result | One signed-integer value of the same declared type |
| Additional outputs | None |
| Error channel | None |
| Storage references | None |
| Ownership-bearing values | None |

The argument is passed as a value, not as access to its originating storage. The function therefore has no interface-level means to update the caller’s original object.

A compatible invocation and valid delivery of the argument are preconditions. Argument acquisition, caller-side conversions, and storage of the returned result are outside this function’s boundary.

### Width and supported domain

The supplied type declaration does not independently establish a concrete bit width. In particular, the evidence does not justify a fixed-width numerical range merely from the declaration or the function’s identifier.

Let \(P\) denote the greatest positive value representable by the argument and result type. For representable arguments, the supported numerical domain is:

\[
D=\{x:\lvert x\rvert\leq P\}.
\]

The magnitude and inequality in this definition are mathematical expressions, not additional arithmetic that the implementation is required to execute.

For every argument in \(D\), the result is representable and exact. An argument outside \(D\) has no defined numerical guarantee under ordinary signed-arithmetic semantics.

This precondition does not mean the function checks membership in \(D\). No explicit overflow validation or error report is established.

The smallest representable value requires target-specific consideration. If the range is asymmetric and extends below \(-P\), that additional negative endpoint is unsupported. If the range is symmetric, the negative endpoint can be supported. “Smallest representable input” alone is therefore insufficient to determine the outcome.

A wider caller-side destination does not enlarge \(D\). Receiving the result in a wider type cannot repair arithmetic that was already outside the function’s defined domain.

### Layout and ownership

No record, array, buffer, length, capacity, pointer, or handle is part of the interface. There are no field offsets, element strides, or serialized layouts to specify.

The exact storage width, padding, alignment, byte order, and physical argument and result locations are **documented_only**. No particular returned bit pattern is established beyond the requirements of the applicable type and calling convention.

There is no ownership transfer, retained reference, allocation obligation, or release obligation.

## Behavior

### Defined numerical result

For a supported invocation, the result is the mathematical absolute value of the argument:

\[
r=\lvert x\rvert.
\]

This is a value-level contract, not a prescription for control flow or machine instructions.

Its consequences include:

- The result is nonnegative.
- A zero argument produces zero.
- A nonzero supported argument produces a positive result.
- Supported arguments with equal magnitudes produce equal results.
- A nonnegative argument produces an equal numerical value.
- Reapplying the operation to a supported result leaves that numerical value unchanged.
- No rounding, approximation, or saturation occurs within the supported domain.

These properties concern numerical equality. They do not require identical execution time, processor activity, or object representations for different invocations.

Where a target permits multiple representations of the same numerical value, the evidence does not establish a representation-normalization policy. Numerical zero, for example, is covered by the value contract without establishing a separate bit-pattern guarantee.

### Arithmetic boundary

An unrepresentable positive magnitude cannot be returned exactly in the declared result type. Under ordinary signed-arithmetic semantics, the supplied operation does not define an arithmetic outcome for that case.

For such an invocation, this contract does not guarantee:

- A returned value.
- Normal completion.
- A wrapped negative result.
- Saturation at \(P\).
- A diagnostic, exception, or trap.
- Preservation of surrounding program state.

These possibilities are not interchangeable accepted responses. There is no established response policy for unsupported arithmetic.

An implementation-specific arithmetic extension could supply additional guarantees. Its existence and effect are **documented_only** unless independently established. The function’s apparent purpose is not evidence of such an extension.

### Completion and errors

For supported invocations, the required outcome is the exact result. No explicit failure path, status value, diagnostic output, or error sentinel is present.

Zero is a successful result for zero. The greatest positive value is a successful result for either supported argument having that magnitude. Neither value is reserved for failure.

No timeout, cancellation, asynchronous completion, or recovery protocol is established. Execution time and generated runtime mechanisms remain outside the available evidence.

## Side effects

### Observable accesses and ordering

The supplied definition identifies no explicit access to caller-owned storage, global storage, files, devices, or shared resources.

At the function boundary, the argument value is available before the returned value is delivered. There are no identified external reads or writes between those events that require an additional ordering contract.

The function does not acquire access to the object from which the caller obtained the argument. For a defined invocation, and absent an independent modification, that object retains its previous value. Assigning the returned result back to that object would be a separate caller action.

This does not validate the caller’s argument acquisition. Invalid storage access, a data race, or an invalid conversion before invocation remains outside this contract.

### State, resources, and concurrency

No persistent application-state transition, allocation, deallocation, resource acquisition, callback, input/output operation, or synchronization operation is established.

There are no explicit callees whose effects must be assumed. Consequently, the side-effect description does not depend on a supposition that an unknown helper preserves globals or manages memory safely.

The evidence establishes no synchronization relationship between surrounding caller actions. A pure numerical result is not a memory-ordering guarantee.

Physical stack activity, generated instrumentation, runtime support, processor-status effects, and timing are **documented_only**. The absence of explicit external accesses must not be extended into a claim about every machine-level effect.

The side-effect guarantees apply to defined invocations. They are not state-preservation guarantees for unsupported arithmetic.

## Dependencies

**Explicit callable dependencies: none.**

No dependency function ID, helper return value, callback result, or downstream ownership rule is needed to express the established behavior.

The relevant environmental requirements are:

| Requirement | Significance |
|---|---|
| Compatible invocation | Enables valid argument and result exchange |
| Actual signed-integer range | Determines representable arguments and magnitudes |
| Applicable arithmetic semantics | Determines the treatment of unsupported arithmetic |
| Applicable implementation extensions, if any | May add guarantees not established by the definition |

Potential compiler-generated support is not established as a dependency by the supplied evidence. Its presence, identity, and effects must not be invented.

## Acceptance examples

The following are conditional acceptance criteria derived from static evidence. **No runtime tests have been performed.**

### Ordinary values

Assume a compatible invocation and a type capable of representing all listed values.

| Argument | Required result |
|---:|---:|
| 0 | 0 |
| 1 | 1 |
| −1 | 1 |
| 23 | 23 |
| −23 | 23 |
| 1,024 | 1,024 |
| −1,024 | 1,024 |

Returning a negative value, approximation, or failure sentinel for these cases would violate the contract.

The two arguments of magnitude 23 must produce equal numerical results. Supplying either result to another invocation must again produce 23. These criteria impose no timing-equivalence requirement.

### Caller storage

Assume a caller-owned object contains −23 and its value is validly supplied as the argument. Assume no independent operation changes that object.

The function must return 23. The original object remains −23. A later caller-side assignment may change it, but that assignment is not an effect of this function.

### Supported extrema

Assume independent target evidence establishes the greatest positive value \(P\).

Both \(P\) and the representable value \(-P\) must produce \(P\). Rejecting either supported argument merely because it is at a boundary would violate the numerical contract.

### Asymmetric range

Assume the actual range is −32,768 through 32,767, ordinary signed-arithmetic semantics apply, and no relevant extension is established.

| Argument | Acceptance consequence |
|---:|---|
| 32,767 | Must return 32,767 |
| −32,767 | Must return 32,767 |
| −32,768 | No defined result or completion guarantee |

The unsupported argument has mathematical magnitude 32,768, which the assumed result type cannot represent. Choosing a wider destination does not change this consequence.

Neither a trap nor a wrapped value is a required outcome for the unsupported case.

### Symmetric range

Assume instead that the actual range is −32,767 through 32,767 and the required arithmetic is defined throughout that range.

Both endpoints must produce 32,767. The negative endpoint is supported despite being the smallest representable value.

These range examples are conditional specifications, not claims about the actual target.

## Uncertainties

The following remain **documented_only**:

- Concrete integer width, storage size, numerical limits, and representation.
- Padding, alignment, byte order, and physical calling convention.
- Representation choices for numerically equal values.
- Any applicable arithmetic extensions.
- Actual consequences of an unsupported-magnitude invocation.
- Consequences of incompatible invocation or invalid caller-side argument acquisition.
- Generated instrumentation and runtime-support behavior.
- Timing, physical temporary storage, and processor-state effects.

The absolute-value role is **inferred intent**. The same-type value interface, exact supported-domain magnitude, and absence of explicit external accesses or calls are **established static evidence**.

These uncertainties do not authorize adding validation, widening, diagnostics, resource effects, or an invented overflow policy.

## Evidence

The supplied definition is the direct static evidence for this contract. It establishes the signed-integer value interface and the exact magnitude result wherever the required arithmetic is defined. It also provides the basis for identifying no explicit callable dependencies or external-storage accesses.

The numerical invariants, supported-domain boundary, and acceptance examples are derived consequences, not execution observations. Concrete target properties cannot be established from the declaration alone.

The prior draft and reviewer feedback are not independent behavioral evidence. No compilation, runtime testing, target inspection, or generated-instruction inspection has been performed.
