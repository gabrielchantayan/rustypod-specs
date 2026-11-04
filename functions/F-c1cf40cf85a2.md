<!-- Trusted provenance: F-c1cf40cf85a2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c1cf40cf85a2

## Purpose

F-c1cf40cf85a2 conditionally retrieves information associated with a context obtained from a dependency. It accepts two selectors and an output address. Successful dispatch either delegates an output operation to another dependency or copies one four-byte word directly from the context.

The function reports success with the return value 1 and reports an unavailable context or unsupported selector combination with 0. Unsupported combinations, unlike an unavailable context, also invoke an error-related dependency with the value 0x500.

These are dispatch-level results. A return value of 1 after a delegated operation does not establish that the dependency validated the destination, wrote a particular amount of data, or completed successfully according to any separate status convention.

The semantic meanings of the selectors, context fields, and error value are not established by the supplied evidence.

## Interface

The function has three inputs:

| Input | Representation | Role |
|---|---|---|
| Primary selector | Integer, interpreted here as a 32-bit value | Enables the retrieval operation only for 0x404 or 0x405. |
| Secondary selector | Integer used in equality and signed ordering comparisons | Selects a context-relative source, subject to a shared comparison value. |
| Output address | Address of storage that can receive a four-byte word; also passed to a dependency | Destination for the selected operation. |

The result is a four-byte value whose normal-return values are exclusively 0 and 1.

The supplied representation explicitly identifies four-byte data accesses and uses integer-valued addresses for context-relative arithmetic. The exact calling convention, native pointer width, register assignments, and original declared types are not supplied. The behavioral description uses a 32-bit integer model for the selectors and shared comparison value; a separate ABI specification would be needed to make their machine-level representation authoritative.

There is no explicit output capacity, context handle, error destination, or ownership flag among the inputs.

### Preconditions

No pointer validation is visible. Normal completion of an operation therefore requires the memory accesses and dependency calls selected by that operation to be valid.

For the direct-copy path, the context must permit reading four bytes beginning at offset 0x878, and the output address must permit writing four bytes.

For delegated paths, the context-relative address and output address must satisfy the requirements of F-01be51f7fed7. Those requirements are unknown. A four-byte pointer type for the output does not establish that this dependency writes only four bytes.

The function does not visibly require a valid output address on paths that make no explicit use of it. This does not establish that an arbitrary invalid address is safe under all dependency behavior, particularly if it aliases shared state.

## Behavior

### Context acquisition takes precedence

The first operation is a call to F-7fba0909c452 without explicit arguments.

If it returns zero, F-c1cf40cf85a2 returns 0 immediately. In this case:

- Neither selector is used to select a context field.
- The shared comparison value is not explicitly read.
- No explicit read or write through the output address occurs.
- Neither F-01be51f7fed7 nor F-375c8199e7c2 is called.

Effects performed by F-7fba0909c452 itself remain possible.

If the returned value is nonzero, the function retains it as the context base for subsequent operations. The evidence does not establish that this value represents a newly allocated object, a borrowed object, thread-local state, or a globally shared object.

### Primary-selector gate

With a nonzero context base, the primary selector enables retrieval only for 0x404 or 0x405.

The supplied representation derives the latter case through subtraction of 0x400 and comparison with 5. Under a 32-bit modular arithmetic interpretation, that condition identifies 0x405 exactly. The accepted values themselves require no overflowing subtraction.

For any other primary selector, the function:

1. Calls F-375c8199e7c2 with the retained context base and 0x500.
2. Returns 0 if that dependency returns normally.

It does not explicitly read the shared comparison value or access the selected output destination on this path.

The two accepted primary selectors have identical downstream behavior. There is no visible distinction in selected offsets, dependency arguments, or return values between them.

### Secondary-selector dispatch

After an accepted primary selector, the function uses a shared comparison value to select a branch. This is external state, not an input argument and not a fixed constant in the supplied evidence.

Equality has first priority. If the secondary selector equals the shared comparison value, the function invokes F-01be51f7fed7 with:

- The context base plus 0x858.
- The caller’s output address.

It then returns 1, without inspecting any return value from that dependency.

Only when the equality comparison fails does the function perform an ordering comparison against the shared comparison value. The supplied expression uses signed integer ordering. Unsigned ordering must not be substituted when the operands include negative or high-bit-set values.

The source expresses the equality test and ordering test as separate references to shared state. It does not explicitly retain a snapshot of that state. Whether these correspond to separate physical loads, and whether concurrent changes are permitted or defined, cannot be established from the supplied representation.

For a stable shared comparison value, dispatch is:

| Secondary-selector condition | Operation |
|---|---|
| Equal to the shared comparison value | Delegate from context offset 0x858. |
| Less than the shared comparison value, and equal to 0x1200 | Delegate from context offset 0x838. |
| Less than the shared comparison value, and equal to 0x1201 | Delegate from context offset 0x848. |
| Greater than the shared comparison value, and equal to 0x1600 | Delegate from context offset 0x868. |
| Greater than the shared comparison value, and equal to 0x1601 | Directly copy the four-byte word at context offset 0x878 to the output address. |
| Any remaining case | Invoke the error-related dependency and return 0. |

The literal secondary values are therefore not independently sufficient for success. Their relation to the shared comparison value matters, and equality overrides their usual literal-specific routes.

For example, when the shared comparison value is 0x1601, a secondary selector of 0x1601 selects the delegated operation at offset 0x858, not the direct copy at offset 0x878.

### Delegated success

Every delegated success calls F-01be51f7fed7 exactly once, with the selected context-relative address first and the unchanged output address second.

The selected offsets are:

- 0x838
- 0x848
- 0x858
- 0x868

These addresses are spaced 0x10 bytes apart. That spacing does not establish the size, alignment, type, or independence of the objects located there.

F-c1cf40cf85a2 does not inspect a status returned by F-01be51f7fed7. If the dependency returns normally, the enclosing function returns 1. If the dependency does not return normally, no normal return result is established.

### Direct-copy success

The direct-copy path reads one four-byte word beginning at context offset 0x878 and stores that word at the output address. The value read is the value subsequently stored. The function then returns 1.

There is no explicit call to F-01be51f7fed7 or F-375c8199e7c2 on this path.

The operation is a word transfer, not a specified conversion. No sign extension, byte swapping, string interpretation, or terminator processing is visible. Endianness and access-alignment requirements belong to the unspecified execution environment.

No non-aliasing requirement is visible. If the output address aliases the source word or another context location, the explicit behavior remains a source-word read followed by a destination-word write, provided those accesses are valid. This does not establish atomicity or synchronization.

### Rejection after context acquisition

All selector rejections with a nonzero context converge on the same visible behavior:

1. Invoke F-375c8199e7c2 with the context base and 0x500.
2. Return 0 after normal completion of that call.

The function does not visibly distinguish an unsupported primary selector from an unsupported secondary selector through different return values or different error arguments.

## Side effects

The following effects are established:

- F-7fba0909c452 is called once on every invocation.
- The shared comparison value is consulted only after obtaining a nonzero context and accepting the primary selector.
- Delegated success exposes a context-relative address and the output address to F-01be51f7fed7.
- Direct-copy success explicitly writes four bytes to the output address.
- Selector rejection with a nonzero context invokes F-375c8199e7c2.

No explicit allocation, release, reference-count adjustment, or context mutation is otherwise visible.

This absence does not establish ownership or lifetime guarantees. Any dependency may have additional effects, including shared-state changes, memory writes, allocation, or release. In particular:

- The context-acquisition dependency must not be assumed to preserve the shared comparison value.
- The delegated-operation dependency must not be assumed to preserve context contents or limit writes to the output.
- The error-related dependency must not be assumed to merely record an error, preserve memory, or release the context.

Consequently, rejection establishes the absence of an explicit output store by F-c1cf40cf85a2, not that output memory necessarily remains unchanged across the entire call.

No concurrency protection or transaction boundary is visible.

