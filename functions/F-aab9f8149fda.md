<!-- Trusted provenance: F-aab9f8149fda; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aab9f8149fda

## Purpose

F-aab9f8149fda performs a mode-dependent operation followed by a common operation, using transient configuration storage and two forwarded 32-bit arguments. It also writes fixed-width fields into a caller-supplied destination.

The established behavior is an ordered sequence of dependency calls, global-value reads, local preparation, and destination writes. The source does not establish the application-level meaning of the configurations, the objects returned by dependencies, or the fields written to the destination.

The distinction between modes is solely whether the second argument equals zero. All nonzero values take the same alternative path.

## Interface

The function accepts four arguments:

| Argument | Width and interpretation |
|---|---|
| Destination | A 32-bit address-bearing value used as the base for destination writes. |
| Mode | A 32-bit integer tested for equality with zero. |
| Forwarded argument A | An opaque 32-bit value passed unchanged to operation dependencies. |
| Forwarded argument B | An opaque 32-bit value passed unchanged to operation dependencies. |

There is no explicit return value. No status or dependency result is returned to the caller.

For ordinary successful execution, the destination must support the writes reached by the selected path. The function performs no visible destination validation, alignment check, bounds check, or null check.

The destination layout used by this function is:

| Byte offset from destination | Access width | When written | Value source |
|---|---:|---|---|
| `0x18` | 32 bits | Zero mode only | First destination-value global |
| `0x1C` | 8 bits | Zero mode only | Literal `9` |
| `0x20` | 32 bits | Zero mode only | Second destination-value global |
| `0x24` | 32 bits | Both modes | Third destination-value global |
| `0x28` | 8 bits | Both modes | Literal `10` |
| `0x2C` | 32 bits | Both modes | Fourth destination-value global |

The furthest explicit write covers bytes `0x2C` through `0x2F`. A contiguous destination object satisfying all these accesses therefore extends through at least byte `0x2F`, although this function does not access every byte in that range.

The byte writes at `0x1C` and `0x28` are not whole-word writes. In particular, this function does not explicitly clear the three following bytes at either location.

## Behavior

### Initial configuration preparation

Before examining the mode, the function invokes an external transfer-shaped primitive twice.

1. The first invocation receives a first 104-byte local configuration region, a first global-supplied source operand, and the length `104`.
2. The second invocation receives a separate 104-byte local configuration region, a second global-supplied source operand, and the length `104`.

Both invocations precede either mode-specific operation.

The call shape supports interpreting these as 104-byte copies into local storage. However, the primitive’s definition is absent. Its exact read/write semantics, fault behavior, and effects on other state are not established independently by this source. Subsequent descriptions of configuration contents are conditional on this apparent copy behavior.

The two source operands are obtained at their respective invocation points. The function does not explicitly snapshot both before invoking the primitive. Consequently, if an earlier invocation can change a later source global or its contents, the later invocation observes the resulting state according to that primitive’s semantics.

### Zero-mode path

When the mode is zero, the function performs the following operations in order:

1. It calls F-8157a180d901 with the 32-bit value `0x220`, equivalent to `544`.
2. It passes the resulting 32-bit value to F-7f6becd152e6 as the first argument. The remaining arguments, in order, are:
   - a literal zero;
   - forwarded argument A;
   - forwarded argument B;
   - the address of the first local configuration region.
3. After that call returns normally, it reads the first destination-value global and writes its 32-bit value at destination offset `0x18`.
4. It writes the 8-bit value `9` at destination offset `0x1C`.
5. It reads the second destination-value global and writes its 32-bit value at destination offset `0x20`.

The return value of F-7f6becd152e6, if its interface has one, is not used.

The destination values are read after the operation dependency returns. They must not be treated as values captured at function entry. A dependency that changes those globals can affect the subsequent writes.

### Nonzero-mode path

Every nonzero mode follows this path, regardless of sign or magnitude.

The function prepares five adjacent 32-bit configuration words with the following values:

| Relative offset in the composite local layout | Value |
|---|---|
| `104` | `30` |
| `108` | A fifth global-supplied value |
| `112` | `2` |
| `116` | `25` |
| `120` | `5` |

