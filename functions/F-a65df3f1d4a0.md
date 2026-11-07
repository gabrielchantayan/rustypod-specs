<!-- Trusted provenance: F-a65df3f1d4a0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a65df3f1d4a0

## Purpose

F-a65df3f1d4a0 emits a sequence through a callback selected from a mutable context. Its directly emitted sequence consists of:

1. An optional byte sequence traversed in increasing address order.
2. Zero or more padding bytes with value 0x30.
3. An optional byte sequence traversed in decreasing address order.

It also adjusts a context accounting field, conditionally invokes F-be81034a42a8 before and/or after the first sequence, updates an emission counter after each directly issued callback, and invokes F-ea83ae94b50b at the end of normal execution.

The arrangement is consistent with formatting reversed digits with a prefix and minimum-width padding. That interpretation is inferred intent, not an established restriction: the bytes are not validated as digits, neither sequence is required to be text, and no string terminator is consulted.

This contract describes static evidence only. No runtime tests have been performed.

## Interface

### Inputs

The recovered interface has five integer-valued arguments:

| Position | Contract name | Meaning |
|---|---|---|
| 1 | Context address | Address of mutable state and callback configuration. |
| 2 | Reverse-sequence address | Base address of bytes emitted from the last selected byte toward the first. |
| 3 | Reverse length | Signed length used both in accounting and to control reverse emission. |
| 4 | Forward-sequence address | Base address of bytes emitted from the first selected byte onward. |
| 5 | Forward length | Signed length used both in accounting and to control forward emission. |

The recovered representation uses 32-bit integer and address words, four-byte context fields, and one-byte sequence elements. Signed comparisons are used for the lengths, width, and padding count. The flags and callback-address masking use unsigned 32-bit operations.

Exact calling convention, pointer provenance, argument-register assignment, and callback argument-extension rules are not established.

### Context layout

Offsets are byte offsets from the context address.

| Offset | Width | Interpretation established by this function |
|---|---:|---|
| 0x18 | 32 bits | Flags. Bit 0x20 selects the stored width; bit 0x10 controls placement of F-be81034a42a8 calls. |
| 0x1C | 32 bits | Encoded callback address. Its lowest two bits are cleared before invocation. |
| 0x24 | 32 bits | Opaque value passed as the callback’s second argument. |
| 0x34 | 32 bits | Signed accounting field reduced before emission. |
| 0x38 | 32 bits | Signed width, read only when flag bit 0x20 is set. |
| 0x3C | 32 bits | Signed emission counter incremented after each direct callback returns. |

These are accessed fields, not a complete context definition. No significance is established for intervening bytes or other fields.

### Outputs

The function has no explicit return value. Its observable results are context writes, calls to dependencies, callback invocations, and any externally visible effects those callees produce.

There is no returned byte count or explicit success/failure status.

### Preconditions for defined, normal completion

The context must support the required reads and writes. Each sequence byte actually visited must be readable when visited. Every selected callback target must be callable under the applicable ABI, and all invoked functions must return normally for the complete sequence described below to occur.

A sequence address is not directly dereferenced when its corresponding length is nonpositive. This does not guarantee that dependencies ignore that address or related memory.

For the arithmetic descriptions below, signed operations and address calculations must remain representable and valid. Behavior outside those conditions is documented as unknown rather than assigned a wrapping interpretation.

## Behavior

### 1. Select and retain a width

The function first reads the flags at offset 0x18.

- If bit 0x20 is clear, the effective width is 1. The stored width is not read on this path, and this stage does not write the flags.
- If bit 0x20 is set, the function reads the signed width at offset 0x38. It then reads the flags for a read-modify-write operation and stores them with bit 0x10 cleared.

The explicit mask in the second case preserves every flag bit other than 0x10. In particular, it does not clear bit 0x20.

The effective width is retained for this invocation. Later changes to the stored width or bit 0x20 do not cause width selection to be repeated.

### 2. Compute padding and debit the accounting field

Let:

- \(W\) be the selected effective width;
- \(N\) be the original reverse length;
- \(P\) be the original forward length.

The padding count \(Z\) is:

\[
Z =
\begin{cases}
W-N, & N<W,\\
0, & N\geq W.
\end{cases}
\]

The comparison is signed. Subject to representable arithmetic, \(Z\) is nonnegative.

Before any helper or callback invocation, the function reads the field at offset 0x34 and writes back:

\[
\text{previous accounting value} - ((Z+N)+P).
\]

This is an accounting adjustment, not a count obtained from successful output. It occurs before emission and is not rolled back by any explicit operation in this function.

For nonnegative lengths, representable arithmetic gives:

\[
Z+N+P = \max(W,N)+P.
\]

