<!-- Trusted provenance: F-37c308adca3e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-37c308adca3e

## Purpose

F-37c308adca3e conditionally replaces an integer property in an object obtained from F-7fba0909c452. The supported selector pair is **(1,032, 5,633)**. For that pair, proposals from **0 through 8,388,608, inclusive**, are accepted.

Rejected requests leave the property untouched by this function’s property access. They may record an error in a separate field, but an observed nonzero error is retained.

“Property” and “error” describe storage roles, not established application meanings. Interpreting this behavior as a bounded property setter with existing-error retention is **inferred intent**. The property’s units and the meanings of the error values are unknown.

This contract is based exclusively on static evidence. **No runtime tests have been performed.**

## Interface

### Inputs and result

The function receives three integer arguments:

| Input | Role |
|---|---|
| First selector | Identifies the supported family when equal to 1,032. |
| Second selector | Identifies the supported operation when equal to 5,633, provided the first selector is supported. |
| Proposed value | Supplies the replacement property value on acceptance. |

The interface exposes no return value or separate status result. Normal completion alone does not distinguish acceptance, rejection, preservation of an existing error, suppression of error recording, or absence of an object.

The target object is not supplied by the caller. Its address comes from F-7fba0909c452.

### Numerical interpretation

The supplied representation uses signed integer arguments and signed integer storage. It does not independently establish the external calling convention or integer width.

For the concrete widths, byte offsets, and extreme-value examples in this contract, the interpretation is **32-bit signed integers occupying four bytes**, with a range of **−2,147,483,648 through 2,147,483,647**. Those layout details are conditional on that interpretation.

The accepted proposal interval is established directly by comparisons. Proposal classification does not require overflowing arithmetic, including at either signed 32-bit extreme.

Selector classification has a different limitation: sufficiently negative selectors require an unrepresentable signed intermediate under the 32-bit interpretation. Those domains are explicitly excluded from unconditional outcome requirements below and documented under **Uncertainties**.

### Relevant storage

Under the four-byte integer interpretation:

| Storage role | Location | Access width |
|---|---|---:|
| Error field | Object base address | 32 bits |
| Property field | 2,168 bytes after the object base | 32 bits |
| Shared error source | Separate named storage location; relationship to the object unknown | 32 bits |

The error and property fields occupy distinct ranges within the object. Nothing establishes the purpose or required accessibility of the intervening bytes.

A null object address is supported. A non-null address receives no demonstrated validity check. Required accessibility depends on the selected path:

- Acceptance requires a writable property field, but no direct read of its previous value or of the error field.
- Rejection with a nonzero selected error requires a readable error field and, if zero is observed, a writable error field.
- Rejection with a zero selected error requires no direct object-field access.
- A null object result requires no subsequent direct access to either identified field.

Every supported request with a non-null object also requires access to the shared error source, including requests that will be accepted.

“Direct” throughout this contract means an access attributable to F-37c308adca3e itself. It excludes dependency-internal activity and distinguishes storage roles where aliasing is unresolved.

## Behavior

### Outcome contract

The following outcomes assume normal dependency completion, valid storage for the accesses actually made, and defined selector classification.

| Object and request | Shared error source | Property field | Error field |
|---|---|---|---|
| Null object | No subsequent direct access | No direct access | No direct access |
| Non-null object; supported selectors; proposal in range | Read | Replaced with proposal | No direct access |
| Non-null object; supported selectors; proposal outside range | Read | No direct access | Conditional error recording |
| Non-null object; unsupported selectors | Not read directly | No direct access | Conditional recording of 1,280 |

A null object ends this function’s work before selector classification. Consequently, the null-object outcome also applies to selector values whose non-null behavior is unresolved because of signed arithmetic.

Dependency side effects are not constrained by any “no direct access” entry in this table.

### Accepted requests

Acceptance requires both the supported selector pair and:

**0 ≤ proposed value ≤ 8,388,608.**

The accepted proposal replaces the entire property field. No direct comparison with the previous property value is required or represented. Supplying the same value already stored therefore does not suppress the represented replacement.

Acceptance is independent of both error sources:

- An existing nonzero object error does not block replacement.
- The object error is not directly inspected, cleared, or replaced.
- A zero, positive, or negative shared error value does not change the accepted replacement.

Nevertheless, the shared error source is read on the accepted path, before the property write. Its irrelevance to the replacement value does not make that represented read optional.