These assignments occur in the listed order, before the additional transfer invocation described below. In particular, the global at offset `108` is read during this preparation, not immediately before the later operation call.

The function then:

1. Invokes the external transfer-shaped primitive with a composite local region as destination, the first local configuration region as source, and length `104`.
2. Writes a 32-bit zero at relative byte offset `48` in that composite region.
3. Reads another global-supplied 32-bit value and passes it to F-8157a180d901.
4. Calls F-4a73b9be0f3a with the returned 32-bit value as its first argument. Its remaining arguments are:
   - a literal zero;
   - forwarded argument A;
   - forwarded argument B;
   - the address of the composite local region.

Under ordinary 104-byte copy semantics, the composite configuration presented to F-4a73b9be0f3a has this layout:

| Relative byte range | Contents |
|---|---|
| `0–47` | Corresponding bytes from the first configuration |
| `48–51` | A 32-bit zero, overriding the copied bytes |
| `52–103` | Corresponding bytes from the first configuration |
| `104–107` | 32-bit value `30` |
| `108–111` | Captured global-supplied 32-bit value |
| `112–115` | 32-bit value `2` |
| `116–119` | 32-bit value `25` |
| `120–123` | 32-bit value `5` |

Thus the visible layout is not merely a 48-byte buffer. The transfer covers 104 bytes, the override lies inside that transferred span, and the five prepared words immediately follow it in the displayed local layout.

The source establishes the addresses and preparation of these fields, but not how much of the region F-4a73b9be0f3a actually reads.

This path contains no explicit destination writes at offsets `0x18`, `0x1C`, or `0x20`. That does not establish that those locations remain unchanged: dependencies may have effects not shown here.

### Common operation and final writes

After either mode-specific path completes normally, the function:

1. Calls F-8157a180d901 with `0x220`.
2. Calls F-7f6becd152e6 with:
   - that returned 32-bit value;
   - a literal zero;
   - forwarded argument A;
   - forwarded argument B;
   - the address of the second local configuration region.
3. Reads the third destination-value global and writes its 32-bit value at destination offset `0x24`.
4. Writes the 8-bit value `10` at destination offset `0x28`.
5. Reads the fourth destination-value global and writes its 32-bit value at destination offset `0x2C`.
6. Returns without a value.

The common operation uses the second configuration region prepared before mode selection. There is no explicit refresh of that region immediately before this call. Its contents at that time depend on the earlier preparation and on any intervening effects.

The final global reads occur after the common F-7f6becd152e6 invocation returns.

## Side effects

### Direct destination effects

The direct stores are separate, ordered operations, not an atomic update.

On a normally completed zero-mode path, the function explicitly issues six destination stores: three mode-specific stores followed later by three common stores.

On a normally completed nonzero-mode path, it explicitly issues only the three common destination stores.

No explicit stores target offsets `0x1D–0x1F` or `0x29–0x2B`. Other untouched destination bytes likewise receive no direct store from this function.

A fault or nonlocal exit can leave a partially updated destination. There is no visible rollback.

### Dependency and global effects

The function makes two calls to F-8157a180d901 on either normal path. It does not check either returned value before forwarding it.

Zero mode calls F-7f6becd152e6 twice. Nonzero mode calls F-4a73b9be0f3a once and F-7f6becd152e6 once.

No assumption is made that these dependencies preserve globals, destination memory, or local configuration contents reachable through supplied pointers. The explicit destination stores happen after the relevant dependency calls and can overwrite effects those calls have already produced.

### Ownership and lifetime

Configuration pointers refer to storage local to this invocation. The function provides no explicit long-lived allocation for these configurations.

The 32-bit results from F-8157a180d901 are forwarded directly to operation dependencies. This source does not establish whether they are addresses, handles, allocated objects, or another kind of token.

No release operation is visible. This is not evidence that ownership is transferred, that no release is required, or that a leak exists. Those questions depend on unavailable dependency contracts.

Retention of a local configuration pointer beyond the invocation is not made safe by anything visible here.

### Errors and arithmetic

There is no explicit error branch, retry, cleanup path, or result validation.

Destination addressing consists of fixed positive byte offsets from the 32-bit base. The source does not establish valid behavior when an address calculation overflows the address space, produces an invalid address, or violates target alignment requirements.