The forward sequence does not reduce zero padding: padding depends only on width and reverse length. The forward length is added separately to the debit.

Negative lengths are not rejected. They can therefore make the accounting adjustment differ from the number of callbacks that the function directly issues.

### 3. Perform the first conditional helper call

After the accounting write, the function reads the current flags.

If bit 0x10 is clear, it invokes F-be81034a42a8 with the context address.

If bit 0x10 is set, it skips this call site.

This decision uses a fresh flag read. It is not a saved copy of the flags used for width selection.

### 4. Emit the forward sequence

For a positive forward length, the function visits exactly that many byte positions, starting at the forward-sequence base address and proceeding in increasing address order.

For each position:

1. It obtains the byte and the current callback configuration.
2. It clears the lowest two bits of the callback-address word.
3. It calls that target with the byte as the first argument and the current 32-bit value at offset 0x24 as the second argument.
4. After the callback returns, it reads the counter at offset 0x3C and writes that value plus one.

The callback-address word and opaque callback argument are obtained separately for every invocation. A callback or helper that changes either field can affect subsequent calls.

The supplied representation does not establish the relative evaluation order among the byte load, callback-address load, and opaque-argument load for a single invocation. All are inputs to that invocation; the counter update follows its return.

A zero or negative forward length produces no direct reads from the forward sequence and no callbacks in this phase.

### 5. Perform the second conditional helper call

After the forward phase, the function reads the flags again.

If bit 0x10 is set, it invokes F-be81034a42a8 with the context address. If that bit is clear, it skips this call site.

The two helper conditions are independent observations of mutable state. Consequently, F-be81034a42a8 can be called zero, one, or two times:

- Clear at both checks: only the first call site is taken.
- Set at both checks: only the second call site is taken.
- Clear at the first check, set at the second: both call sites are taken.
- Set at the first check, clear at the second: neither call site is taken.

These possibilities concern the observed values at the checks, regardless of which intervening operation changed them.

### 6. Emit padding

The function issues \(Z\) callbacks with first argument 0x30, provided arithmetic produced the representable padding count described above.

Each invocation independently reloads the callback-address word and opaque argument. After each callback returns, the function increments the current context counter, then reduces its retained padding count.

Changes to the stored width during earlier calls do not alter this phase’s retained count.

The byte value 0x30 conventionally represents the character “0” in ASCII-compatible encodings. The established behavior is the numeric byte value, not an encoding guarantee.

### 7. Emit the reverse sequence

For a positive original reverse length \(N\), the function reads bytes at offsets:

\[
N-1,\ N-2,\ \ldots,\ 0
\]

from the reverse-sequence address.

Each byte is submitted through the then-current callback target with the then-current opaque argument. After each callback returns, the context counter is read and incremented.

A zero or negative reverse length produces no direct reverse-sequence reads or callbacks. The original negative value may nevertheless already have affected padding and accounting.

The local remaining length decreases during emission. No length is written back to either input sequence.

### 8. Invoke the final dependency

After the reverse phase, the function invokes F-ea83ae94b50b with the context address, then returns.

This final call is reached even when every emission phase is empty, provided all earlier operations complete normally. It is not an exception handler or guaranteed cleanup action after a fault or nonreturning dependency.

## Side effects

### Direct state changes

The function directly performs:

- A conditional flags write clearing bit 0x10 when bit 0x20 selected the stored width.
- One accounting-field subtraction before all helper and callback calls.
- One counter increment after each directly issued callback returns.

With no intervening callee changes and no overflow, the direct counter increase is:

\[
\max(P,0)+Z+\max(N,0).
\]

This is not necessarily the final counter difference. Each increment operates on the value present after the callback returns, and dependencies may also modify the field.

### Reads are live, not snapshotted

The byte sequences are not copied before emission. Changes made to a byte before its turn can affect the emitted value, including changes made by earlier callbacks when memory aliases.

Similarly, callback configuration and helper-condition flags remain live across calls. The function does not restore them afterward.

### Ownership and lifetime

No explicit allocation, deallocation, ownership transfer, or sequence-buffer write appears in this function.

That observation does not establish the behavior of dependencies. In particular, this contract does not assume they preserve context fields, retain or release memory, avoid reentrancy, or leave sequence buffers unchanged. Subsequent accesses require their target memory to remain valid.

### Errors and interruption

There is no visible bounds checking, null checking, callback-target validation, error-status branch, or rollback.

Callback results, if any, are not inspected. The counter is incremented after a callback returns even though this function has no evidence that an external device or sink accepted the byte.

If a callback does not return, its following counter increment and the remainder of the function are not reached.

## Dependencies

