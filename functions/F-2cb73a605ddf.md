<!-- Trusted provenance: F-2cb73a605ddf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2cb73a605ddf

## Purpose

F-2cb73a605ddf conditionally initializes a caller-supplied context using a descriptor, callback operations, and metadata obtained through those operations. It returns a 32-bit status-like value.

The routine has three broad outcomes:

- Reject required arguments or an absent initialization context.
- Return without initialization because the descriptor already contains a nonzero association or meets a size-related shortcut.
- Publish the context in the descriptor, obtain and process metadata, and either complete through a final dependency or clear the descriptor association and invoke that same dependency on an unwind path.

The initialization, metadata interpretation, and unwind roles are inferred intent. The observable behavior is the reads, writes, calls, branches, and return selection described below. In particular, the final dependency is not established to be a resource releaser, and the routine does not provide an unconditional rollback guarantee.

## Interface

### Arguments

The interface has seven positional arguments:

| Position | Contract-facing name | Representation and role |
|---|---|---|
| 1 | Descriptor | Pointer to a record containing at least six 32-bit words for the accesses described below. |
| 2 | Context | 32-bit address of writable caller-supplied storage. A null value is rejected only on the initialization path. |
| 3 | Auxiliary value | Opaque 32-bit value stored in the context during initialization. |
| 4 | Acquisition callback | Required nonnull callable address. Called with three scalar arguments and a pointer to local working storage. |
| 5 | Transfer callback | Required nonnull callable address. Called on one metadata-dependent path. |
| 6 | Required value | Opaque 32-bit value. Must be nonzero and is stored in the context. |
| 7 | Configuration callback | Required nonnull callable address. Called initially and, on one path, again using metadata-derived arguments. |

These names describe use, not an established external API.

A nonnull descriptor, both nonnull operation callbacks, a nonzero required value, and a nonnull configuration callback are mandatory even when no initialization would otherwise be necessary.

### Descriptor layout

Offsets are in bytes from the descriptor address.

| Offset | Width | Observed use |
|---|---:|---|
| +0 | 32 bits | Association/state word; tested for zero, assigned the context address, and cleared on the unwind path. |
| +4 | 32 bits | Value copied into context +4 before callbacks begin. |
| +16 | 32 bits | Compared as unsigned against 512 on the size-related shortcut. |
| +20 | 32 bits | Tested for zero before that unsigned comparison. |

No other descriptor fields are directly accessed.

### Context layout

Initialization writes the following fields. The context must support accesses through byte offset +75.

| Offset | Width | Value or use |
|---|---:|---|
| +1 | 8 bits | Copied from metadata +50. |
| +2 | 16 bits | Read through the metadata-derived pointer passed to F-467db564e65a. |
| +4 | 32 bits | Initially copied from descriptor +4; subsequently reread for callback arguments. |
| +8 | 32 bits | Auxiliary argument. |
| +12, +16, +20, +24 | 32 bits each | Results of F-4055be9c3f18 at metadata offsets +16, +20, +24, and +28. |
| +28 | 16 bits | Metadata +32, also captured as a signed 16-bit branch/configuration value. |
| +30 | 16 bits | Metadata +34, also captured as an unsigned 16-bit threshold value. |
| +32, +36 | 32 bits each | Results of F-4055be9c3f18 at metadata offsets +36 and +40. |
| +42 | 16 bits | Constant 1. |
| +44 | 32 bits | Constant 0. |
| +48 | 32 bits | Result of F-4055be9c3f18 at metadata +52, optionally ORed with 6. |
| +52 | 32 bits | Constant 1. |
| +60 | 32 bits | Acquisition callback address. |
| +64 | 32 bits | Transfer callback address. |
| +68 | 32 bits | Required value. |
| +72 | 32 bits | Constant 1. |

The routine does not zero the context as a whole. Fields and gaps not listed are not directly initialized.

### Return representation

The return has a pointer-shaped 32-bit representation but is used as a status:

- `0` on the explicit success/no-work paths.
- `0xFFFFFFCE`, equivalent to signed −50, for missing mandatory arguments.
- `0xFFFFFF94`, equivalent to signed −108, when initialization requires a context and the context is null.
- A value loaded from a global slot on the size-related shortcut.
- A nonzero callback or dependency result on the corresponding failure path.

These returned bit patterns must not be treated as dereferenceable pointers merely because of the recovered return representation.

## Behavior

### 1. Validate mandatory arguments

The routine checks the descriptor, acquisition callback, transfer callback, required value, and configuration callback for zero.

If any is zero, it returns `0xFFFFFFCE`. It does not dereference the descriptor or context and does not invoke a dependency on this path.

The context and auxiliary value are not part of this initial validation.

### 2. Decide whether initialization is needed

The descriptor association at +0 is read.

If it is nonzero, the routine returns zero without validating the context, examining the size-related descriptor fields, or invoking callbacks. This is an association test, not a validation of the object referenced by that association.

If the association is zero:

1. Read descriptor +20.
2. If that value is zero, read descriptor +16 and compare it as an unsigned 32-bit value against 512.
3. If both conditions hold, return the current value from a global slot.