## Dependencies

| Opaque ID | Observed interface and use | Unknown contract |
|---|---|---|
| F-7fba0909c452 | Called without explicit arguments; zero causes immediate failure, nonzero supplies the context base. | Context validity, ownership, lifetime, acquisition failures, and side effects. |
| F-01be51f7fed7 | Called with a selected context-relative address and the caller’s output address; its result is not inspected. | Source layout, destination extent, transformations, failure signaling, ownership, and side effects. |
| F-375c8199e7c2 | Called with the context base and 0x500 for unsupported selector combinations. | Meaning of 0x500, error storage, resource handling, and whether normal return is guaranteed. |

The only directly established context field width is the four-byte word at offset 0x878. The delegated source addresses do not justify assigning a structure type or field size.

## Acceptance examples

These examples are conditional static acceptance cases, not runtime test results. Unless stated otherwise, dependencies return normally, the context base is nonzero, the shared comparison value is stable, and required memory accesses are valid.

1. **Unavailable context overrides selectors.**  
   If F-7fba0909c452 returns zero, the result is 0 even for primary selector 0x404 and an otherwise supported secondary selector. No error-related call follows.

2. **Unsupported primary selector.**  
   With primary selector 0x403, the function calls F-375c8199e7c2 with the context base and 0x500, then returns 0. No secondary dispatch occurs.

3. **Both accepted primary selectors behave alike.**  
   With a shared comparison value of 0x1400 and secondary selector 0x1200, either primary selector 0x404 or 0x405 delegates from offset 0x838 and returns 1.

4. **Lower-side alternative.**  
   With shared comparison value 0x1400 and secondary selector 0x1201, the delegated source is offset 0x848.

5. **Equality accepts a value outside the literal alternatives.**  
   With shared comparison value and secondary selector both 0x1300, the delegated source is offset 0x858. The absence of 0x1300 from the literal alternatives does not cause rejection.

6. **Upper-side delegated alternative.**  
   With shared comparison value 0x1400 and secondary selector 0x1600, the delegated source is offset 0x868.

7. **Upper-side direct transfer.**  
   With shared comparison value 0x1400, secondary selector 0x1601, and source word 0xA1B2C3D4 at offset 0x878, the output receives that word and the result is 1.

8. **Equality overrides the direct-transfer selector.**  
   With shared comparison value and secondary selector both 0x1601, the function delegates from offset 0x858. It does not explicitly copy the word at offset 0x878.

9. **A recognized literal on the wrong side is rejected.**  
   With shared comparison value 0x1000 and secondary selector 0x1200, the selector is greater than the comparison value but is neither 0x1600 nor 0x1601. The function invokes the error-related dependency and returns 0.

10. **Signed ordering matters.**  
    With shared comparison value −1 and secondary selector 0x1200, signed ordering places the secondary selector above the comparison value. It is rejected rather than dispatched to offset 0x838.

## Uncertainties

The following are **documented_only**:

- The semantic names and units of all selectors.
- The location, initialization, permitted values, and synchronization of the shared comparison value.
- The exact ABI, pointer width, and machine-level signedness guarantees.
- The original-language arithmetic rules for subtraction on rejected extreme primary-selector values. Under 32-bit modular arithmetic they introduce no additional accepted selector.
- Valid context extent, field alignment, byte order, and access atomicity.
- The amount and format of output produced by F-01be51f7fed7.
- Dependency behavior on invalid addresses, failures, or nonlocal exits.
- Memory ownership and lifetime before and after dependency calls.
- Whether repeated calls observe stable context or shared state.

Interpreting the four delegated locations as a family of related records is plausible intent, but not an established layout contract.

## Evidence

Established behavior comes solely from the supplied static function body and dependency-ID mapping: call order, branch precedence, literal selectors, offsets, explicit four-byte transfer, error arguments, and normal return values.

Dependency implementations and runtime observations were not supplied. No runtime tests have been performed. Acceptance examples describe consequences of the visible control flow under their stated conditions; they are not observed executions.
