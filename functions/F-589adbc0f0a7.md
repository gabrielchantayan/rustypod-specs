<!-- Trusted provenance: F-589adbc0f0a7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-589adbc0f0a7

## Purpose

F-589adbc0f0a7 prepares an input resource, attempts to open an object through an explicitly selected provider or a registry of providers, and performs shared initialization after ordinary provider success. It may publish the resulting reference through a caller-supplied output slot.

An alternate opener is available only after automatic provider selection and only for particular status-byte values. Its success bypasses shared initialization.

“Resource,” “provider,” and “object” describe inferred roles. The established behavior consists of argument handling, field accesses, dependency calls, branch conditions, and stores. Dependency names do not establish ownership, release semantics, or preservation of memory.

## Interface

### Arguments and return value

| Argument | Representation and use |
|---|---|
| Context | Pointer to a context containing an allocation-related reference and a provider registry. No null check is visible. |
| Input description | Pointer to a record containing 32-bit flags and fields. Null is rejected. |
| Selection | Signed 32-bit value forwarded unchanged to opening dependencies. A negative value skips two later initializers. |
| Output | Null or a pointer to one writable 32-bit object-reference slot. Null is permitted only for negative selection. |

The result is a 32-bit unsigned status. Zero controls success branches. The function directly selects status 6 for invalid admitted arguments, status 32 for an explicit provider lacking the required capability, and status 2 when canonicalizing a particular alternate-opener failure.

Other statuses come from dependency returns or the addressable status slot described below. No implicit clearing of the caller’s output slot occurs.

### Required layouts

Pointer-bearing fields use the supplied 32-bit representation.

**Input description**

| Byte offset | Direct interpretation |
|---|---|
| `0x00` | 32-bit flags |
| `0x10` | 32-bit value consulted only when flag `0x02` is set |
| `0x14` | Explicit-provider pointer, consulted when flag `0x08` is set |
| `0x18` | First optional 32-bit opening argument |
| `0x1c` | Second optional 32-bit opening argument |

Flag `0x10` enables the optional arguments. When enabled, the function reads offset `0x1c` before offset `0x18`, although it passes them to the opener in first-argument, second-argument order. When disabled, it supplies two zeros without reading those fields.

**Context**

| Byte offset | Direct interpretation |
|---|---|
| `0x00` | Reference captured after successful preparation and later passed to allocation |
| `0x18` | 32-bit provider-entry count |
| `0x1c` onward | Contiguous 32-bit provider pointers |

A provider’s first 32-bit pointer leads to a word whose bit `0x01` determines eligibility.

**Ordinarily opened object**

| Byte offset | Direct interpretation |
|---|---|
| `0x08` | 32-bit flags |
| `0x1c` | Signed 32-bit auxiliary-record count |
| `0x20` | Pointer to 16-byte auxiliary records |
| `0x4a` | Signed 16-bit normalization field |
| `0x4e` | 16-bit copy destination |
| `0x58` | Reference assigned after the second selection-dependent initializer |
| `0x60` | Pointer used to derive an address 24 bytes beyond it |
| `0x80` | Pointer to six writable 32-bit state words |

Each auxiliary record exposes a signed 16-bit field at offset `0x00` and signed 32-bit fields at offsets `0x08` and `0x0c`.

### Preconditions and status-slot qualification

Except for the two explicit argument checks, valid pointers, accessible records, sufficient storage, and appropriate alignment are unchecked preconditions. Invalid storage has no defined error-status conversion in the visible representation.

The allocation dependency receives the address of the function’s local status slot. Its contract does not establish whether it retains that address. Later dependencies could therefore modify the slot through an escaped alias.

Consequently:

- Later control flow uses the status value present at each explicit check.
- Cleanup return values are ignored, but that does not prove that cleanup preserves the status slot.
- The common return reads the current status slot after any final diagnostic call.
- Publication and a zero return are distinct properties: publication can occur before a later dependency changes the status.
- Ordinary-path outcome claims that require an unchanged status are conditional on dependencies not modifying escaped local slots.

Other local output slots also have their addresses passed to dependencies. Their retention and subsequent mutation are likewise undocumented.

## Behavior

### Admission and preparation

