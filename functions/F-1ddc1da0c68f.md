<!-- Trusted provenance: F-1ddc1da0c68f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1ddc1da0c68f

## Purpose

F-1ddc1da0c68f conditionally updates state associated with a caller-supplied object. It compares two requested 32-bit values against two object fields. A matching pair produces a success indication without dependency calls, direct object writes, or the opaque global read described below.

If either comparison fails, the function obtains a provider, transforms the second requested value, and invokes an operation through the provider’s dispatch table. Immediately after that operation returns, it reads an opaque 32-bit global into temporary storage, **before testing the operation’s result**. This read occurs for both zero and nonzero provider results.

A nonzero provider result causes the requested pair to be written into the object. A zero result causes an ordered failure-handling sequence involving two helpers and several direct field resets.

“Provider,” “requested pair,” and “update” are descriptive labels. The application meaning of the object and its fields is not established. A cache-like interpretation is plausible but inferred, not part of the required behavior.

## Interface

### Inputs

The supplied representation has four input positions:

| Input | Represented width | Established use |
|---|---:|---|
| Object address | 32 bits | Base for field accesses and addresses supplied to dependencies. |
| First requested value | 32 bits | Compared against offset `0x0C`, supplied to the provider operation, and stored on success. |
| Second requested value | 32 bits | Conditionally compared against offset `0x10`, supplied to F-beb6c6f57d68, and stored on success. |
| Fourth input | 32 bits | Copied to temporary storage on the update path; that storage is overwritten before the provider operation. No distinct effect attributable to its value is established. |

The pair comparisons use full 32-bit equality. No range restriction or signed ordering comparison is shown.

### Object storage

All offsets below are relative to the supplied object address.

| Offset | Width and use |
|---|---|
| `0x08` | Address supplied to F-56a66efac853. No direct field read at this offset is shown. |
| `0x0C` | Four-byte first comparison field. Read first; written on provider success or failure. |
| `0x10` | Four-byte second comparison field. Read only when the first comparison matches; written on provider success, but not directly written on failure. |
| `0x14` | Address supplied to the provider operation and conditionally to F-151ab4bfe553. The extent and internal layout of the associated state are unknown. |
| `0x1C` | Address staged before the provider operation. Directly set to four all-ones bytes on failure. |
| `0x20` | Address staged before the provider operation. Directly cleared with a four-byte write on failure. |
| `0x24` | Address staged before the provider operation. Directly cleared with a one-byte write on failure. |

The final reset at `0x24` does not directly clear bytes `0x25` through `0x27`.

### Opaque global and temporary storage

On every update path whose provider operation returns normally, the function reads one opaque 32-bit global into a local temporary. Its address and application meaning are not specified here.

The read occurs after the provider operation and before the zero/nonzero result branch. Consequently, the loaded value reflects global state at that point, not a value captured before the dependency calls.

On failure, an adjacent temporary word is subsequently set to zero. The address of the temporary containing the global value is used in the address comparison and passed to F-289ec4311c64. The semantic type and full extent of this temporary object are unknown.

### Return value

The represented result is a 64-bit concatenation of two 32-bit words:

| Path | Upper 32 bits | Lower 32 bits |
|---|---|---|
| Both requested values match | First requested value | `1` |
| Provider returns nonzero | Represented object address plus `0x1C` | `1` |
| Provider returns zero | Represented object address plus `0x1C` | `0` |

The lower word is normalized to zero or one; it is not the provider’s original result. Negative nonzero provider results take the same branch as positive nonzero results.

The complete 64-bit value is **not necessarily a Boolean**. In particular, a failure can have a nonzero complete return because its upper word contains an address. Whether the upper word is an intended public result or an artifact of the supplied representation remains documented_only.

### Preconditions

Storage actually accessed must be valid and suitably readable or writable. On the update path, this includes the opaque global, the provider’s dispatch-table reference, and the selected operation.

The function shows no null, bounds, alignment, or address-overflow checks. It establishes no defined error response for invalid storage or invalid dispatch targets.

Addresses and offsets are represented through 32-bit values. No wider-address behavior is established. Behavior when an address addition wraps or required storage crosses the address-space boundary is documented_only.

## Behavior

### Initial comparisons and matching path

The function first reads four bytes at offset `0x0C` and compares them with the first requested value.

Only if that comparison matches does it read four bytes at offset `0x10` and compare them with the second requested value. If the first comparison fails, the initial comparison read at `0x10` is skipped.

If both values match, the function returns with lower word `1` and upper word equal to the first requested value. On this path:

- No dependency is invoked.
- No direct object write occurs.
- The opaque global is not read.
- The state at offsets `0x14`, `0x1C`, `0x20`, and `0x24` is not inspected directly.

