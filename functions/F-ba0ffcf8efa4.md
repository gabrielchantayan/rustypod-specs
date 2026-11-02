<!-- Trusted provenance: F-ba0ffcf8efa4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ba0ffcf8efa4

## Purpose

F-ba0ffcf8efa4 conditionally routes an input record through one of three processing paths. An indirect operation on a receiver first determines whether processing proceeds. If that operation returns zero, the function returns a zero-valued 32-bit result without inspecting the record’s selector.

When processing is enabled, a selector at byte offset 4 within the record chooses between two conversion-and-processing paths and a fallback path:

- Selectors 7 and 13 use conversion parameter 2560, then invoke F-9a45315b9740.
- Selector 21 uses conversion parameter 13568, then invokes F-e9270c70c772.
- Every other selector invokes the fallback dependency directly.

The function’s result is either zero from the disabled path or the result supplied by the selected processing dependency.

This is a structural behavioral contract. The supplied evidence does not establish the domain meaning of the receiver, record, selectors, conversion parameters, or returned values.

## Interface

### Inputs

The function has two explicit inputs:

1. **Receiver reference.** An address used both to locate an indirect operation and as the receiver argument to subsequent processing.
2. **Record address or address-bearing value.** A value used to access a selector at byte offset 4 and passed unchanged to either the conversion dependency or the fallback dependency.

The supplied recovery represents the receiver as a pointer to an integer-sized field and the record input as an integer used in address arithmetic. These recovered types do not establish a complete object model, a portable declaration, or the native pointer width.

### Receiver layout required by the visible access

The receiver’s initial field supplies the base address of a dispatch structure. At byte offset 352, hexadecimal 0x160, within that structure is a callable entry.

That entry is invoked with the receiver as its sole visible explicit argument. This contract calls that otherwise unidentified dependency **U-01**.

The visible behavior requires:

- A readable receiver initial field.
- A readable callable entry at dispatch-base offset 352.
- A valid call target with a calling convention compatible with the invocation.

No other receiver fields are directly accessed by this function.

### Record layout required by the visible access

If U-01 returns nonzero, the function reads one integer selector at record byte offset 4. The supplied recovery presents this as an integer-width read, treated here as a 32-bit selector under the recovery’s integer model.

No direct access to record offset 0, its payload, or its total extent is visible. Dependencies may access additional storage.

### Output

The visible return container is 32 bits.

- If U-01 returns zero, the returned bit pattern is zero.
- Otherwise, the result comes from the selected final processing dependency.
- The conversion result is an intermediate value, not independently returned.

No output parameter is visible. The meaning of a nonzero return, including whether it indicates success, failure, a handle, or another quantity, is not established.

## Behavior

### 1. Perform the receiver-dependent gate

The function first obtains and invokes U-01 through the receiver’s dispatch structure.

The dispatch operation occurs before any direct read of the record selector. The receiver is supplied as the visible argument; the record is not supplied to this operation.

Only the distinction between zero and nonzero in U-01’s result affects routing:

- **Zero:** return zero without invoking the conversion, either specialized processor, or the fallback.
- **Nonzero:** continue to the selector read.

The function does not normalize a nonzero gate result into its final result and does not otherwise use its numeric magnitude.

A zero result does not imply that the call was side-effect-free. U-01 has already executed and may have changed memory or external state.

### 2. Read and classify the selector

After U-01 returns nonzero, the function reads the selector from record byte offset 4.

This ordering matters: the selector used for routing is the value observed after U-01 completes, not necessarily the value present at function entry. If U-01 can reach and modify that storage through aliases or other state, the later read observes the resulting state subject to the execution environment’s memory rules.

The visible function performs one selector read and classifies that value into three disjoint categories:

| Selector | Selected path |
|---|---|
| 7 or 13 | First conversion-and-processing path |
| 21 | Second conversion-and-processing path |
| Any other value | Fallback |

There is no visible range validation or rejection of negative, zero, or otherwise unrecognized selector values. Any value not equal to 7, 13, or 21 selects the fallback.

### 3. Process selectors 7 and 13

For either selector:

1. F-682adfd5a561 is called with the original record value as its first argument and 2560, hexadecimal 0x0A00, as its second.
2. After that call returns, its 32-bit result is passed as the second argument to F-9a45315b9740. The original receiver is the first argument.
3. The result of F-9a45315b9740 becomes the function’s return value.

There is no visible distinction between selectors 7 and 13 after classification.

The conversion result is not checked for zero, a sentinel, or any other condition before being passed onward. Thus a zero-valued conversion result still leads to the processor call, provided the conversion dependency returns normally.

### 4. Process selector 21

For selector 21:

1. F-682adfd5a561 is called with the original record value and 13568, hexadecimal 0x3500.
2. Its 32-bit result is passed to F-e9270c70c772 as the second argument, with the original receiver as the first.
3. The result of F-e9270c70c772 becomes the function’s return value.

As in the preceding path, there is no visible intermediate-result validation, gate recheck, or selector reread.

### 5. Process all other selectors

For every other selector, the function calls a fallback dependency with the original receiver and original record value. This otherwise unidentified dependency is designated **U-02**.

The fallback result becomes the function’s return value. F-682adfd5a561 and both specialized processors are bypassed.

### Arithmetic and comparisons

The visible constants are exact:

- Dispatch entry offset: 352 bytes.
- Selector offset: 4 bytes.
- Selector values: 7, 13, and 21.
- Conversion parameters: 2560 and 13568.

The selector tests are equality comparisons. Their outcomes for these positive constants do not depend on a signed ordering interpretation.

No payload-length arithmetic, allocation-size arithmetic, multiplication, or iteration is visible. Address formation uses the two fixed offsets. Overflow, invalid addressing, alignment violations, and inaccessible memory are not checked or given a defined recovery path by the supplied body.

## Side effects

### Direct effects

The function visibly reads:

1. The receiver’s initial dispatch-base field.
2. The callable entry at dispatch-base offset 352.
3. The record selector at byte offset 4, but only after a nonzero gate result.

No direct write to receiver storage, record storage, or global storage appears in the supplied body. There is no visible direct allocation or release.

### Dependency effects and ordering

All dependency side effects remain possible unless constrained by their own contracts. In particular, this function does not establish that any dependency:

- Preserves global state.
- Leaves the receiver or record unchanged.
- Preserves data reachable through aliases.
- Allocates, retains, transfers, or releases memory.
- Returns normally.

For a recognized selector, conversion completes before the specialized processor begins. Consequently, processor behavior may depend on state changed by conversion. The function neither restores earlier state nor revalidates the receiver, record, or intermediate result between those calls.

There is no visible rollback or cleanup operation after a dependency call.

### Ownership and lifetime

No ownership transfer is established by the call sites alone. Passing the record to conversion does not prove that conversion consumes it. Passing the conversion result onward does not prove that it is an owned pointer or that the processor releases it.

Inputs and dependency-accessed storage must remain usable for the accesses actually made. The complete lifetime obligations are documented_only because the dependency contracts are unavailable.

## Dependencies

| Identifier | Visible role | Visible arguments | Use of result |
|---|---|---|---|
| U-01 | Receiver-dispatched gate | Receiver | Zero stops processing; nonzero enables selector dispatch |
| F-682adfd5a561 | Conversion dependency | Original record; 2560 or 13568 | Passed directly to the selected specialized processor |
| F-9a45315b9740 | Processor for selectors 7 and 13 | Original receiver; conversion result | Returned |
| F-e9270c70c772 | Processor for selector 21 | Original receiver; conversion result | Returned |
| U-02 | Fallback dependency | Original receiver; original record | Returned |

U-01 and U-02 are local opaque identifiers used because published function IDs for these dependencies were not supplied. They do not assert identities beyond their distinct visible call sites.

All normal-return descriptions are conditional on the invoked dependencies returning normally. No dependency-specific error or exception contract is established.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not reports of executed tests.

### Disabled processing

Given a valid receiver dispatch and U-01 returning zero:

- The result is the zero 32-bit value.
- No direct selector read occurs.
- No conversion, specialized processor, or fallback call occurs.

An unreadable record is therefore not directly dereferenced by this function on that path. This does not guarantee safety if U-01 independently accesses that storage.

### Shared path for selectors 7 and 13

Given U-01 returning nonzero and the subsequent selector read yielding either 7 or 13:

- Conversion receives the original record and 2560.
- F-9a45315b9740 receives the original receiver and exactly the conversion result.
- The final returned value is F-9a45315b9740’s result.
- The other specialized processor and fallback are not called.

### Selector 21

Given U-01 returning nonzero and the selector read yielding 21:

- Conversion receives the original record and 13568.
- F-e9270c70c772 receives the conversion result.
- Its result is returned.

### Unrecognized selector

Given U-01 returning nonzero and a selector such as 0, 8, or −1 under the recovered signed integer interpretation:

- U-02 receives the original receiver and record.
- No conversion occurs.
- U-02’s result is returned without a visible success/failure interpretation.

### Zero conversion result

Given a recognized selector and conversion returning zero normally, the selected processor is still called with zero as its second argument. The wrapper does not substitute a fallback or return early.

### Mutation during dependency execution

If U-01 changes the selector from 7 to 21 before returning nonzero, and the subsequent selector read observes 21, the selector-21 path is required.

If conversion changes the record selector after it has been read, the already selected processing path remains unchanged. There is no visible selector reread.

## Uncertainties

The following remain documented_only:

- Native pointer width, endianness, alignment rules, and exact calling conventions.
- Whether the dispatch structure represents a language-level virtual table or another dispatch mechanism.
- The complete receiver and record layouts.
- The semantic meanings of the selector values and conversion parameters.
- Whether conversion returns a scalar, pointer-like value, token, or status.
- Ownership, retention, aliasing, and release behavior within dependencies.
- Dependency error conventions, nonlocal exits, and synchronization requirements.
- Whether callers guarantee non-null, aligned, accessible inputs.

“Gate,” “conversion,” and “processor” describe structural roles. They are not evidence of domain-specific validation, transformation, or success semantics.

## Evidence

Established static evidence consists of the supplied function body and dependency identity map. It supports the dispatch offsets, call ordering, selector partition, constant arguments, direct argument forwarding, absence of visible intermediate checks, and final result selection described above.

The 32-bit return and intermediate-value descriptions follow the supplied four-byte recovery types. The selector-width description follows the recovery’s integer model; complete target ABI details were not supplied.

No dependency implementations or runtime observations were supplied. No runtime tests have been performed. The acceptance examples express required behavior conditional on the stated inputs and dependency outcomes; they are not execution evidence.
