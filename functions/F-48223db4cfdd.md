<!-- Trusted provenance: F-48223db4cfdd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-48223db4cfdd

## Purpose

F-48223db4cfdd initializes selected fields of a caller-supplied memory region, invokes two dependencies using shared temporary storage, obtains a 16-bit result through indirect dispatch, invokes a final dependency, and stores that result in the caller’s region.

The established behavior is a sequence of memory accesses and calls. The source does not establish a domain meaning for the region, its fields, the selector input, or the returned 16-bit value. Describing the operation as initialization is an inference from its writes, not evidence of allocation, construction, or ownership transfer.

This contract describes normal execution where all accessed addresses are valid and every call returns normally. It does not establish recovery behavior for invalid memory, failed dependency operations, or nonlocal exits.

## Interface

### Inputs

The function accepts four inputs:

| Input | Source-visible representation | Established use |
|---|---|---|
| Destination address | 32-bit integer used as an address | Base for fixed-offset writes and an interior address passed to a dependency |
| Selector | Unsigned 32-bit value | Stored unchanged at offset `0x34`; determines a value written at offset `0x10` |
| Third input | Opaque 32-bit value | Passed twice to F-abeb071e13cc |
| Fourth input | Opaque 32-bit value | Initializes temporary storage and is also passed by value to F-abeb071e13cc |

The source presents a 32-bit address model. A portable host-language implementation must preserve the represented address and access behavior rather than assume that a native integer or pointer has the same width.

There is no established restriction of the selector to Boolean values. Its full unsigned 32-bit range has defined consequences for the direct field writes.

### Return

There is no explicit return value. The observable output is produced through memory writes and dependency effects.

### Destination layout

The function directly accesses the following destination-relative locations:

| Offset | Width | Directly written value |
|---|---:|---|
| `0x04` | 32 bits | Value read from a fixed global data location |
| `0x08` | 32 bits | `6` |
| `0x0C` | 32 bits | `0` |
| `0x10` | 32 bits | `1` if the selector is zero; otherwise `0` |
| `0x14` | 32 bits | `0` |
| `0x18` | Not established | Address passed to F-abeb071e13cc; this function does not directly write this location |
| `0x34` | 32 bits | Selector, without narrowing |
| `0x38` | 32 bits | `0` |
| `0x3C` | 16 bits | Result obtained through indirect dispatch |

The last direct write occupies offsets `0x3C` and `0x3D`. Consequently, ordinary contiguous storage covering all direct accesses must extend through at least offset `0x3D`, for a minimum extent of `0x3E` bytes. This is not a sufficient size guarantee for dependency access: the extent required by the dependency receiving the interior address is unknown.

No direct write to offset `0x00` is shown. No direct write to offsets `0x3E` or `0x3F` is shown. This does not establish that dependencies leave those locations unchanged.

Byte order and alignment requirements are not established by the source. The contract specifies access widths and relative addresses, not a byte serialization format.

### Preconditions

For normal completion:

- Every destination address used by the direct stores must be valid and writable at the indicated width.
- Destination-relative address calculations must identify the intended storage; no bounds or address-overflow checks are shown.
- The interior address at offset `0x18` must satisfy F-abeb071e13cc’s requirements, which are not supplied.
- Fixed global data locations must be readable.
- The object-like value returned by F-641936cb8a81 must support the pointer-chasing and indirect call described below.
- Every dependency must accept the supplied arguments and return normally.

The function contains no explicit validation of these conditions.

## Behavior

### 1. Initial destination writes

The direct writes begin in this order:

1. Store the full unsigned 32-bit selector at offset `0x34`.
2. Store 32-bit zero at offset `0x38`.
3. Store the selector-derived 32-bit value at offset `0x10`.
4. Store 32-bit zero at offset `0x14`.
5. Store 32-bit zero at offset `0x0C`.

The selector-derived value is exactly:

- Selector `0`: value `1`.
- Selector `1`: value `0`.
- Any selector greater than `1`, under unsigned comparison: value `0`.

The source initially forms a subtraction involving `1` and the unsigned selector, but replaces the result with zero for every selector greater than one. Thus no wrapped subtraction result reaches offset `0x10`. In particular, selectors with the high bit set do not produce a negative or nonzero stored value there.