The local opened-object reference starts at zero.

The function returns 6 immediately if the input description is null, or if output is null while selection is nonnegative. These returns perform no dependency calls and no output write.

For admitted arguments, it captures a disposition boolean:

- Flag `0x02` must be set.
- If set, the word at input offset `0x10` must also be nonzero.

Offset `0x10` is not read when the flag is clear. The captured boolean is not recomputed after dependencies run.

F-352a6facb968 receives the context, input description, and an address for producing a prepared-resource reference. Its return is assigned to the status slot. Immediately afterward, the function captures a global diagnostic-state pointer.

A nonzero preparation status bypasses provider selection and both cleanup dependencies, proceeding to the common diagnostic return. This function does not clean up a resource partially produced by failed preparation.

After successful preparation, the context’s first word is captured for later allocation.

### Explicit-provider path

The function rereads input flags after preparation. Explicit selection requires both flag `0x08` and a nonnull provider pointer at offset `0x14`. A set flag with a null pointer uses automatic search instead.

For a usable explicit pointer:

1. Follow its first pointer and inspect capability bit `0x01`.
2. If clear, select status 32 and take resource cleanup followed by object cleanup.
3. Otherwise, read the current optional-argument enable flag and obtain the optional arguments as specified above.
4. Call F-c30df85ef8a0 with the provider, prepared resource, unchanged selection, optional arguments, and the address of the local object-reference slot.

The opener’s return replaces the status slot. Zero enters shared initialization. Nonzero takes resource cleanup followed by object cleanup.

Explicit-provider failure never initiates registry search or alternate opening, including when its status has low byte `0x02` or `0x55`.

### Automatic-provider search

The registry starts at context offset `0x1c`. Its end address is computed once from that start and the entry count. Entries are visited in ascending address order. Count arithmetic, address overflow, and registry bounds are not validated.

For each entry:

- A provider without capability bit `0x01` is skipped.
- For an eligible provider, the function rereads the optional-argument flag and enabled fields.
- F-c30df85ef8a0 receives the same kinds of arguments as on the explicit path.
- Zero enters shared initialization immediately.
- A nonzero result with low byte `0x02` permits another provider attempt.
- Any other nonzero result stops iteration.

The local object-reference slot is not reset between attempts.

After search stops or exhausts the registry, alternate opening is considered only when the current status has low byte `0x02` or `0x55`.

If no provider was attempted, preparation’s zero status remains in the slot. The alternate opener is not called; resource cleanup and object cleanup follow. On this path, the allocation dependency has not received the status-slot address.

### Alternate opening

F-334d9d13d211 receives the context, prepared resource, unchanged selection, caller’s output pointer, and input description. Its return replaces the status slot.

- On zero, F-b24117df4553 receives the prepared resource and captured disposition boolean. The function then returns the current status directly, bypassing shared initialization, object cleanup, and final diagnostics.
- On nonzero with low byte `0x02`, the full status is replaced with exactly 2.
- Other nonzero statuses remain unchanged by the function before cleanup.

Alternate failure takes resource cleanup and then object cleanup.

The alternate-success path does not expose the status-slot address to the allocation dependency. This function makes no direct output write on that path; the alternate opener has direct access to the caller’s output pointer.

### Shared initialization

After ordinary provider success, a diagnostic call is conditionally made when the captured diagnostic-state pointer’s signed 32-bit field at offset `0x1c` exceeds 3.

If the saved disposition boolean is true, bit `0x400` is set in the object flags at offset `0x08`, preserving other bits from that read.

F-3ce007b692cc then receives the captured context reference, requested size 12, and the address of the status slot. Its return supplies an allocation reference; its status output is tested separately.

**An ordinary provider’s success is not sufficient to reach normalization or publication.** If the status slot is nonzero immediately after allocation, the function skips all subsequent initialization and calls object cleanup. It does not directly call resource cleanup on this path.

If the slot is zero:

1. Store the current object reference at allocation offset `0x08`.
2. Read the object pointer at offset `0x60`, add `0x18`, and pass that address to F-24b3b16470f4.
3. Conditionally emit a diagnostic event using the same threshold.
4. Branch on the selection sign.

