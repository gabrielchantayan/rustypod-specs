<!-- Trusted provenance: F-56ee0cebf586; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-56ee0cebf586

## Purpose

F-56ee0cebf586 conditionally overwrites an unsigned numeric field at an address selected by a base address and a signed index. It accepts an index below three and, under the recovered 32-bit model, a value between zero and 255 inclusive. Rejected inputs transfer control to F-a817d8d58c10, which the supplied static representation identifies as non-returning.

The index restriction is an upper bound only. Negative indices pass that restriction. The function does not establish that an accepted index selects allocated or writable memory.

A regularly spaced record layout is an inferred interpretation of the address calculation. Neither the existence of an array nor the number of allocated records is established.

## Interface

The function receives three scalar inputs and supplies no return value.

| Input | Meaning | Width and interpretation |
|---|---|---|
| Base | Integer address component used to locate the destination | Signed integer in the supplied representation; treated as 32 bits in the recovered model |
| Index | Position selector used in validation and address formation | Signed integer in the supplied representation; treated as 32 bits in the recovered model |
| Value | Candidate numeric field value | Unsigned integer in the supplied representation; treated as 32 bits in the recovered model |

The destination is represented as an unsigned integer location. Under the recovered model, the write therefore replaces a 32-bit field.

**Width qualification:** the supplied text does not independently establish the target application binary interface or the widths of its integer types. The 32-bit model is the basis for the concrete ranges and field sizes below, not a separate platform declaration. The validation examines value bits numbered eight through 31, with bit zero being the least significant bit. If the unsigned input were wider than 32 bits, that check alone would not reject set bits above bit 31.

A successful invocation requires more than passing validation:

- The address-forming arithmetic must have defined behavior in the execution environment.
- The resulting address must support the indicated unsigned integer write.
- Any applicable alignment, access-permission, object-lifetime, and memory-mapping requirements must be satisfied.

The function does not visibly enforce these requirements.

On the accepted path, the function writes the field and returns without returned data. On the rejected path, it calls the dependency; normal return from that dependency is not represented.

## Behavior

### Admission conditions

The signed index must be strictly less than three. There is no lower-bound restriction.

If that index condition succeeds, the function checks that value bits eight through 31 are all clear. Under the 32-bit unsigned model, this accepts exactly zero through 255 inclusive.

The value check is validation, not permission to truncate an otherwise rejected value. In particular, a value of 256 does not cause a successful write of zero.

The index condition is evaluated first. If it fails, the value condition is skipped. Neither condition explicitly reads the destination or any other memory selected by Base.

For 32-bit signed indices, the validation condition alone admits every representable negative index, as well as zero, one, and two. That does not guarantee defined address arithmetic or a successful write for all such inputs.

### Successful field update

The selected address is displaced from Base by 32 bytes per unit of Index, followed by a further 16-byte displacement.

The resulting offsets for representative indices are:

| Index | Destination relative to Base |
|---:|---:|
| Negative two | 48 bytes before Base |
| Negative one | 16 bytes before Base |
| Zero | 16 bytes after Base |
| One | 48 bytes after Base |
| Two | 80 bytes after Base |

These offsets describe ordinary arithmetic where the intermediate calculations are representable. They do not establish valid addresses.

At the selected address, the function writes the numeric value represented by the lowest eight bits of Value into the entire unsigned destination field. In the recovered 32-bit model, validation ensures that this numeric value equals the original Value.

The write is not limited to one byte. For an ordinary 32-bit destination, the entire field is replaced, including clearing the bits above bit seven when the stored value is nonzero. A zero value clears the whole field.

There is no explicit read of the old field contents and no conditional preservation of any part of that field. After the write completes, the function returns. No dependency call is visible on this path.

The address spacing establishes a 32-byte stride and a 16-byte field displacement. It does not establish the type, ownership, size, or validity of any surrounding object.

### Rejection

An index of three or greater selects the rejection path regardless of Value. An index below three also selects rejection when the checked value bits are not all clear.

Rejection invokes F-a817d8d58c10. No arguments are explicitly supplied in the shown call representation.

The function performs no destination write before invoking that dependency. This is a statement about this function’s explicit operations, not a guarantee that destination memory remains unchanged after the dependency receives control.

The dependency is annotated as non-returning. The available evidence does not distinguish process termination, exception propagation, a nonlocal control transfer, or another mechanism. It also does not establish an error code, diagnostic message, or recoverable failure result.

No behavior is established for an unexpected return from the dependency contrary to that annotation.

### Arithmetic and address validity

The address calculation uses signed integer inputs before the result is treated as a destination address. No overflow, address-range, alignment, or writability check is visible.

Both intermediate multiplication and subsequent additions matter. A mathematically representable final address does not, by itself, establish that every intermediate signed operation is representable.

The contract does not assign wraparound behavior to signed overflow. It also does not establish a portable integer-to-address conversion rule. Consequently, the mathematical offsets in this document are conditional on valid arithmetic and the execution environment’s address interpretation.

Negative indices are not a special rejection case. For example, negative one passes validation and selects a location 16 bytes before Base when the calculation is valid. Whether that location is usable is a caller-side memory-layout question.