There is no additional validity check that can independently reject an otherwise matching pair.

### Provider preparation and invocation

If either comparison fails, the function calls F-56a66efac853 with the object address plus `0x08`. Its result supplies the provider.

It next calls F-beb6c6f57d68 with the second requested value. The result of that call is supplied to the provider operation.

Before invoking the provider operation, it stages the addresses of offsets `0x24`, `0x20`, and `0x1C` into temporary storage, in that visible assignment order. These are not direct writes to those object fields.

The indirect operation is selected through the provider’s dispatch table at byte offset `0x20`. The represented call explicitly supplies:

- The provider as receiver.
- The first requested value.
- The transformed second requested value.
- The object address plus `0x14`.

The three staged addresses suggest additional output arguments, but their actual transmission and argument positions are not fully exposed by the supplied call expression. A complete indirect-call signature is therefore documented_only.

Provider acquisition precedes transformation, and both precede the indirect invocation. The dispatch-table lookup belongs to the later invocation; it is not shown as a lookup captured before F-beb6c6f57d68 runs.

### Unconditional post-provider global read

Immediately after the indirect operation returns, the function reads the opaque 32-bit global and stores the value in the local temporary.

Only after this read does it test whether the provider result is zero.

This ordering applies to both provider-result branches. In particular:

- A successful provider result does not bypass the global read.
- The global read precedes both success-path comparison-field writes.
- On failure, the global read precedes clearing offset `0x0C`.
- Any global mutation performed by the provider before returning can affect the loaded value.
- No preservation of that global by F-56a66efac853, F-beb6c6f57d68, or the provider operation is assumed.

The function does not directly write the global. No later reread of it is shown.

### Nonzero provider result

After the global read, a nonzero provider result causes two direct four-byte writes, in order:

1. The first requested value is written at offset `0x0C`.
2. The second requested value is written at offset `0x10`.

The function then returns lower word `1`, with the represented upper word derived from the address at offset `0x1C`.

Neither failure helper is called on this branch. The loaded global value is not visibly used in any subsequent operation on this branch, but the read itself remains part of the established sequence.

The comparison fields are not updated by these success writes until after the provider operation and global read. Dependencies may nevertheless independently modify those fields.

### Zero provider result

After the global read and the zero-result decision, the function performs this sequence:

1. Clears offset `0x0C` with a four-byte zero write.
2. Sets the temporary word adjacent to the global-value temporary to zero.
3. Compares the object address plus `0x14` with the address of the global-value temporary.
4. If those addresses differ, calls F-151ab4bfe553 with the object address plus `0x14` and a zero-valued second argument.
5. Calls F-289ec4311c64 with the address of the global-value temporary.
6. Writes four bytes representing `0xFFFFFFFF` at offset `0x1C`.
7. Clears offset `0x20` with a four-byte zero write.
8. Clears offset `0x24` with a one-byte zero write.

It returns lower word `0`, with the represented upper word derived from the address at offset `0x1C`.

The first failure helper is conditional on address inequality. The second is reached regardless of that comparison, provided preceding operations return normally.

There is no direct failure-path write to offset `0x10`. Its final value depends on its earlier contents and any dependency effects.

The global value was already loaded before offset `0x0C` was cleared. The failure helpers do not trigger a shown reread of the global. Whether either helper changes the temporary through aliases or affects other state is not established.

## Side effects

The matching path performs only the ordered comparison reads. It has no visible dependency calls, direct writes, or opaque global read.

The update path exposes object-related addresses to dependencies and reads the opaque global after the provider returns. This global access is established even when the provider reports success.

Dependency effects are not constrained by the function body. In particular, the evidence does not establish preservation of:

- The opaque global or other global state.
- Comparison fields or unrelated object storage.
- Provider state or dispatch-table contents.
- Memory reachable through supplied addresses.

Allocation, release, retention, logging, I/O, synchronization, and nonlocal failure behavior remain unknown. Passing an address does not establish a borrow-only contract or prohibit retention.

No explicit ownership transfer of the caller’s object is shown. Conversely, no dependency ownership or lifetime guarantee can be inferred from the absence of a direct release operation here.

The local temporary is invocation-local storage. Its use with F-289ec4311c64 resembles a lifecycle operation, but destruction, deallocation, and resource ownership are not established.

All later actions depend on earlier operations completing normally. For example, failure of the post-provider global access can prevent the result branch and final object writes even when the provider has returned a nonzero value.

There is no visible transaction, rollback mechanism, lock, or atomic multi-field update. Dependency effects can precede the function’s final direct writes. The evidence provides no coherent-snapshot guarantee under concurrent mutation.