### 2. Global-derived field installation

After those five stores, the function reads one 32-bit value from a fixed global data location. It then:

1. Stores 32-bit `6` at destination offset `0x08`.
2. Stores the previously read global value at destination offset `0x04`.

The global read therefore precedes both of these stores, even though the destination store of that value occurs second.

The global value’s identity and meaning are unknown. It must not be replaced with an assumed constant, pointer, type identifier, or table address.

All seven direct destination stores described so far occur before the first dependency call.

### 3. Shared temporary storage and first dependency

The function creates addressable 32-bit temporary storage and initializes it with the fourth input.

It then calls F-abeb071e13cc with five arguments, in this order:

1. Address of that temporary storage.
2. Destination address plus `0x18`.
3. Third input.
4. Fourth input.
5. Third input again.

Both occurrences of the third input carry the same incoming 32-bit value. The fourth argument carries the incoming fourth input, independently of any later mutation of the temporary storage.

The dependency receives the temporary by address, not merely its initial value. It can therefore observe or alter that storage subject to its own unknown contract. It also receives an interior destination address. No return value from this call is used.

No direct destination write is shown between this call and the subsequent dispatch sequence.

### 4. Second dependency and returned address

After F-abeb071e13cc returns, the function calls F-641936cb8a81 with five arguments:

1. Address of the same temporary storage.
2. A 32-bit value read from a second fixed global data location.
3. Integer value `1`.
4. Integer value `0x20`.
5. A 32-bit value read from a third fixed global data location.

The two global arguments are read for this call after the first dependency has returned. Their order relative to each other is not established by the call expression.

They must not be assumed to retain values they had before F-abeb071e13cc ran. Conversely, the value already written at destination offset `0x04` came from the earlier global read and is not explicitly refreshed.

The return from F-641936cb8a81 is treated as an address. There is no null check or status check before it is dereferenced.

### 5. Indirect dispatch

Using the returned address, the function:

- Reads a 32-bit word at that address.
- Uses that word as a base address and reads a further 32-bit word at offset `0x30`.
- Clears the lowest two bits of the second word, using the 32-bit mask `0xFFFFFFFC`.
- Calls the resulting target address with the returned address as the first argument and `0x20` as the second argument.

This is consistent with table-based dispatch, but the role of either loaded word is not established. In particular, the contract does not assume a particular object model or assign a meaning to the cleared low bits.

The indirect call’s result is consumed as a 16-bit value. The source does not establish whether a wider machine-level return exists or what any additional bits would mean.

There is no check that the two pointer reads are valid or that the masked address is callable.

### 6. Final dependency and result publication

After the indirect call returns, the function preserves its 16-bit result and calls F-07fef63861ea with the address of the same temporary storage.

Only after F-07fef63861ea returns does the function write the preserved 16-bit result to destination offset `0x3C`.

This ordering is part of the contract:

- The indirect result is obtained before F-07fef63861ea.
- F-07fef63861ea runs before the final destination write.
- The final write is 16 bits, not 32 bits.
- The function then returns without an explicit result.

The final dependency’s name or position does not prove that it releases resources. Nor does the source establish that it preserves globals, destination contents, or temporary contents.

## Side effects

### Direct effects

The function performs eight direct destination stores: seven 32-bit stores before any dependency call, followed by one 16-bit store after all calls return.

It reads three fixed global values: one for an early destination store and two for the second dependency’s arguments. It also performs two dependent 32-bit reads to resolve the indirect call target.

The temporary storage is initialized once by this function. Its address is passed to three dependencies. Any intervening changes made through that address remain available to later calls; the function does not reinitialize it.

### Dependency-mediated effects

The dependencies may have additional effects not specified here. Relevant possibilities include writes through supplied addresses, changes to global state, allocation, release, or nonlocal control transfer. None is established or excluded by the supplied source.

Earlier destination writes are not guaranteed to remain unchanged after dependency execution. The final direct store establishes the value at offset `0x3C` upon normal completion, provided the destination remains valid, but does not restore any other field.

### Ownership and lifetime

No direct allocation or deallocation operation is shown. Passing the destination’s interior address does not establish ownership transfer.