The behavior is not saturation. A rejected negative proposal does not produce a property replacement of zero, and a rejected proposal above the upper bound does not produce a replacement of 8,388,608.

### Rejected requests

There are two sources of rejection errors:

| Rejection category | Selected error |
|---|---|
| Unsupported selector pair | 1,280 |
| Supported pair with an out-of-range proposal | Value read from the shared error source |

For defined selector classification, an unsupported first selector makes the second selector and proposal irrelevant to the selected error. With a supported first selector, an unsupported second selector makes the proposal irrelevant.

Neither unsupported-selector category directly reads the shared error source or accesses the property field.

Error recording depends only on whether the selected error and the observed object error are zero:

| Selected error | Object error observation | Direct result |
|---|---|---|
| Zero | No observation | No error-field write |
| Nonzero | Zero | Replace the error field with the selected error |
| Nonzero | Nonzero | Preserve the observed error without a replacement |

Negative values count as nonzero in both roles. A negative selected error can be recorded, and a negative existing error prevents replacement just as a positive existing error does. No priority or severity ordering among nonzero errors is established.

For a supported request with a rejected proposal and a zero shared value, the only direct storage access after obtaining the object is the shared-source read. Neither identified object field is directly accessed.

### Completion and exceptional conditions

Normal completion supplies no explicit diagnostic result.

Invalid non-null addresses, inaccessible storage, inadequate alignment, abnormal dependency completion, and access faults have no established recovery behavior. The error field is not established as a mechanism for reporting failures encountered while accessing the object itself.

Completion and state after such conditions remain **documented_only**.

## Side effects

### Observable access ordering

The established ordering constraints are:

- The dependency call precedes this function’s direct shared-source and object-field accesses.
- For supported requests, the shared-source read precedes the property replacement or any rejection-related error-field access.
- Any error-field replacement follows a read of that field that observed zero.

Under normal conditions, the represented function performs at most one shared-source read, at most one error-field read, and at most one identified object-field write. Acceptance writes the property; rejection may write the error field. The function does not directly write both fields during one invocation.

These are constraints on the supplied representation. They do not establish volatile semantics, physical memory-transaction counts, indivisible accesses, memory barriers, or cross-thread visibility.

### State timing and preservation

Values described as “existing” refer to the state available at the relevant access after F-7fba0909c452 returns. They are not necessarily invocation-entry values.

The dependency is not known to preserve the object, shared source, or other global state. In particular:

- A supported request uses the shared value available after the dependency call.
- Error retention concerns the value observed by the later error-field read.
- This function’s lack of a direct write does not establish that the dependency left the corresponding storage unchanged.

### Aliasing

The shared source’s disjointness from the returned object is unknown. The function has no direct write addressed as an update to that source, but this is not an unconditional preservation guarantee for its bytes.

If storage overlaps, a property or error-field replacement may also change shared-source storage. Conversely, a shared-source read may access bytes belonging to an object field.

Statements such as “no direct error-field access” identify the role of the represented access. They do not rule out an access to overlapping bytes through another role.

### Ownership and concurrency

No allocation, release, retained reference, or ownership transfer is explicit in F-37c308adca3e. No equivalent restriction may be inferred for F-7fba0909c452.

The object address must remain valid through every actual access. The evidence establishes neither ownership of that address nor an obligation or permission to release it.

Existing-error retention is not an established atomic first-error operation. The zero observation and subsequent replacement are separate represented accesses. Concurrent mutation can therefore invalidate a broader claim that an earlier error is always preserved. Synchronization requirements and race behavior remain **documented_only**.

## Dependencies

### F-7fba0909c452

| Aspect | Established interaction |
|---|---|
| Invocation | One call, before request classification |
| Explicit inputs | None |
| Result used | Object address |
| Null result | Prevents subsequent direct shared-source and identified object-field accesses |
| Non-null result | Provides the base address for the two identified fields |

The call occurs even when the supplied selectors or proposal would otherwise cause rejection.

Its object-selection policy, internal reads and writes, allocation or release behavior, ownership requirements, lifetime guarantees, synchronization, and abnormal completion are **documented_only**. No global-state preservation is assumed.

No other function dependency is shown. The shared error source is a storage dependency, not another function invocation.

## Acceptance examples

These examples are conditional specifications derived from static evidence, not executed tests.