If arithmetic or the destination access is invalid, successful completion is not guaranteed. The function shows no recovery mechanism and does not visibly route such failures through F-a817d8d58c10.

## Side effects

On a successful invocation with ordinary memory and the recovered widths, the explicit memory effect is one 32-bit field write. No old destination value is explicitly read.

The numeric result is established; its byte representation depends on target byte order. For example, a stored value of seven produces a 32-bit field whose numeric value is seven, not a field whose previous upper bits survive.

Only the selected field is explicitly written by this function. Any aliases of that field observe the same replacement according to the execution environment’s memory rules. The function performs no alias detection.

The evidence does not establish:

- Atomicity or synchronization.
- Inter-thread visibility or ordering guarantees.
- Volatile or device-register semantics.
- The number or width of physical memory transactions.

No allocation, release, resource acquisition, or ownership transfer is visible on the successful path. Base is used to form an address; it is not visibly retained for later use. The caller remains responsible for the destination’s lifetime and accessibility.

On rejection, the dependency may modify memory, globals, resources, or external state. Its lack of explicit arguments does not establish that it has no access to such state. No preservation or ownership guarantee may be inferred for that call.

## Dependencies

### F-a817d8d58c10

| Property | Established contract |
|---|---|
| Invocation condition | The index restriction or the checked-value-bit restriction fails |
| Explicit arguments | None shown |
| Prior destination access | No destination read or write is explicitly performed by this function before the call |
| Return behavior | Annotated as non-returning |
| Observable role | Receives control on rejected input |
| Unresolved effects | Diagnostics, termination mechanism, memory changes, global changes, resource handling, and ownership effects |

No other function dependency is visible in the supplied representation.

“Failure handler” is a useful interpretation of this dependency’s role, but it does not specify its implementation or its externally observable failure policy.

## Acceptance examples

These are conditional expectations derived from static evidence. They are not runtime test results.

Successful-write examples assume the recovered 32-bit model, defined intermediate arithmetic, a valid integer-to-address interpretation, and a writable destination meeting the execution environment’s access requirements.

| Inputs or condition | Required observable behavior |
|---|---|
| Index zero; Value zero | Replace the 32-bit field 16 bytes after Base with numeric zero, then return without returned data. |
| Index one; Value 255 | Replace the 32-bit field 48 bytes after Base with numeric 255, then return. |
| Index two; Value 128 | Replace the 32-bit field 80 bytes after Base with numeric 128, then return. |
| Index three; Value zero | Invoke F-a817d8d58c10 without this function first writing the destination. The value condition is skipped. |
| Index four; Value 255 | Reject because the signed index restriction fails. |
| Index zero; Value 256 | Invoke F-a817d8d58c10. Do not perform a successful zero-valued destination write. |
| Index two; Value 65,535 | Reject because checked bits above bit seven are set. |
| Index negative one; Value seven | Validation passes. If arithmetic and access are valid, replace the 32-bit field 16 bytes before Base with numeric seven and return. |
| Index negative one; Value 256 | Reject because the value restriction fails, even though the index restriction passes. |
| Very large negative index; Value seven | Validation passes, but no successful-write guarantee follows without establishing arithmetic and address validity. |
| Otherwise accepted inputs; inaccessible destination | No successful return is guaranteed, and no explicit destination-access recovery is shown. |

For a valid ordinary-memory field initially containing numeric 4,294,967,295, an accepted write of seven replaces that entire 32-bit value with seven. It does not preserve the previous upper 24 bits.

For an index of three and an invalid Base, the function’s rejection path does not itself dereference the computed destination. This does not constrain what the dependency might independently access.

## Uncertainties

The following remain **documented_only**:

- Exact integer widths and calling convention beyond the recovered 32-bit model.
- Treatment of value bits above bit 31 if the unsigned input type is wider than assumed. Those bits are not covered by the visible validation mask.
- Signed-overflow behavior and integer-to-address conversion rules.
- Byte order, alignment constraints, address-space rules, and access granularity.
- Allocation boundaries, object identity, and the validity of storage before or after Base.
- Whether the destination is ordinary memory, shared memory, or device-backed storage.
- Whether accepting negative indices is intentional.
- The dependency’s implementation, complete effects, and non-return mechanism.
- Behavior if the dependency returns despite its annotation.

The regularly spaced record interpretation is **inferred intent**. The upper-bound-only index check, checked value-bit range, computed field write, and rejection call are **established static evidence**. Neither category supplies the missing platform or dependency guarantees.

## Evidence

This contract is based solely on the supplied function representation.

The signed comparison establishes the index upper bound and the absence of a lower-bound test. The short-circuit condition establishes validation order. The fixed value-bit check establishes rejection of set bits eight through 31. The address calculation establishes the stride and field displacement. The destination type and final value selection support the full-field unsigned write described under the recovered width model.

The rejection branch contains a dependency call annotated as non-returning. That annotation supports the represented control-flow boundary, not a specific termination mechanism or side-effect policy.

No implementation access, runtime execution, or tests have been performed. The acceptance examples are static consequences subject to the stated preconditions and documented uncertainties.
