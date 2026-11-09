<!-- Trusted provenance: F-37abfee82f99; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-37abfee82f99

## Purpose

F-37abfee82f99 processes an object’s existing association, requests a replacement, and selects a continuation using an unsigned 32-bit comparison.

The established control flow is:

1. If an association is present, perform mode-dependent handling, prepare temporary objects, conditionally notify a recipient, and process the association.
2. Request a replacement.
3. If a replacement is returned, prepare comparison values and a packed record, install the replacement, and return a continuation’s result.
4. Otherwise, clear the association, notify a dependency with argument `3`, and return zero.

“Object,” “association,” “recipient,” and “mode” describe roles visible at this interface. They do not establish application-level types, ownership, resource-release behavior, or success semantics.

## Interface

### Arguments and result

| Argument | Visible representation | Established use |
|---|---|---|
| Object | 32-bit pointer | Base for association, recipient, and dispatch-table accesses |
| Mode | 32-bit word | Original value controls zero-mode handling; a mutable copy is used later |
| Auxiliary A | Opaque 32-bit word | Initializes storage exposed to dependencies and later used in packed-record construction |
| Auxiliary B | Opaque 32-bit word | Initializes adjacent storage; no later explicit scalar read is visible |

The result is a 32-bit word:

- Zero if replacement acquisition returns zero.
- The unchanged result of F-910df819391b on the lower-comparison branch.
- The unchanged result of a dynamically selected method on the other comparison branch.

The nonzero-replacement paths do not impose a success/failure interpretation on their returned values.

### Object layout and access requirements

The following fields are accessed on paths that reach them:

| Object byte offset | Access width | Role |
|---|---:|---|
| `0x00` | 32 bits | Dispatch-table address |
| `0xC4` | 32 bits | Association address or zero |
| `0xC8` | 32 bits | Recipient value |

An association used during existing-state processing supplies:

| Association byte offset | Access width | Role |
|---|---:|---|
| `0x14` | 32 bits | Flags |
| `0x1C` | 32 bits | Discriminator compared with signed `−1` |
| `0x20` | Address supplied to a dependency | Source for temporary preparation |

The dynamic continuation is obtained through the object’s dispatch table at byte offset `0xE8`.

The caller and dependency contracts must make every reached access and call valid. There is no explicit null-object check, bounds check, association validation, dispatch-table validation, or recovery for invalid addresses.

### Temporary storage

Before processing the association, the function copies all four arguments into addressable temporary storage. The visible layout places the copied object pointer, Mode, Auxiliary A, and Auxiliary B in consecutive four-byte slots in that order.

These slots are not immutable argument snapshots:

- The copied-object slot is used as the beginning of a temporary object.
- The copied-Auxiliary-A slot is used as the beginning of another temporary object.
- Dependencies receive the address of the copied-Mode slot.
- Packed-record construction reads the then-current Mode and Auxiliary A storage.

A separate eight-byte temporary area is supplied to other dependencies. Another eight-byte area holds the packed continuation record.

The exact object sizes used by dependencies, their access beyond the supplied starting addresses, and their pointer-retention rules are not established. In particular, no assumption that a dependency accesses only one four-byte slot is justified.

## Behavior

### Existing-association processing

The function first reads the object’s association field at `0xC4`. Zero skips this entire phase but does not skip replacement acquisition.

A nonzero entry value enables the following operations. Later association accesses are fresh field reads, not guaranteed uses of that entry value.

### Zero-mode handling

This subphase occurs only when the original Mode argument is zero.

The function loads a 32-bit value through a global location before reading the association’s discriminator at `0x1C`. The association field is read to obtain the discriminator’s base address.

If the discriminator is `−1`, the function reads the association field again to obtain the flags at `0x14`. It calls F-583efa6014bd with the object and zero exactly when both bits selected by mask `0x0000000C` are set.

Consequently:

- `0x04` and `0x08` must both be present.
- Either bit alone is insufficient.
- Other flag bits do not affect this decision.

The global load still occurs on this sentinel path, although its loaded value is not used by the subsequent explicit call.

For any discriminator other than `−1`, the function makes the following calls in order:

1. F-97df3ab481d5 with the loaded global value.
2. F-2b06c99f8427 with that same value and the discriminator.
3. F-0bcbb4e0e550 with that same value and `5`.
4. F-44a900a27091 with that same value.
5. F-e0ed773ee385 with that same value, `0xFFFFFFFF`, and zero.

The global value and discriminator are retained for this sequence rather than refreshed between calls. No return value from these five calls is explicitly tested. The all-one argument is a 32-bit bit pattern; its dependency-specific interpretation is unknown.

A nonzero original Mode skips the global load and the discriminator/flags handling described above.

### Temporary preparation and conditional notification

After zero-mode handling, or immediately for nonzero Mode, the function performs these operations in order:

1. Re-reads the association field and supplies the association-derived address at offset `0x20` to F-b9acea4b06ad, together with the address of the copied-object temporary.
2. Calls F-398e0e9a318a with the copied-Auxiliary-A temporary’s address and an opaque callback/address token.
3. Calls F-44362605ab79 with no explicit arguments.
4. Calls F-4a6c0bf0740c with the current copied-Mode word and the preceding call’s result.
5. Calls F-289ec4311c64 on the copied-Auxiliary-A temporary.

If F-4a6c0bf0740c returned nonzero, the function then reads the recipient field at `0xC8` and calls F-3b9cee72cef2 with that value and the copied-object temporary’s address.

Thus, F-289ec4311c64 on the second temporary precedes the conditional notification. A zero decision skips the notification and its explicit recipient-field read.

The function then:

1. Re-reads the association field and calls F-42168b326728 with the object and that field value.
2. Calls F-289ec4311c64 on the copied-object temporary.

The original nonzero association check does not validate later association values. An intervening dependency may change the field, including to zero. The function performs no explicit revalidation before the later address calculation or processing call.

### Replacement acquisition

After existing-association processing, including when it was skipped, F-fd8692810e8b is called with no explicit arguments.

Zero selects the no-replacement path. Any nonzero 32-bit result selects the replacement path. There is no separate sign, range, alignment, or validity check.

### Nonzero replacement path

The replacement result is retained while the function performs the following ordered operations:

1. Calls F-ebc906d2c2d5 with the replacement.
2. Calls F-dee541b2a059 with that result and the copied-Mode storage address.
3. Calls F-6132bda8e448 with the separate eight-byte temporary area.
4. Calls F-b8274406a146 with the copied-Mode storage address and zero, retaining its 32-bit result.
5. Forms a threshold by adding `0x3C` to that result modulo \(2^{32}\).
6. Calls F-dee541b2a059 with the threshold and the copied-Mode storage address.
7. Constructs the eight-byte packed record from the current copied-Mode and copied-Auxiliary-A contents.
8. Writes the retained replacement to the object’s association field at `0xC4`.
9. Calls F-b8274406a146 with the separate eight-byte temporary area.
10. Compares this second 32-bit extraction result against the threshold using unsigned ordering.

The replacement write occurs after packed-record construction but before the second extraction and either continuation. There is no explicit restoration of the old association.

The two visible calls to F-b8274406a146 have different explicit argument lists. A complete ABI-level explanation is not supplied.

### Packed-record construction and byte order

The construction combines numeric truncations with reads of portions of temporary storage. Those are distinct operations.

The first packed word is formed from:

| Numeric destination bits | Source |
|---|---|
| `15:0` | The two-byte portion of Auxiliary A storage beginning at byte offset `2`, interpreted as a 16-bit value |
| `23:16` | The low eight numeric bits of the current Auxiliary A word |
| `31:24` | The byte at offset `3` of current Mode storage |

The second packed word is formed from:

| Numeric destination bits | Source |
|---|---|
| `7:0` | The byte at offset `2` of current Mode storage |
| `15:8` | The byte at offset `1` of current Mode storage |
| `23:16` | The low eight numeric bits of the current Mode word |
| `31:24` | Zero |

Storage offsets are measured from the beginning of the corresponding temporary word. Mapping these offset-based reads to numeric significance requires the target’s byte order. The two-byte read likewise requires its storage interpretation.