Unless an example states otherwise, assume a normally returning dependency supplies a non-null object; signed 32-bit, four-byte integers; valid and non-overlapping storage; no concurrent mutation; normally completing accesses; and defined selector arithmetic. Initial field and shared values mean their post-dependency values.

| Conditions | Required behavior |
|---|---|
| Supported selectors; proposal 0 | Read the shared source, then replace the property with 0. No direct error-field access. |
| Supported selectors; proposal 8,388,608 | Read the shared source, then replace the property with 8,388,608. |
| Supported selectors; proposal 100; existing error 41; shared value −7 | Read the shared source and replace the property with 100. No direct inspection or replacement of error 41. |
| Supported selectors; proposal equals the existing in-range property | Perform the represented property replacement without a direct read to compare the old value. |
| Supported selectors; proposal −1; shared value 23; existing error zero | Leave the property unaccessed; read the zero error and replace it with 23. |
| Supported selectors; proposal 8,388,609; shared value 23; existing error zero | Record 23. Do not replace the property with a boundary value. |
| Supported selectors; rejected proposal; shared value zero | Read the shared source; perform no direct access to either object field. |
| Supported selectors; rejected proposal; shared value −7; existing error zero | Read the zero error and replace it with −7. |
| Supported selectors; rejected proposal; shared value 23; existing error −9 | Read and retain −9 without an error-field write or property access. |
| First selector 1,033; existing error zero | Record 1,280 without directly reading the shared source, regardless of the other supplied argument values. |
| Selectors (1,032, 5,632); existing error 41 | Read and retain 41. No direct property or shared-source access. |
| Dependency changes the shared source to 29; supported selectors; rejected proposal; existing error zero | Read the post-call value and record 29. |
| Dependency returns null; any arguments | No subsequent direct shared-source or identified object-field access. Dependency effects remain unconstrained. |

With supported selectors, proposals **−2,147,483,648** and **2,147,483,647** are both rejected without an overflowing proposal-classification intermediate.

With a non-null object, a first selector of **−2,147,483,648** does not have an established ordinary unsupported-selector outcome under the signed-arithmetic interpretation. With a null object, that same input still requires no subsequent direct accesses.

## Uncertainties

### Inferred intent

The behavior is consistent with setting a bounded property while retaining an already recorded error. This does not establish an application-level property name, units, error taxonomy, or the semantic significance of 1,280.

### documented_only: selector arithmetic

Under signed 32-bit arithmetic, the following are the lowest selector values that avoid the identified unrepresentable classification intermediate:

| Applicable input | Lowest value avoiding that intermediate |
|---|---:|
| First selector, when the object is non-null | −2,147,482,616 |
| Second selector, when the first selector is 1,032 | −2,147,478,016 |

Values below the applicable threshold require an intermediate below the signed 32-bit minimum in the supplied representation. The evidence does not establish wrapping, trapping, or other consequences for those inputs.

For the affected non-null paths, classification, completion, subsequent accesses, and final state remain **documented_only**. Ordinary unsupported-selector behavior must not be extrapolated into those ranges.

The second-selector limitation is irrelevant when a defined first-selector classification already establishes rejection. Neither limitation is reached after a null object result.

### documented_only: environment and storage

Unresolved matters include:

- Confirmation of integer width and the external calling convention.
- Pointer width, byte order, and exact alignment requirements.
- Shared-source initialization, placement, lifetime, and possible thread-local status.
- Disjointness of shared storage and object storage.
- Volatility, atomicity, synchronization, and concurrent-access behavior.
- Dependency memory-management obligations and returned-object lifetime.
- Fault handling and abnormal dependency completion.

These unknowns do not justify assumptions of dependency purity, global-state preservation, or ownership transfer.

## Evidence

The supplied static representation supports the selector pair, inclusive proposal bounds, dependency interaction, null-object behavior, conditional error recording, and access-order constraints described above.

It also establishes that accepted requests read the shared source, do not consult the existing property or error, and perform the property replacement. Rejected requests do not perform a property replacement or saturation. Unsupported selectors select 1,280 without a direct shared-source read. A zero selected error suppresses direct error-field access; a nonzero selected error is recorded only following a zero error observation.

The byte offsets and extreme-value qualifications depend on the stated four-byte signed-integer interpretation. Application semantics, dependency internals, ownership, synchronization, and abnormal execution are not established.

**No runtime tests have been performed.** The acceptance examples specify conditional behavior; they do not report observed execution.