| Dependency | Invocation contract established here | Not established |
|---|---|---|
| F-be81034a42a8 | Receives the context address at either or both flag-controlled call sites. The first is before forward emission; the second is afterward and before padding. | Output, ownership effects, field preservation, error behavior, and the significance of bit 0x10. |
| F-ea83ae94b50b | Receives the context address after all direct emission phases complete normally. | Whether it flushes, pads, finalizes, releases resources, changes counters, or reports errors elsewhere. |
| Dynamically selected callback | Target comes from the current word at offset 0x1C with its lowest two bits cleared. Receives one emitted byte/value and the current word at offset 0x24. | Semantic identity, argument signedness or extension, return convention, preservation guarantees, and ownership behavior. |

Helper-generated output, if any, is additional to the direct callback sequence specified here.

## Acceptance examples

These are conditional acceptance examples derived from static evidence, not executed tests. Unless stated otherwise, assume valid memory, representable arithmetic, returning callees, and no callee changes to fields or buffers.

### Default width with a nonempty reverse sequence

With bit 0x20 clear, reverse bytes `[A, B, C]`, reverse length 3, and forward bytes `[X, Y]` of length 2:

- Effective width is 1 and padding is zero.
- Accounting decreases by 5.
- Direct callback values are `X, Y, C, B, A`.
- The direct counter increase is 5.
- Bit 0x10 determines which one of the two helper call sites is taken.

### Stored width forces the early helper placement

With bit 0x20 set, stored width 5, reverse bytes `[7, 2]` of length 2, and a one-byte forward sequence containing `−`:

- Bit 0x10 is cleared during width selection.
- Padding is 3.
- Accounting decreases by 6.
- F-be81034a42a8 is called before the forward sequence.
- Direct values are `−, 0x30, 0x30, 0x30, 2, 7`.
- The later F-be81034a42a8 call is skipped under the no-mutation assumption.

The illustrative symbols denote supplied byte values; they impose no text-encoding contract.

### Empty sequences still allow default padding

With bit 0x20 clear and both lengths zero:

- Width is 1 and padding is 1.
- Accounting decreases by 1.
- One direct callback receives 0x30.
- F-ea83ae94b50b is still invoked.

### Empty sequences with zero stored width

With bit 0x20 set, stored width zero, and both lengths zero:

- Bit 0x10 is cleared.
- Accounting is written with an unchanged numeric value.
- F-be81034a42a8 is called at the first site.
- No direct callbacks occur.
- F-ea83ae94b50b is invoked.

### Negative reverse length

With bit 0x20 clear, reverse length −2, and forward length zero:

- Width is 1 and padding is 3.
- Accounting decreases by 1.
- Three direct callbacks receive 0x30.
- No reverse-sequence byte is read.

Thus the accounting debit is not a reliable direct-emission count for negative lengths.

### Negative forward length

With default width, reverse length 2, and forward length −3:

- Padding is zero.
- The debit is −1, so the accounting value increases by 1.
- No forward byte is read.
- Two reverse bytes are emitted.

### Mutation changes helper placement

If bit 0x10 is clear at the first check and F-be81034a42a8 sets it before returning, both helper call sites are taken, provided the forward phase leaves it set.

Conversely, if the first check sees it set and a forward callback clears it, neither helper call site is taken.

### Callback changes the counter

If a callback returns after setting the counter to 100, the immediately following direct increment writes 101, not the pre-call counter plus one.

## Uncertainties

- **documented_only — Arithmetic extremes:** Signed overflow behavior is not established. Width minus length, the debit additions, accounting subtraction, and counter increments can exceed signed 32-bit range. No saturation or error path is visible.
- **documented_only — Address arithmetic:** Behavior for invalid addresses, out-of-range address additions, or inaccessible bytes is not established.
- **documented_only — ABI details:** Exact integer-to-pointer representation, alignment requirements, byte extension at callback boundaries, and the reason for clearing two callback-address bits are unknown.
- **documented_only — Dependency semantics:** No preservation, ownership, output, or error guarantees are available for either named dependency or the callback.
- **documented_only — Concurrency:** No synchronization contract is supplied. Fresh reads show sensitivity to intervening calls; they do not establish safe concurrent access.
- **Inferred intent:** Width-controlled numeric formatting is plausible. It does not justify rejecting negative lengths, treating inputs as strings, or assigning undocumented meanings to context fields.

## Evidence

The supplied static function representation establishes the field offsets, masks, signed comparisons, arithmetic structure, forward and reverse traversal directions, callback-address masking, post-callback counter updates, two independent helper checks, and final dependency call.

No dependency bodies, public type definitions, callers, target ABI specification, or runtime observations were supplied. The contract therefore distinguishes direct behavior from inferred formatting intent and leaves unsupported dependency and overflow behavior documented rather than assumed.