**Under an explicit little-endian assumption only**, numbering numeric bytes from least significant byte `0` to most significant byte `3`, the construction becomes:

| Packed word | Numeric bytes, least to most significant |
|---|---|
| First | Auxiliary A byte `2`, Auxiliary A byte `3`, Auxiliary A byte `0`, Mode byte `3` |
| Second | Mode byte `2`, Mode byte `1`, Mode byte `0`, zero |

Under that assumption, Auxiliary A byte `1` is omitted. The unconditional statement supported by the construction is that Auxiliary A contributes its numeric low byte and its two-byte storage portion at offset `2`; which numeric bytes that portion represents depends on byte order.

Target endianness is not established by the supplied material. This uncertainty affects the numeric packed words themselves, not merely their later serialization.

### Continuation selection

If the second extraction result is unsigned-less-than the threshold, the function calls F-910df819391b with integer `1` and the packed-record address, then returns its result unchanged.

Otherwise, including equality, the function loads the object’s dispatch-table pointer, obtains the callable entry at offset `0xE8`, invokes it with the object and integer `2`, and returns its result unchanged.

The comparison is unsigned. Threshold formation wraps modulo \(2^{32}\); unbounded mathematical addition would produce different decisions near the top of the range.

### Zero replacement path

If replacement acquisition returns zero, the function:

1. Writes zero to the association field.
2. Calls F-f2b8332efb8a with `3`.
3. Returns zero.

The field is cleared before notification. It is not re-read or cleared again afterward. A notification dependency that changes the field can therefore change the object state observed at return.

## Side effects

### Explicit writes and calls

The only explicit persistent object-field write is to the association at `0xC4`: either the retained replacement or zero.

The function also initializes and modifies temporary storage, exposes temporary addresses, and invokes dependencies in the order specified above. Those dependencies may have additional effects not described by this body.

Repeated association reads matter: later operations can observe changes made by earlier dependencies. Conversely, the global value used in the five-call sequence is loaded once and reused; intervening changes to its global location do not refresh the retained argument.

No general preservation guarantee for object fields, globals, or exposed temporary storage follows from an unknown callee.

### Ownership and lifetime

The supplied body does not establish ownership of either association. It does not establish that F-42168b326728 releases an association or that F-289ec4311c64 destroys a temporary.

No explicit heap allocation or deallocation is visible in this function. That does not constrain allocation or release inside dependencies.

Temporary addresses are invocation-local. Whether a dependency copies their contents, retains their addresses, or requires cleanup is documented_only. No safe post-return use of those addresses is established.

### Errors and abnormal completion

There is no explicit exception handling, rollback, retry, or error translation.

A zero replacement result is a control-flow outcome, not proof of allocation failure or any other specific error. Behavior following invalid memory access, invalid dispatch, or a dependency that does not return normally is not established. In particular, the listed later operations are conditional on preceding operations completing normally.

## Dependencies

Only call-site roles are established.

| Opaque function ID | Visible role |
|---|---|
| F-583efa6014bd | Conditional sentinel/flags handling with object and zero |
| F-97df3ab481d5 | First call of the non-sentinel sequence |
| F-2b06c99f8427 | Receives retained global value and discriminator |
| F-0bcbb4e0e550 | Receives retained global value and `5` |
| F-44a900a27091 | Fourth call of that sequence |
| F-e0ed773ee385 | Receives retained global value, all-one word, and zero |
| F-b9acea4b06ad | Receives first temporary address and association-derived source address |
| F-398e0e9a318a | Receives second temporary address and opaque callback/address token |
| F-44362605ab79 | Supplies a value to the following decision call |
| F-4a6c0bf0740c | Returns the conditional-notification decision |
| F-289ec4311c64 | Operates on each temporary at separate points |
| F-3b9cee72cef2 | Receives recipient and first temporary address |
| F-42168b326728 | Receives object and freshly read association |
| F-fd8692810e8b | Returns replacement or zero |
| F-ebc906d2c2d5 | Receives nonzero replacement |
| F-dee541b2a059 | Twice receives a value and copied-Mode storage address |
| F-6132bda8e448 | Receives separate eight-byte temporary area |
| F-b8274406a146 | Supplies two 32-bit extraction results |
| F-910df819391b | Lower-comparison continuation |
| F-f2b8332efb8a | No-replacement notification with `3` |