The 32-bit and 8-bit stores have distinct widths. Byte order for multibyte memory representations is not established by this excerpt.

## Dependencies

| Dependency | Established use |
|---|---|
| F-8157a180d901 | Receives one 32-bit argument and supplies a 32-bit result that is forwarded to another dependency. Called twice on each normal path. |
| F-7f6becd152e6 | Receives five arguments: the preceding result, zero, two forwarded values, and a local configuration pointer. Used for the zero-mode first operation and for the common operation. |
| F-4a73b9be0f3a | Receives the same five-argument shape, using the composite configuration. Used only for nonzero mode. |
| External transfer-shaped primitive; opaque ID unavailable | Invoked with destination-like operand, source-like operand, and length `104`. Called twice in zero mode and three times in nonzero mode. Exact semantics are documented_only. |

The naming of the first three dependencies conveys no additional behavioral guarantee.

## Acceptance examples

These examples specify conditional observable behavior. They are not runtime test results.

### Zero mode with normally returning dependencies

Given a valid destination, zero mode, and dependencies that return normally:

- Both initial configuration-preparation invocations occur before the first F-8157a180d901 call.
- F-8157a180d901 receives `544` twice.
- F-7f6becd152e6 is invoked first with the first configuration pointer and later with the second.
- The direct destination-write sequence is: 32 bits at `0x18`, byte `9` at `0x1C`, 32 bits at `0x20`, then—after the common operation—32 bits at `0x24`, byte `10` at `0x28`, and 32 bits at `0x2C`.

An implementation that writes the zero-mode fields before the first operation returns would violate this ordering.

### Any nonzero mode

Given any nonzero 32-bit mode:

- The same alternative branch is selected.
- F-4a73b9be0f3a is used for the first operation.
- The first F-8157a180d901 argument comes from a global rather than the fixed value `544`.
- The common F-8157a180d901 argument remains `544`.
- No direct stores to destination offsets `0x18`, `0x1C`, or `0x20` are issued by this function.

No distinction is made between a mode of `1`, a negative mode, and other nonzero bit patterns.

### Composite configuration under copy semantics

If the transfer primitive copies exactly 104 bytes and does not disturb neighboring storage, and no intervening dependency changes the prepared configuration:

- Composite bytes `48–51` are zero when F-4a73b9be0f3a is invoked.
- Copied bytes outside that four-byte override match the first configuration.
- The five adjacent words at offsets `104` through `120` have the prepared values listed above.

Treating the zero override as a single-byte write, or placing it at offset `104`, would be incorrect.

### Global mutation by a dependency

If the first zero-mode operation changes the first destination-value global before returning, the subsequent store at destination offset `0x18` uses the changed value.

Likewise, if the common operation changes the final destination-value globals, the stores at offsets `0x24` and `0x2C` use their values when those reads occur. Entry-time snapshots are not equivalent.

### Failure before common completion

If the common operation does not return normally, the three common destination stores are not reached in the shown control flow. Earlier effects are not undone.

## Uncertainties

The following are documented_only:

- Exact contents and meanings of the initial configuration sources.
- Exact semantics of the external transfer-shaped primitive.
- Values and application meanings of the global-supplied words.
- Whether the argument to F-8157a180d901 denotes a size, identifier, or something else.
- Validity rules, failure representations, and ownership of its results.
- Configuration schemas expected by the operation dependencies.
- Whether dependencies mutate globals, retain pointers, release resources, or modify reachable memory.
- Target endianness, alignment requirements, and behavior on invalid addresses.
- Whether the wider application permits aliasing between the destination and global storage.

Inferred intent: the function appears to prepare two related operations, with an alternate first configuration for nonzero mode, and then record descriptor-like fields in the destination. That interpretation is not required to reproduce the established call and write behavior.

## Evidence

The supplied static source establishes the zero/nonzero branch, call order, argument forwarding, explicit constants, destination offsets, access widths, and relative local configuration layout.

The composite offset `48` follows from the displayed local layout. The five trailing 32-bit fields begin at relative offset `104`, immediately after the 104-byte transfer span.

The source supplies no definitions for the dependencies, no global contents, and no runtime observations. No runtime tests have been performed.
