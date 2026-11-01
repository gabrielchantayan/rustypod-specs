<!-- Trusted provenance: F-1bc5bcdc9cca; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1bc5bcdc9cca

## Purpose

F-1bc5bcdc9cca returns a 16-bit value from a location selected relative to an address-bearing integer. The selection uses a signed integer, a 32-byte spacing, and a fixed 28-byte displacement.

Its admission rule is asymmetric: selectors greater than two invoke F-a817d8d58c10, which is annotated as non-returning. Every selector at or below two, including negative values, takes the direct-read path. Admission does not establish that the resulting memory access is valid.

The spacing is consistent with accessing a field in fixed-size records. That is **inferred intent**, not an established data model. Neither the number of records nor the meaning of the returned field is supplied.

## Interface

### Inputs

The function receives two signed integer inputs:

| Input | Behavioral role |
|---|---|
| Base | An address-bearing integer used to locate the selected two-byte field. |
| Selector | A signed integer controlling both admission and displacement. |

The selector is compared as signed. Negative values therefore pass the upper-bound check; they are not treated as large unsigned selectors.

The supplied representation identifies the inputs as signed integers but does not independently specify their bit width, the address width, or the calling convention. These properties remain **documented_only**. In particular, the base must not be described as a portable pointer parameter or an ownership-bearing reference.

### Output

Successful direct completion returns the 16-bit value read from the selected location.

No content-dependent interpretation or transformation is established. All 16-bit patterns are eligible results when obtained through a valid read. Result signedness, semantic meaning, and handling of any surrounding return-register bits remain **documented_only**.

The result is a value, not a returned address, newly allocated object, or ownership token.

For a selector greater than two, the established behavior is an invocation of the annotated non-returning dependency. No ordinary returned value is established for that path.

### Conditions for successful direct completion

A successful read requires all of the following:

- The relevant integer arithmetic has a defined outcome in the execution environment.
- Conversion of the computed integer to a memory address is meaningful in that environment.
- Both bytes at the selected location are readable.
- Any alignment requirement is satisfied.
- The storage remains available when accessed.

These are caller- or environment-dependent conditions, not validations performed by this function. No explicit check establishes a valid allocation, sufficient extent, permissions, alignment, or lifetime.

## Behavior

### Selection contract

For admitted selectors and ordinary representable arithmetic, the selected field occupies the following locations:

| Selector | Field start relative to the base | Two-byte extent |
|---:|---:|---|
| −2 | 36 bytes before | Displacements −36 and −35 |
| −1 | 4 bytes before | Displacements −4 and −3 |
| 0 | 28 bytes after | Displacements 28 and 29 |
| 1 | 60 bytes after | Displacements 60 and 61 |
| 2 | 92 bytes after | Displacements 92 and 93 |

Increasing the selector by one advances the field start by 32 bytes, where both calculations remain valid.

This table describes selected fields, not entire records or allocations. The function does not require an intervening region to be read merely because the selected field lies beyond it. It also does not inspect neighboring fields.

A negative selector can select storage before the base. Such a selector is not rejected, clamped to zero, or replaced with another selector.

### Rejected selections

Any signed selector greater than two invokes F-a817d8d58c10 without explicit arguments. This invocation precedes any selected-field read by F-1bc5bcdc9cca on that path.

The dependency carries a non-returning annotation. Ordinary completion of the rejected path therefore has no established return value. The annotation does not identify whether control ends through termination, a trap, a nonlocal transfer, indefinite execution, or another mechanism.

There is no separate local error return or fallback field for rejected selections. A bad base does not introduce a base-validation step before the dependency invocation.

### Arithmetic constraints

The field displacement uses a selector-scaled contribution of 32 bytes per unit and a fixed contribution of 28 bytes. The supplied expression places the scaled contribution into an addition with the base before the fixed contribution is added.

This intermediate structure matters. A final mathematical address within range is not, by itself, enough to establish valid evaluation: the scaled contribution and the intermediate addition must also be representable under the applicable arithmetic rules.

The evidence shows no explicit overflow detection, saturation, recovery, or conversion to a wider integer type before this arithmetic. It does not establish wrapping arithmetic.

The selector upper bound does not make all admitted arithmetic safe:

- A sufficiently negative selector can exceed the range available for its scaled contribution.
- An admitted selector can produce an out-of-range intermediate sum with the base.
- Even selector zero can encounter a limit when the fixed displacement is added.
- A representable integer result can still fail to identify accessible memory.

If the execution environment uses 32-bit signed arithmetic, the scale contribution for selector −67,108,864 is exactly the minimum 32-bit signed value; the contribution for selector −67,108,865 is below that range. Both selectors pass the admission comparison. This is a **conditional width example**, not evidence that the inputs are necessarily 32 bits wide.

Under the same 32-bit condition, selector −1 combined with a base equal to the minimum signed value plus four would have a final mathematical displacement at the minimum value. Nevertheless, the intermediate addition involving the negative scaled contribution would already be out of range. Final-address reasoning alone would therefore miss an arithmetic failure.

Exact overflow consequences and integer-to-address interpretation remain **documented_only**. No selected location or result is promised when the required arithmetic lacks a defined interpretation.

### Memory access and returned value

The direct-read path contains one explicit 16-bit load. That load supplies the returned value without an established content-based rejection or substitution.

The represented load does not establish:

- Two separately ordered byte reads.
- A single physical memory transaction.
- Atomicity.
- Support for unaligned addresses.
- A particular byte order.
- A stable value under concurrent modification.

Byte order affects the numerical interpretation of the selected bytes. It does not change which two byte locations are selected.