The shortcut precedes context validation. Consequently, a null context can produce the global-slot result rather than −108. The global-slot value is not established by the supplied evidence.

Unsigned comparison matters: a descriptor +16 bit pattern of `0xFFFFFFFF` is not less than 512.

If the shortcut does not apply and the context is zero, return `0xFFFFFF94`.

### 3. Publish the context and perform the initial configuration

For an eligible initialization, writes occur in this order:

1. Store the acquisition callback at context +60.
2. Store the transfer callback at context +64.
3. Store the required value at context +68.
4. Store the auxiliary value at context +8.
5. Set the first word of local working storage to zero.
6. Set a separate local 32-bit value to 512.
7. Copy descriptor +4 into context +4.
8. Store the context address into descriptor +0.

The routine then calls the configuration callback with:

- The value read from context +4.
- The 32-bit value 512.
- The 32-bit value 1.

The association is therefore visible before this first callback. A callback that can inspect the descriptor can observe the context during partial initialization.

If this callback returns nonzero, skip metadata acquisition and enter the unwind sequence, preserving this result for return.

### 4. Acquire metadata

If initial configuration returns zero, invoke the acquisition callback with:

- The then-current value at context +4.
- Zero.
- Zero.
- A pointer to local working storage.

The first working word is read immediately after the call and captured as a metadata base value.

If acquisition returns nonzero, the routine explicitly sets the first and second working words to zero, then enters the unwind sequence with the acquisition result preserved.

If acquisition returns zero, pass the descriptor and an address 14 bytes beyond the captured metadata base to F-467db564e65a.

If F-467db564e65a returns nonzero, enter the unwind sequence with that result preserved.

The routine does not itself check the acquired metadata pointer for null, range, alignment, or length. Successful callbacks must provide storage suitable for the later accesses.

### 5. Populate metadata-derived context fields

After F-467db564e65a returns zero, the routine performs these operations in order:

1. Read 16 bits through the metadata-derived address passed to F-467db564e65a, and store them at context +2.
2. Invoke F-4055be9c3f18 at metadata +16; store its 32-bit result at context +12.
3. Invoke it at metadata +20; store its result at context +16.
4. Invoke it at metadata +24; store its result at context +20.
5. Invoke it at metadata +28; store its result at context +24.
6. Read metadata +32 as a signed 16-bit value, retain it for later branching, and store its bits at context +28.
7. Read metadata +34 as an unsigned 16-bit value, retain it for the threshold test, and store it at context +30.
8. Invoke F-4055be9c3f18 at metadata +36; store its result at context +32.
9. Invoke it at metadata +40; store its result at context +36.
10. Copy one byte from metadata +50 to context +1.
11. Invoke F-4055be9c3f18 at metadata +52; store its result at context +48.
12. If the captured unsigned value from metadata +34 exceeds 40, rewrite context +48 with that returned result ORed with `0x00000006`.
13. Store 1 at context +42 as 16 bits.
14. Store 1 at context +52 as 32 bits.
15. Store 0 at context +44 as 32 bits.
16. Store 1 at context +72 as 32 bits.

F-4055be9c3f18 results are stored, not tested as status codes.

The retained signed and unsigned 16-bit values govern later decisions even if an intervening dependency changes the corresponding context fields or metadata. Conversely, subsequent explicit memory reads can observe dependency mutations. There is no established guarantee that these dependencies preserve shared storage.

### 6. Select the metadata-dependent processing path

#### Captured signed value equals 512

Call F-850578bcb4fe with the context and the current first word of working storage.

If it returns zero, proceed to finalization.

If it returns nonzero:

1. Read context +4.
2. Read a 32-bit pointer at offset +4 from the object addressed by that value.
3. Read the 16-bit word addressed by that pointer.
4. Write that word back with bit 0 set.
5. Enter the unwind sequence, preserving the F-850578bcb4fe result.

The nested pointer accesses are unchecked.

#### Captured signed value does not equal 512

Call the configuration callback with:

- The then-current context +4 value.
- The captured signed 16-bit value, promoted with signed meaning.
- The 32-bit value 32.

If it returns nonzero, enter unwind.

Otherwise, call the transfer callback with the current context +4 value, the working-storage pointer, and the 32-bit value 4. A nonzero result enters unwind.

Otherwise, call F-5f9a16493218 with the context, zero, and the working-storage pointer. A nonzero result enters unwind.

Only if all three calls return zero does execution reach finalization.

A metadata value with its high bit set is negative for the second configuration call; it is not implicitly an unsigned value in the range 32768–65535.

### 7. Finalization and unwind

Finalization calls F-b0f922796692 with the context and the working-storage pointer.

If it returns zero, the routine returns zero immediately. It does not itself clear descriptor +0 on this path.

If finalization returns nonzero, its result is preserved and execution enters unwind.

Unwind always:

1. Writes zero to descriptor +0.
2. Calls F-b0f922796692 with the context and the same working-storage pointer.
3. Returns the previously selected nonzero result, ignoring the unwind call’s result.

