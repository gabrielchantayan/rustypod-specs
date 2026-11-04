<!-- Trusted provenance: F-6651d6fbf9ca; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6651d6fbf9ca

## Purpose

F-6651d6fbf9ca is a selector-driven dispatch operation. It first asks a dependency to process an input and a mutable 32-bit selector. If that dependency reports success, the operation uses the selector’s low eight bits to choose one of six branches.

Each supported branch invokes a common dependency with a branch-specific argument, then invokes a distinct no-argument dependency to obtain a 32-bit result. The operation stores that result through the caller’s output pointer and returns zero.

A nonzero status from the initial dependency, or an unsupported low-byte selector, causes a return value of one without a direct write to the output location.

The broader meaning of the selector, branch arguments, and returned result is not established. In particular, the result must not be assumed to be an owned allocation, object handle, or pointer.

## Interface

### Inputs

The visible interface has four arguments:

| Position | Representation | Established use |
|---|---|---|
| First | Opaque 32-bit value | Not directly read or forwarded by this operation. |
| Second | Opaque 32-bit value | Passed unchanged as the first argument to F-79e7676a2ca1. |
| Third | Pointer to a 32-bit output slot | Receives the selected branch’s result on the successful path. |
| Fourth | Unsigned 32-bit selector seed | Copied into a local 32-bit slot whose address is passed to F-79e7676a2ca1. |

The second argument’s semantic type is unknown. Its visible width does not establish whether it represents an integer, address, identifier, or other encoded value.

The output slot is four bytes wide. No surrounding structure, field offset, byte order, or additional output layout is established.

### Return value

The operation returns a 32-bit status:

- **0:** A supported branch completed, and its 32-bit result was directly stored in the output slot.
- **1:** F-79e7676a2ca1 returned a nonzero status, or the selector’s low byte was outside the supported range.

This status describes the wrapper’s visible control flow. It does not establish that the selected dependencies succeeded according to any broader semantic contract.

### Preconditions

For a successful return, the output pointer must designate storage suitable for a 32-bit write under the execution environment’s memory and alignment rules. No explicit pointer validation is visible.

The initial dependency receives the address of a local selector slot. That address is available during the dependency call; this operation supplies no guarantee that retaining or later accessing it is valid.

Requirements imposed by dependencies on the second argument, branch-specific arguments, global state, or execution context are unknown. This operation does not visibly validate them.

## Behavior

### Initial selector processing

The operation first initializes a local 32-bit selector slot from the fourth argument. The full 32-bit value is supplied to F-79e7676a2ca1 through that slot’s address, together with the unchanged second argument.

Consequently, the initial dependency can observe the full selector seed, not merely its low byte. It can also modify the selector slot before returning.

The dependency’s status is tested for equality with zero:

- Any nonzero status causes an immediate return of one.
- Zero permits selector dispatch.

The operation does not propagate the dependency’s particular nonzero status. All such statuses are reduced to the wrapper status one.

If the dependency reports failure after modifying the selector slot, the modified selector is not used for dispatch.

### Selector interpretation

After a zero status from F-79e7676a2ca1, the operation reads the selector slot and considers only its low eight bits.

The supported selector values are 0 through 5 inclusive. Values 6 through 255 are unsupported and cause a return of one.

The upper 24 bits do not affect this dispatch decision. They may nevertheless have affected the initial dependency, which received the complete 32-bit seed and access to the complete selector slot.

There is no visible signed range test, overflow-sensitive calculation, or arithmetic transformation of the selector. The relevant operation is an eight-bit mask applied to a 32-bit value.

### Branch selection and call order

Each supported selector produces the following ordered pair of calls:

| Selector low byte | Argument supplied to F-8157a180d901 | Subsequent result-producing dependency |
|---|---|---|
| 0 | Value read from the first branch-specific data location | F-6cc10c166978 |
| 1 | Value read from the second branch-specific data location | F-6f5f44f1867e |
| 2 | 0x2E0, decimal 736 | F-2ec5f215188c |
| 3 | 0x440, decimal 1088 | F-62950bd9317d |
| 4 | 0x40, decimal 64 | F-2d3e9624722e |
| 5 | 0x38, decimal 56 | F-1dd12713926d |

For selectors 0 and 1, the visible evidence identifies data-backed arguments but does not provide their values. They must not be replaced with guessed constants or inferred sizes.

For each branch:

1. F-8157a180d901 is invoked with the corresponding argument.
2. After it returns, the selected result-producing dependency is invoked without explicit arguments.
3. After that dependency returns, its 32-bit result is written to the caller’s output slot.
4. The operation returns zero.

The visible operation does not inspect a result from F-8157a180d901. It proceeds to the selected result-producing dependency if the first call returns normally.

The selected result-producing dependency’s returned bits are copied directly into the output slot. There is no visible conversion, range check, null check, or interpretation of that result. A zero-valued result therefore still leads to wrapper status zero.

### Output timing and failure behavior

The wrapper’s direct output write occurs only after both branch calls return. It does not initialize or clear the output slot before selector processing.

On an initial-dependency failure or unsupported selector, the wrapper performs no direct write through the output pointer.

This is narrower than a guarantee that the output memory remains unchanged. Dependencies may have access to the same memory through aliases, global state, or the second argument. Their side effects are not established here.

If a dependency does not return normally, or if an attempted memory access is invalid, the evidence does not establish a normal status return. No local recovery mechanism is visible.

## Side effects

### Direct effects

The operation directly:

- Initializes a local 32-bit selector slot.
- Exposes that slot’s address to F-79e7676a2ca1.
- Reads the selector for dispatch after that dependency reports success.
- Reads one branch-specific data value when selector 0 or 1 is selected.
- Writes one 32-bit result through the output pointer on a successful path.

No direct write to a global data location is visible.

The first argument has no visible effect on these actions. Holding other behavior constant, changing that argument does not change the wrapper’s direct dispatch logic.

### Dependency-mediated effects

Every dependency may have effects beyond those visible at its call site. The evidence does not establish that any dependency:

- Preserves globals.
- Avoids allocation or deallocation.
- Preserves memory reachable through other arguments.
- Is reentrant or thread-safe.
- Cannot terminate execution, transfer control nonlocally, or fail to return.

For selectors 0 and 1, the branch-specific data value is obtained after the initial dependency returns. Its value therefore cannot safely be treated as an entry-time snapshot: the initial dependency could affect relevant global state.

Likewise, the call to F-8157a180d901 precedes the selected result-producing dependency. The latter may observe state changes caused by the former. The absence of an explicit argument connecting the calls does not establish their independence.

### Ownership

The wrapper exposes no explicit ownership protocol.

It does not directly allocate, free, retain, release, or destroy the returned 32-bit value. It merely writes the value supplied by the selected dependency. Whether that value denotes a resource, and whether the caller acquires any responsibility for it, is documented_only.

The output storage remains caller-provided storage; no transfer of ownership of that storage is visible.

## Dependencies

| Opaque function ID | Established role at this call site |
|---|---|
| F-79e7676a2ca1 | Receives the second argument and a pointer to the local selector slot. Its zero/nonzero status determines whether dispatch is attempted. |
| F-8157a180d901 | Called once on each supported branch, before the result-producing dependency, with the branch-specific argument. No return value is used by this operation. |
| F-6cc10c166978 | Supplies the 32-bit result for selector 0. |
| F-6f5f44f1867e | Supplies the 32-bit result for selector 1. |
| F-2ec5f215188c | Supplies the 32-bit result for selector 2. |
| F-62950bd9317d | Supplies the 32-bit result for selector 3. |
| F-2d3e9624722e | Supplies the 32-bit result for selector 4. |
| F-1dd12713926d | Supplies the 32-bit result for selector 5. |

These roles describe visible argument flow and ordering, not the dependencies’ internal contracts.

## Acceptance examples

These are conditional acceptance cases derived from static evidence. They are not reports of executed tests. Cases expecting a successful return assume valid output storage and normally returning dependencies.

### Initial dependency rejects the input

Given that F-79e7676a2ca1 returns any nonzero status:

- The wrapper returns one.
- F-8157a180d901 is not called by the wrapper.
- No result-producing dependency is called by the wrapper.
- The wrapper performs no direct output write.

This remains true if the initial dependency changed the local selector to a supported value before returning its nonzero status.

### Selector modified into a supported branch

Given an initial seed of 0x00000009, and F-79e7676a2ca1 replacing the selector with 0xABCD0002 while returning zero:

- Selector 2 is chosen.
- F-8157a180d901 receives 736.
- F-2ec5f215188c is called next.
- Its 32-bit result is written to the output slot.
- The wrapper returns zero.

The original seed does not independently control dispatch after the selector slot has been modified.

### Upper bits do not change dispatch

Given that the selector slot contains 0xFFFFFF05 after the initial dependency returns zero:

- Selector 5 is chosen.
- F-8157a180d901 receives 56.
- F-1dd12713926d supplies the output value.

This does not imply that supplying 0xFFFFFF05 and 0x00000005 as initial seeds has identical overall effects; F-79e7676a2ca1 sees the full seed.

### Unsupported low byte

Given that F-79e7676a2ca1 returns zero and leaves the selector as 0x00000106:

- The low byte is 6.
- The wrapper returns one without either branch call.
- No direct output write occurs.

The presence of other set bits does not make the selector supported.

### Zero-valued branch result

Given a supported selector and a selected result-producing dependency that returns 0x00000000:

- The output slot is written with 0x00000000.
- The wrapper returns zero.

A zero result is not converted into wrapper failure.

### Data-backed branch argument

Given selector 0 and a value available from its branch-specific data location:

- That value is supplied to F-8157a180d901.
- F-6cc10c166978 is called afterward.
- The output comes from F-6cc10c166978, not from the data location or from a return value of F-8157a180d901.

## Uncertainties

The following remain **documented_only**:

- The semantic meaning of the arguments, selectors, and result values.
- The values and declared widths of the two data-backed branch arguments.
- Whether F-8157a180d901 performs allocation, configuration, selection, or some other operation.
- The dependencies’ memory, global-state, error, and ownership contracts.
- Whether any returned value is a pointer or resource identifier.
- Environmental alignment requirements and behavior on invalid output storage.
- Concurrency requirements and whether branch-specific data may change concurrently.
- Whether dependencies retain the local selector address or access aliased output memory.

**Inferred intent:** The repeated common-call-then-result-call pattern suggests branch-specific preparation followed by result acquisition. That interpretation is not required to explain the established behavior and does not justify assigning allocation or initialization semantics to either call.

## Evidence

The supplied static body establishes the argument widths shown at this interface, the local selector copy, the initial dependency call and zero-status gate, low-byte dispatch, six supported branches, branch call ordering, direct output assignment, and zero/one return convention.

It does not establish dependency implementations, the contents of the two referenced data locations, ownership rules, or behavior under invalid memory access.

No runtime tests have been performed. The acceptance examples specify conditional behavior supported by the visible control flow; they are not execution evidence.