### Failures outside the selector check

No local recovery behavior is established for inaccessible memory, invalid address conversion, unsupported alignment, or arithmetic failure. In particular, the function provides no established sentinel value, diagnostic, or recoverable status for those conditions.

A zero base is not explicitly rejected. Its consequences depend on the computed address and the execution environment.

The admission decision must therefore not be presented as a memory-safety check. Passing it establishes only that the selector did not exceed two.

## Side effects

On the direct-read path, the selected 16-bit load is the only explicit data-memory operation shown. No explicit memory write, allocation, release, reference-count change, or persistent-state update is established.

This is not a guarantee of observational purity. A read-sensitive memory region may produce external effects when accessed. The evidence does not restrict the selected location to ordinary application storage.

No synchronization, lifetime protection, or concurrent-access protocol is shown. The function does not establish an atomic snapshot or prevent another actor from changing or releasing the selected storage.

The base conveys no established ownership transfer. The direct path neither explicitly acquires ownership nor explicitly relinquishes it.

The observable local ordering is limited to these facts:

- The signed admission decision precedes the selected-field access.
- A rejected selection invokes the dependency without a preceding selected-field load by this function.
- Successful direct completion returns the value supplied by the selected load.

These facts do not constrain the dependency’s own reads, writes, external effects, or resource handling.

## Dependencies

### F-a817d8d58c10

| Property | Established contract |
|---|---|
| Invocation condition | The signed selector is greater than two. |
| Explicit arguments | None. |
| Control-flow annotation | Non-returning. |
| Ordinary result used by this function | None established. |

An invocation without explicit arguments does not establish an absence of inputs. The dependency may observe globals, thread-local state, external resources, or other implicit context.

Its implementation is unavailable. The following remain **documented_only**:

- Memory reads and writes.
- Changes to global or external state.
- Allocation, release, or other resource-management behavior.
- Diagnostic output.
- Exceptions, termination status, or other control-transfer mechanisms.
- Synchronization effects.
- Whether the non-returning annotation holds in every execution.

The dependency must not be assumed to preserve globals or to retain or release any particular memory.

The supplied representation places the selected read after the conditional invocation, but also marks that invocation as non-returning. If the dependency unexpectedly returns, a supported execution contract cannot be established from those facts alone. No reliable returned value or recovery behavior is promised for that circumstance.

No other callable dependency is identified in the supplied body.

## Acceptance examples

These are conditional criteria derived from static evidence. No runtime tests have been performed.

### Distinct admitted fields

Suppose the fields beginning 28, 60, and 92 bytes after the base are readable and contain distinct 16-bit values. Assume valid arithmetic and stable storage for each access.

Selectors zero, one, and two must return the respective values from those locations without invoking F-a817d8d58c10. Reading adjacent two-byte fields instead would violate the 32-byte selection spacing.

### Negative selection

Suppose the two bytes beginning four bytes before the base are readable and the required arithmetic is valid.

Selector −1 must return that field’s value without invoking the dependency. Rejecting it solely because it is negative would change the established behavior.

If those bytes are inaccessible instead, the selector still passes admission. Successful completion is not required, and no local sentinel result is established.

### First rejected selection

Selector three must invoke F-a817d8d58c10 before any selected-field read by this function.

Under the non-returning annotation, no ordinary result follows. Acceptance does not require a particular message, exception, or exit status.

### Rejection independent of base validity

For a selector greater than two, an unsuitable base does not cause this function to load a selected field before invoking the dependency.

This does not require harmless execution: the dependency’s accesses and effects remain unknown.

### Unrestricted field content

Given a valid selected read, a field containing all zero bits or all one bits must be returned without a content-based fallback. No reserved-value rule is established.

### Byte order

Suppose the selected bytes, in increasing address order, are hexadecimal 12 and 34.

A little-endian interpretation yields hexadecimal 3412; a big-endian interpretation yields hexadecimal 1234. The evidence does not select between these target properties.

### Arithmetic boundary

An extremely negative selector remains admitted by the signed upper-bound comparison. If its scaled contribution or a subsequent addition cannot be evaluated under the applicable arithmetic rules, admission does not require a successful read, a wrapped address, or a returned error value.

## Uncertainties

The following remain **documented_only**:

- Exact signed input widths, address width, and calling convention.
- Arithmetic overflow semantics and integer-to-address conversion.
- Byte order and alignment requirements.
- Result signedness, field meaning, and unused return-register bits.
- Storage layout beyond the selected two-byte fields.
- Allocation extent, accessibility, lifetime, and ownership.
- Atomicity and concurrent-access behavior.
- Effects of reading read-sensitive storage.
- Dependency state changes, resource behavior, and non-return mechanism.
- Supported behavior following an unexpected dependency return.

The fixed-record interpretation remains **inferred intent**. It does not justify assuming a three-element allocation, adding a lower-bound rejection, or requiring complete surrounding records to be accessible.

## Evidence

The established static evidence supports:

- Two signed integer inputs.
- A signed upper-bound comparison against two.
- Conditional invocation of F-a817d8d58c10 with no explicit arguments.
- A non-returning annotation for that dependency.
- Selection at 32-byte intervals, with selector zero identifying a field 28 bytes after the base.
- A 16-bit memory load supplying the direct result.
- No explicit lower-bound check, selected-memory write, allocation, release, content validation, overflow recovery, or local error result.

The storage model and field meaning are not established by the spacing alone. Unavailable target and dependency properties have not been filled in with assumptions.

No runtime tests, target inspection, or dependency-body analysis have been performed.