For negative selection, normalization follows without another status check at this point. A dependency called after allocation could therefore change the escaped status slot without preventing normalization.

For nonnegative selection:

1. Call F-84cf36eb7469 with the object and zero; assign its return to the status slot.
2. If nonzero at the check, take object cleanup.
3. Otherwise, conditionally emit another diagnostic event.
4. Call F-92239182f0ae with the object and an output-reference address; assign its return to the status slot.
5. If nonzero at the check, take object cleanup.
6. Otherwise, store the produced reference at object offset `0x58` and continue to normalization.

The allocated 12-byte record is not explicitly released here. The visible call to F-24b3b16470f4 receives only the derived address; whether it registers the allocation or changes ownership is unknown.

### Normalization and publication

The object flags are read once for the normalization decisions.

With flag `0x01` set:

- A negative signed 16-bit value at `0x4a` is negated and stored back into 16 bits.
- If flag `0x20` is clear, the resulting 16-bit representation is copied from `0x4a` to `0x4e`.

Negating the minimum signed 16-bit value and retaining 16 bits leaves representation `0x8000`.

With flag `0x02` set, auxiliary records are processed in increasing index order. The signed count is reread for every loop condition, and the record-base pointer is read for each processed record. A nonpositive count causes no record accesses.

Within each record, operations occur in this order:

1. Negate the signed 16-bit field at offset `0x00` if negative, storing 16 bits.
2. If the signed 32-bit field at offset `0x08` is negative, multiply by −65536, then shift the product right by 16.
3. Negate the signed 32-bit field at offset `0x0c` if negative, storing 32 bits.

Under wrapping 32-bit multiplication and arithmetic right shift, the second operation produces the signed interpretation of the negated low 16 bits. It is not general 32-bit absolute value. Under wrapping negation, the minimum signed 32-bit value at offset `0x0c` retains representation `0x80000000`. Exact overflowing arithmetic semantics remain qualified below.

The six words reached through object offset `0x80` are written in this order:

| Relative offset | Value |
|---|---|
| `0x00` | `0x00010000` |
| `0x04` | 0 |
| `0x0c` | `0x00010000` |
| `0x08` | 0 |
| `0x10` | 0 |
| `0x14` | 0 |

An identity transform with zero translation is an inferred interpretation.

A nonnull output then receives the current object reference, followed by the common diagnostic return without object cleanup. Null output instead leads to object cleanup. Neither branch establishes an unconditional zero return after the status-slot address has escaped.

## Side effects

Direct writes include the optional object-flag update, allocation field, initializer-produced reference, normalized fields, six-word state block, and ordinary-path output publication. Dependencies may have additional effects not established here.

Cleanup routing is:

| Reached condition | Calls made by this function |
|---|---|
| Preparation failure | Neither cleanup |
| Explicit capability rejection or opener failure | Resource cleanup, then object cleanup |
| Automatic search ends without ordinary success; alternate absent or fails | Resource cleanup, then object cleanup |
| Alternate success | Resource cleanup only |
| Allocation or later initializer fails its status check | Object cleanup only |
| Normalization completes with nonnull output | Neither cleanup |
| Normalization completes with null output | Object cleanup only |

Resource cleanup is F-b24117df4553 with the prepared resource and saved boolean. Object cleanup is F-de1a6df0c88a with the current local object reference, potentially zero.

These calls do not prove release, destruction, or ownership transfer. Their return values do not replace the status slot. Earlier direct writes are not rolled back.

Most non-early-return paths reach a final diagnostic gate. The diagnostic-state pointer is captured once after preparation, but its threshold field is reread at each gate. The final diagnostic receives the status value read for that call; the function reads the slot again for its return. A diagnostic with access to an escaped status alias could change the returned value.

## Dependencies

| Opaque ID | Visible role |
|---|---|
| F-352a6facb968 | Prepare resource and produce its reference |
| F-c30df85ef8a0 | Attempt ordinary provider opening |
| F-334d9d13d211 | Alternate opener with caller-output access |
| F-3ce007b692cc | Request 12 bytes; receive status-slot address |
| F-24b3b16470f4 | Receive an address derived from the opened object |
| F-84cf36eb7469 | First nonnegative-selection initializer |
| F-92239182f0ae | Second initializer; produce a reference |
| F-b24117df4553 | Resource-cleanup-role call |
| F-de1a6df0c88a | Object-cleanup-role call |
| F-4457a9a44a68 | Conditional diagnostic reporting |