The temporary storage belongs to this invocation. The source provides no lifetime guarantee for retaining its address after return. Whether any dependency retains that address, and whether doing so is valid under its own contract, is documented_only.

### Errors and partial execution

There is no explicit error return, rollback, retry, or failure branch.

If a dependency fails to return normally, earlier stores may already be visible and the final 16-bit store may not occur. The source shows no guaranteed finalization on an abnormal exit. In particular, F-07fef63861ea is reached only after the indirect call returns normally.

## Dependencies

| Opaque identifier | Established interaction | Unknown contract |
|---|---|---|
| F-abeb071e13cc | Called first with temporary address, interior destination address, and three opaque value arguments | Initialization semantics, write extent, resource effects, failures |
| F-641936cb8a81 | Called second with temporary address, two global-derived values, `1`, and `0x20`; returns an address used immediately | Return validity, object layout guarantees, ownership, failure representation |
| F-07fef63861ea | Called after indirect dispatch with temporary address | Whether it finalizes, releases, mutates, or performs unrelated work |

A fourth dependency is selected dynamically through the two memory reads and low-bit masking. Its opaque identity cannot be determined from the supplied source. Its established interface is an address argument, a second argument of `0x20`, and a result consumed at 16-bit width.

No dependency is assumed to preserve globals or to acquire, transfer, or release memory.

## Acceptance examples

These examples are conditional behavioral criteria, not executed tests.

### Zero selector

Given a valid destination, selector `0`, valid dependencies, and normal completion:

- The first direct store places `0` at offset `0x34`.
- The direct store at offset `0x10` places `1`.
- The other early stores and all calls occur in the specified order.
- The final store places the indirect call’s 16-bit result at offset `0x3C`.

The final contents of other fields remain conditional on dependency effects.

### Nonzero selector boundaries

Under the same normal-execution assumptions:

| Selector | Direct value at `0x34` | Direct value at `0x10` |
|---:|---:|---:|
| `1` | `1` | `0` |
| `2` | `2` | `0` |
| `0x80000000` | `0x80000000` | `0` |
| `0xFFFFFFFF` | `0xFFFFFFFF` | `0` |

An implementation that stores the selector’s wrapped subtraction result at offset `0x10` for the last three cases does not satisfy this contract.

### Shared temporary mutation

If F-abeb071e13cc changes the temporary storage, F-641936cb8a81 receives the same storage address with that changed content available. If subsequent calls change it again, F-07fef63861ea receives that same address without a reset by this function.

### Dispatch masking

If the dispatch-entry word is `0x1234567B`, the indirect target is `0x12345678`. Acceptance remains conditional on that target being valid and callable. The unmasked value is not the call target.

### Final-store timing and width

Suppose the indirect call supplies the 16-bit result `0xBEEF`. F-07fef63861ea runs before this function writes `0xBEEF` at offset `0x3C`.

If that dependency writes a different value there and returns normally, the final direct store overwrites the same two bytes. This function does not directly overwrite bytes at offsets `0x3E` and `0x3F`.

### Abnormal dependency exit

If the indirect call does not return normally, the source does not establish execution of F-07fef63861ea or the final result store. An unconditional-finalization guarantee would exceed the evidence.

## Uncertainties

The following remain documented_only:

- Destination type, total extent, alignment, and allocation provenance.
- Meanings of all destination fields and fixed global values.
- Dependency write ranges, ownership rules, and error behavior.
- Whether fixed global data locations can alias destination storage.
- Indirect-dispatch object model and significance of cleared address bits.
- Target byte order and exact calling convention.
- Validity of retaining the temporary address beyond this invocation.
- Concurrency, synchronization, reentrancy, and atomicity guarantees.

Initialization and finalization are plausible interpretations of the call sequence, not established dependency semantics.

## Evidence

The supplied static source establishes the explicit access widths, offsets, selector mapping, direct-store order, temporary-address reuse, dependency argument positions, pointer-chasing sequence, dispatch mask, and final 16-bit store timing.

This contract does not rely on comments, prior documentation, or inferred dependency names. No implementation access or runtime tests have been performed. Acceptance examples describe required behavior under stated conditions; they are not observations of execution.