Thus a nonzero first finalization result causes F-b0f922796692 to be called twice. The second call receives working storage as left by the first call; the routine does not reset it.

Clearing the association occurs before the unwind dependency. Since that dependency is not established to preserve the descriptor, the final association value cannot be guaranteed solely from the clear performed by this routine.

## Side effects

- The descriptor association is published before callbacks execute and cleared before the unwind dependency executes.
- Context writes are incremental. A failed initialization can leave callback addresses, the auxiliary value, metadata-derived fields, and constants in caller storage.
- No direct context-wide rollback or clearing occurs.
- On one failure path, a separately referenced 16-bit word has bit 0 set.
- The routine directly reads one global slot on the size-related shortcut.
- All external calls may have additional effects not established here.

No direct allocation or deallocation operation is visible. Passing metadata and working-storage pointers to dependencies does not establish ownership transfer, retention permission, or release behavior.

The local working area contains three 32-bit words, but only the first is initialized before the first callback. The second is explicitly zeroed only after acquisition failure; no explicit initialization of the third is shown. A separate local word is assigned 512, but whether it belongs to a larger ABI-level working record is unresolved. Consumers must not assume all working storage is initially zero.

## Dependencies

| ID or callback | Established interaction |
|---|---|
| Configuration callback | Called initially with the handle-like context field, 512, and 1; conditionally called again with a signed metadata value and 32. Zero permits progress. |
| Acquisition callback | Receives the handle-like value, two zeros, and writable working storage. Zero permits metadata processing. |
| Transfer callback | Receives the handle-like value, working storage, and 4. Called only on the non-512 path. |
| F-467db564e65a | Receives the descriptor and metadata base +14. Zero permits field population. |
| F-4055be9c3f18 | Called seven times with metadata-relative addresses. Each 32-bit result is stored directly. Conversion semantics are unknown. |
| F-850578bcb4fe | Called only when the captured signed metadata field equals 512. Its nonzero result triggers the nested status-bit write. |
| F-5f9a16493218 | Called after successful secondary configuration and transfer. Zero permits finalization. |
| F-b0f922796692 | Called for finalization and unwind. A failed finalization causes a second call; unwind-call results are ignored. |

No dependency is assumed to preserve globals, metadata, context fields, descriptor fields, or working storage.

## Acceptance examples

These are conditional static acceptance criteria, not runtime test results.

1. **Required argument missing:** A null acquisition callback yields `0xFFFFFFCE`, even if the descriptor association is nonzero. No dependency is called.

2. **Existing association:** With mandatory arguments valid and descriptor +0 nonzero, return zero. A null context does not change this outcome.

3. **Shortcut boundary:** With association zero and descriptor +20 zero, an unsigned descriptor +16 value of 511 selects the global-slot return. A value of 512 does not.

4. **Missing context after shortcut rejection:** With association zero, descriptor +20 nonzero, and context null, return `0xFFFFFF94`.

5. **Initial configuration failure:** If initial configuration returns a nonzero value E, clear descriptor +0, call F-b0f922796692 once, and return E regardless of that unwind call’s return.

6. **Acquisition failure:** If acquisition returns E, explicitly zero the first two working words before unwind. Metadata validation and population do not occur.

7. **Threshold boundary:** A captured unsigned metadata value of 40 does not cause the OR-with-6 rewrite. A value of 41 does. Other bits in the conversion result are preserved.

8. **512-path failure:** If the captured signed field is 512 and F-850578bcb4fe returns E, set bit 0 in the nested 16-bit status word before clearing the descriptor association and unwinding.

9. **Successful non-512 path:** With configuration, transfer, F-5f9a16493218, and finalization all returning zero, return zero without a direct association clear.

10. **Failed finalization:** If the first F-b0f922796692 call returns E, clear the descriptor association and call it again. Return E, not the second result.

## Uncertainties

The following remain **documented_only**:

- The global-slot return value and its intended meaning.
- External names, semantic types, and complete sizes of the records.
- Whether F-4055be9c3f18 performs byte-order conversion, decoding, or another transformation.
- Callback calling-convention details beyond the visible argument values and status tests.
- Ownership, lifetime, retention, and release rules for acquired metadata and working storage.
- The full working-record layout and the relevance of the separately assigned 512 value.
- Dependency effects on globals, shared objects, and aliases.
- Validity requirements for direct 16-bit accesses, including alignment and platform byte order.
- Behavior for invalid addresses, insufficient storage, concurrent mutation, or unsupported aliasing.

The visible address arithmetic uses a 32-bit address model and fixed byte displacements. No overflow, bounds, or pointer-chain validation is present. Successful use requires all reached accesses to be valid; no recovery behavior for memory faults is established.

## Evidence

Established static evidence consists of the supplied function body’s argument checks, fixed-width accesses, branch conditions, call arguments, store ordering, and return selection.

“Initialization,” “metadata,” “configuration,” “transfer,” “finalization,” and “unwind” are contract-facing descriptions of observable roles, not independently established domain semantics.

No runtime tests have been performed. The acceptance examples express consequences of the static control flow under their stated conditions, not observed executions.