No dependency is assumed to preserve globals, input records, object state, registry storage, or escaped local slots. No undocumented ownership convention is imposed.

## Acceptance examples

These are conditional static scenarios, not executed tests. Storage is valid, calls return normally, and dependency side effects leave stated branch conditions intact unless expressly described.

1. **Required output missing.** Nonnegative selection and null output produce status 6 before preparation.

2. **Explicit capability rejection.** Preparation succeeds and the explicit provider lacks capability bit `0x01`. Status 32 is selected; resource cleanup precedes object cleanup. Neither opener runs.

3. **Null explicit pointer.** Flag `0x08` is set but its pointer is zero. Automatic search is used.

4. **Retry classification.** An automatic provider returns `0x00000102`. Another entry may be attempted because continuation uses only the low byte.

5. **Alternate canonicalization.** An eligible alternate attempt returns `0xabcd0002`. The function selects exactly 2 before resource and object cleanup.

6. **Explicit failure cannot fall back.** An explicit opener returns `0x55`. Cleanup follows without alternate opening.

7. **No eligible providers.** Preparation succeeds, and no registry provider is eligible. Neither opener runs. Both cleanups occur; absent dependency mutation of relevant state, zero is returned.

8. **Negative selection with publication.** An ordinary provider succeeds, allocation supplies valid storage and leaves its status output zero, selection is −1, and output is nonnull. With dependencies leaving escaped local slots unchanged, both selection-dependent initializers are skipped, normalization and state reset occur, the object is published, and zero is returned.

9. **Negative selection with null output.** Under the same allocation-success and local-slot-preservation conditions as example 8, output is null. Normalization occurs, then object cleanup; zero is returned without publication.

10. **Allocation failure after opening success.** An ordinary provider succeeds, but allocation leaves status 9. Object cleanup follows without normalization, state reset, publication, or direct resource cleanup. Return is 9 only if later dependencies leave the status slot unchanged.

11. **Escaped status changes before normalization.** Allocation leaves zero and selection is negative. F-24b3b16470f4 then changes the escaped status slot to 9 without changing other relevant storage. There is no intervening status check: normalization and nonnull-output publication still occur. If no later change occurs, the function returns 9.

12. **16-bit boundary.** With flag `0x01` set, `0x8000` at offset `0x4a` retains that representation after the 16-bit store. With flag `0x20` clear, it is copied to `0x4e`.

13. **Non-absolute auxiliary arithmetic.** Under wrapping multiplication and arithmetic right shift, −65536 at auxiliary offset `0x08` becomes zero. Ordinary absolute value would produce a different result.

## Uncertainties

- **documented_only — Escaped local addresses:** Retention, lifetime discipline, and later mutation of the status and other local output slots are undocumented. Ordinary-path status preservation cannot be guaranteed.
- **documented_only — Ownership:** Allocation, registration, release, destruction, partial-failure outputs, and ownership transfers are not established.
- **documented_only — Arithmetic reconstruction:** Overflowing multiplication, minimum-value negation, signed right shift, and address-overflow semantics have not been independently verified. Arithmetic examples identify the model they require.
- **documented_only — Invalid storage:** Malformed pointers, registry bounds, auxiliary storage, and diagnostic-state storage lack visible guarded error handling.
- **documented_only — Semantic roles:** Provider, object, and transform terminology is inferred.
- **documented_only — Mutation and concurrency:** Dependencies may change later-read storage. No synchronization, reentrancy guarantee, or exclusion of aliasing is established.

## Evidence

This contract uses static inspection of the supplied function representation and opaque dependency mapping only.

Established evidence includes argument guards, represented widths and offsets, explicit field-read and store order, dependency arguments, status assignments and checks, low-byte classification, cleanup routing, and the allocation call’s exposure of the status-slot address.

Semantic interpretations and arithmetic-model consequences are identified separately from established operations. Dependency internals and address-retention behavior remain unresolved.

No runtime execution, tests, implementation access, or independent dependency inspection has been performed.