The dynamic method and callback/address token lack supplied opaque function IDs. Their identities, effects, and contracts remain unknown.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Unless stated otherwise, dependencies return normally and do not introduce additional state changes.

1. **Absent association and absent replacement.**  
   An initial zero association skips existing-association processing. A zero replacement result causes an explicit zero write, notification through F-f2b8332efb8a with `3`, and a zero return.

2. **Nonzero Mode.**  
   With an existing association and nonzero original Mode, the zero-mode global load and discriminator/flags accesses are skipped. Temporary preparation, decision evaluation, association processing, and replacement acquisition still occur.

3. **Sentinel flags.**  
   With Mode zero and discriminator `−1`, flags `0x0C` or `0x8C` select F-583efa6014bd. Flags zero, `0x04`, or `0x08` do not. None of these sentinel cases selects the five-call sequence.

4. **Retained global argument.**  
   With discriminator `7`, the five calls occur in their specified order, and F-2b06c99f8427 receives `7`. Even if an earlier call changes the global location, later calls receive the originally loaded global value.

5. **Notification timing.**  
   A nonzero F-4a6c0bf0740c result causes recipient notification only after F-289ec4311c64 has operated on the second temporary. A zero result skips notification but not subsequent association processing.

6. **Unsigned boundary and equality.**  
   A first extraction of `0x1000` gives threshold `0x103C`. A second extraction of `0x103B` selects F-910df819391b; `0x103C` selects dynamic dispatch.

7. **Threshold wraparound.**  
   A first extraction of `0xFFFFFFF0` gives threshold `0x0000002C`. A second extraction of `0x20` selects F-910df819391b; `0x2C` selects dynamic dispatch.

8. **Packed values, explicitly conditional on little-endian storage.**  
   If the target is little-endian and the current Mode and Auxiliary A words are respectively `0x11223344` and `0xAABBCCDD`, the packed numeric words are `0x11DDAABB` and `0x00443322`. These results use current temporary contents, not necessarily the original inputs. Without the little-endian condition, these numeric results are not acceptance requirements.

9. **Changed association.**  
   If an earlier dependency changes the association, a later explicit field read uses the changed value. The entry association is not a preserved snapshot.

10. **Notification changes final state.**  
    On the zero-replacement path, if F-f2b8332efb8a changes the association after the explicit clear, the function still returns zero and does not overwrite that later change.

## Uncertainties

### Established static evidence

The supplied body establishes the explicit field offsets, branch conditions, dependency order, argument constants, temporary-storage accesses, association writes, modulo threshold arithmetic, unsigned comparison, and return propagation.

It establishes storage-offset reads in packed-record construction, but does not independently establish their mapping to numeric byte significance.

### Inferred intent

The temporary-preparation calls paired with later F-289ec4311c64 calls suggest scoped resource management. The discriminator sequence suggests state selection, and replacement installation followed by dispatch suggests a transition.

These interpretations are not requirements and do not establish ownership or cleanup.

### documented_only

Unresolved matters include:

- Target byte order and, consequently, unconditional numeric packed-word values.
- Application-level meanings of all fields and constants.
- Complete temporary-object layouts and dependency access extents.
- Dependency argument contracts beyond the visible call representation.
- Pointer retention, ownership, allocation, release, and cleanup obligations.
- Callback behavior and dynamic target identity.
- Synchronization, concurrent mutation, and reentrancy guarantees.
- The ABI explanation for the differing F-b8274406a146 argument lists.
- Dependency error behavior and abnormal-completion effects.

## Evidence

This contract is based solely on the supplied static function body and opaque dependency mapping. Embedded comments, the prior draft, and reviewer feedback are not treated as independent evidence of behavior.

No dependency implementations, authoritative target-endianness evidence, independent binary inspection, or execution traces were supplied. No runtime tests have been performed.