## Dependencies

| Opaque dependency | Established interaction | Unresolved contract |
|---|---|---|
| F-56a66efac853 | Called with the object address plus `0x08`; returns the provider used later. | Nullability, lifetime, ownership, mutations, and failure behavior. |
| F-beb6c6f57d68 | Called with the second requested value; its result is supplied to the provider operation. | Transformation semantics, precise result interpretation, and side effects. |
| Indirect provider operation | Selected at dispatch-table byte offset `0x20`; receives the explicitly represented arguments and returns a 32-bit value tested for zero. | Stable opaque function ID, complete calling convention, staged-address arguments, output semantics, and ownership. |
| F-151ab4bfe553 | Called only on provider failure and unequal addresses, with the object address plus `0x14` and zero. | Whether it clears, replaces, releases, retains, or otherwise changes state. |
| F-289ec4311c64 | Called on provider failure with the address of the temporary containing the loaded global value. | Temporary interpretation, mutation extent, lifecycle semantics, and resource effects. |

The opaque global is a data dependency, not a function dependency. Its contents are read after the provider operation on both result branches. No empty-value, type-descriptor, allocator, or null-object interpretation is established.

## Acceptance examples

These are conditional static expectations, not executed tests. Unless stated otherwise, dependencies return normally, addresses are valid, and dependencies do not independently change the fields discussed.

### Matching pair

With requested values `7` and `9`, and fields `0x0C = 7` and `0x10 = 9`:

- The reads occur at `0x0C`, then `0x10`.
- No dependency is called.
- The opaque global is not read.
- No field is directly changed.
- The represented return is `0x0000000700000001`.

### First comparison differs

With first requested value `7` and field `0x0C = 6`, the initial read at `0x10` is skipped. Provider acquisition, transformation, and provider invocation still occur, regardless of whether the existing second field would match.

### Provider succeeds and changes the global

Suppose the provider changes the opaque global from `0x11111111` to `0x22222222`, then returns nonzero. If no intervening mutation occurs:

- The post-provider read loads `0x22222222`.
- That read precedes both direct comparison-field writes.
- The requested first and second values are written at `0x0C` and `0x10`, respectively.
- No failure helper is invoked.
- The lower return word is `1`.

The global read is required even though its loaded value has no further visible use on this branch.

### Provider returns zero

For a zero provider result, the global is first read into temporary storage. The zero-result branch then clears `0x0C`, zeros the adjacent temporary word, and performs the address comparison.

If the addresses differ, F-151ab4bfe553 precedes F-289ec4311c64. If they are equal, only the first helper is skipped.

The final direct writes set `0x1C` to all ones, `0x20` to zero, and the byte at `0x24` to zero. There is no direct write to `0x10`. The lower return word is `0`; the complete return need not be zero.

### Byte-width reset

If bytes `0x24` through `0x27` initially contain `AA BB CC DD`, and dependencies leave them unchanged, failure’s final direct write produces `00 BB CC DD`.

### Matching zeros after failure

Suppose a completed failure leaves both comparison fields zero. A later request for `(0, 0)` takes the matching path without consulting the other reset fields. It returns lower word `1`, invokes no provider, and does not read the opaque global.

## Uncertainties

The following remain documented_only:

- Application meanings of the requested values, fields, and opaque global.
- Semantic type and extent of the state beginning at `0x14`.
- Complete indirect-call signature and staged-address transmission.
- Intended public interpretation of the upper return word.
- Any effect attributable specifically to the fourth input.
- Semantic type and lifecycle of the failure temporary.
- Dependency ownership, retention, allocation, release, global mutation, and reentrancy behavior.
- Behavior under invalid pointers, address wrapping, concurrency, or nonlocal dependency failure.

Zeroing the first comparison field is not proven to invalidate the object for every possible request. The visible matching logic accepts a zero first field when zero is requested.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping. Prior prose, feedback, names, and apparent domain conventions are not independent behavioral authority.

Established static evidence includes the short-circuit comparison reads, direct write widths and order, dependency call order, dispatch-table offset, zero/nonzero branch, and represented 64-bit return construction.

It also establishes the unconditional post-provider 32-bit global read: after the indirect operation returns, before its result is tested, and therefore before either success writes or failure clearing. Only the adjacent temporary’s zeroing is failure-specific.

The provider terminology, cache-like interpretation, likely additional output arguments, and possible lifecycle role of the temporary helpers are inferences rather than established semantics.

No implementation access, runtime execution, smoke checks, or tests have been performed. Acceptance examples express consequences of the static representation under their stated conditions, not observed runtime results.
