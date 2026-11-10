<!-- Trusted provenance: F-b1d73aae29eb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b1d73aae29eb

## Purpose

F-b1d73aae29eb interprets an encoded 32-bit value and a 32-bit selector, updates selection-related fields in a caller-provided context, and conditionally invokes dependent operations.

The established behavior comprises:

- Rejecting encoded values outside `0x40`–`0x7F`.
- Deriving a signed selection index from the selector and encoded-value range.
- Preparing either a local 32-bit payload or an indexed offset.
- Applying a lower bound determined by a context mode byte.
- Updating selection fields and issuing conditional notifications.
- Clearing a context byte.
- Conditionally invoking a final operation with locally held flags and an optional payload pointer.

The payload constants resemble packed colors, but that is **inferred intent**, not an established semantic requirement. Their numeric bit patterns are specified without assigning channel meanings.

## Interface

The function has four inputs and a 32-bit return value.

| Input | Established representation and use |
|---|---|
| Context | A 32-bit address-like value serving as the base of mutable storage. |
| Selector | A 32-bit value matched against specific constants without prior narrowing. |
| Encoded value | An unsigned 32-bit value; accepted only within `0x40`–`0x7F`, inclusive. |
| Auxiliary argument | An opaque 32-bit value forwarded to F-35c2e4297234 when that dependency is called. |

The return value is `0` for a rejected encoded value and `1` when accepted processing reaches its normal end. Dependency results are not examined. Returning `1` therefore indicates completion of the local path, not verified success of dependent operations.

The context has these established accesses:

| Offset | Width | Observed role |
|---|---:|---|
| `0xA1` | 8 bits | Mode used for index constraints and a notification predicate. |
| `0xB0` | 32 bits | Final-operation guard and, through a separate read, its handle argument. |
| `0xB4` | 32 bits | Indexed offset or zero for local-payload mode. |
| `0xB8` | 32 bits | Current selection index. |
| `0xD4` | 32 bits | Secondary index synchronized under specific conditions. |
| `0xD9` | 8 bits | Cleared late in accepted processing. |

These offsets describe accessed fields, not the complete object layout.

For accepted inputs, storage must permit the reads and writes actually taken by the selected path. No local null, bounds, alignment, lifetime, or access-permission checks are established. Dependencies may impose additional preconditions that are unknown here.

Rejected inputs return before any established context access, direct global read, or dependency call.

## Behavior

### Validation and selection derivation

Validation subtracts `0x40` using unsigned 32-bit arithmetic and accepts only results no greater than `0x3F`. Thus, the accepted set is exactly `0x40` through `0x7F`. Values below the interval wrap during subtraction and are rejected.

For accepted inputs, the selector and the encoded-value range determine the following **unconstrained index**:

| Selector | Encoded value `0x40`–`0x5F` | Encoded value `0x60`–`0x7F` |
|---|---:|---:|
| `0x10`, `0x18` | 10 | 11 |
| `0x11`, `0x19` | 0 | 1 |
| `0x12`, `0x1A` | 2 | 3 |
| `0x13`, `0x1B` | 11 | 12 |
| `0x14`, `0x1C` | 13 | 14 |
| `0x15`, `0x1D` | 4 | 5 |
| `0x16`, `0x1E` | 6 | 7 |
| `0x17`, `0x1F` | 8 | 9 |
| Any other 32-bit value | −1 | 0 |

Unknown selectors are not rejected. Index −1 has the stored 32-bit representation `0xFFFFFFFF`.

A local 32-bit flags word initially contains `4` if the encoded value is odd, or `0` otherwise. A special local-payload case later sets bit 1.

The index range before context constraints is −1 through 14. Its derivation does not require signed overflow.

### Payload mode and dependency input

Bits 1–3 of the encoded value form a subindex from 0 through 7. Bit 0 affects the initial flags but not this subindex.

| Encoded-value range | Processing mode |
|---|---|
| `0x40`–`0x4F`, `0x60`–`0x6F` | Local-payload mode |
| `0x50`–`0x5F`, `0x70`–`0x7F` | Indexed mode |

A local 32-bit payload initially holds the entire encoded value.

Whenever F-35c2e4297234 is invoked, its six arguments are:

1. The address of the local payload.
2. A value obtained by directly reading a global storage location.
3. The encoded value.
4. The auxiliary argument.
5. The context.
6. The selector.

The global read is an access performed by this function to supply the call argument; it is not merely an unknown internal effect of the dependency. It occurs on every path that executes this call. Fixed-payload paths do not perform that direct global read.

The global’s value, type, access width, and meaning are not established by the supplied representation. Its evaluation order relative to the other call arguments is also not established.

F-35c2e4297234 receives the address of an initialized 32-bit local object and may modify it. Its return value is ignored. Its effects on the context or other state are unknown.

### Indexed mode

Every indexed-mode input invokes F-35c2e4297234.

After that call returns, the function writes four times the subindex to context offset `0xB4`, using a 32-bit write. The possible values are `0`, `4`, `8`, `12`, `16`, `20`, `24`, and `28`; this multiplication does not overflow.

The final operation, if reached, receives a null payload pointer in this mode. The earlier payload address passed to F-35c2e4297234 does not change that decision.

### Local-payload mode

The subindex determines the local payload and any additional flag:

| Subindex | Payload behavior | Additional flag effect |
|---:|---|---|
| 0 | Invoke F-35c2e4297234; use the payload contents present after its return. | None |
| 1 | Set payload to `0xFF43FF50`. | None |
| 2 | Set payload to `0xFFFF9343`. | None |
| 3 | Set payload to `0xFFFFFD43`. | None |
| 4 | Set payload to `0xFF3030FF`. | None |
| 5 | Set payload to `0xFF33F8FF`. | None |
| 6 | Set payload to `0xFFA83EF9`. | None |
| 7 | Invoke F-35c2e4297234; use the payload contents present after its return. | Retain the flags’ low byte and set bit 1. |

For subindices 0 and 7, no particular payload transformation is guaranteed. If the dependency leaves the payload unchanged, it still contains the encoded value.

For subindex 7, the established local flags become `2` for an even encoded value or `6` for an odd one.

After payload preparation, the function writes zero to context offset `0xB4`, using a 32-bit write. It retains a local indication that the final operation should receive the payload address.

In both processing modes, the function’s `0xB4` write occurs after any F-35c2e4297234 call. Any value that dependency wrote to `0xB4` is therefore overwritten at this stage.

### Mode constraint

After writing `0xB4`, the function reads the byte at `0xA1` as an unsigned value.

Only modes 2, 3, and 4 impose a minimum:

| Saved mode | Minimum index |
|---:|---:|
| 2 | 1 |
| 3 | 2 |
| 4 | 3 |

If the unconstrained index is below the applicable minimum, the selected index becomes that minimum. Other byte values leave it unchanged.

The mode-range test uses unsigned arithmetic; its subtraction admits exactly 2, 3, and 4. Within that set, the subsequent signed comparison implements the stated lower bound without signed overflow.

This mode read follows F-35c2e4297234, if called. The constraint therefore uses the context contents after that dependency, not necessarily the entry mode.

### Selection changes and notifications

The function reads the 32-bit field at `0xB8` and compares it with the selected index.

If equal:

- No write to `0xB8` occurs.
- `0xD4` is not read or synchronized.
- Neither F-1ae7559ffffd nor F-681d5ec70de3 is called.

If different:

1. The selected index is written to `0xB8`.
2. If the saved mode is 2, 3, or 4, `0xD4` is read.
3. If that secondary index differs from the selected index, the selected index is written to `0xD4`, then F-1ae7559ffffd is invoked with the context.
4. F-681d5ec70de3 is invoked with:
   - The context.
   - The low eight bits of a fresh 32-bit read from `0xB8`.
   - A Boolean indicating whether a fresh byte read from `0xA1` differs from 5.

The synchronization decision uses the saved mode. The later notification uses fresh context reads after any F-1ae7559ffffd call. Changes made by that dependency can therefore affect the notification arguments.

The representation does not establish an evaluation order between the two context reads supplying F-681d5ec70de3’s arguments.

### Final operation and completion

After selection-change processing, the function clears the byte at `0xD9`.

It then performs a 32-bit read of `0xB0` to decide whether to enter the final-call branch:

- If this **guard read** is zero, the final dependency is skipped.
- If it is nonzero, the function prepares and invokes F-fc9444c74c1d.

The final call has six arguments:

1. The result of a **separate, fresh 32-bit read** from `0xB0`.
2. The low eight bits of a fresh 32-bit read from `0xB4`.
3. Zero.
4. The address of the local 32-bit flags word.
5. The local payload address in local-payload mode, or a null pointer in indexed mode.
6. Zero.

The handle argument is not a retained result of the guard read. The representation also does not establish the evaluation order between the handle-argument read from `0xB0` and the offset-argument read from `0xB4`.

No dependency call is shown between the guard read and the final call. Nevertheless, the two reads must remain distinct in this contract: the guard determines branch entry, while the later read supplies the argument. There is no second nonzero check on that argument.

Both argument fields are read after earlier notifications. They need not contain the values present on entry or immediately after this function’s own writes. The locally retained payload-presence decision does not depend on those late field contents.

After the final dependency returns, or after it is skipped, the function returns `1`. No subsequent direct context write is established.

## Side effects

The direct context writes occur in this order, where their conditions apply:

1. A 32-bit write to `0xB4`.
2. A conditional 32-bit write to `0xB8`.
3. A conditional 32-bit write to `0xD4`.
4. An 8-bit zero write to `0xD9`.

Dependency calls interleave with these writes:

- F-35c2e4297234 precedes the `0xB4` write and saved-mode read.
- F-1ae7559ffffd follows the conditional `0xD4` write.
- F-681d5ec70de3 follows selection-change processing and precedes the `0xD9` clear.
- F-fc9444c74c1d follows the clear and the nonzero handle guard.

A direct global read also occurs whenever F-35c2e4297234 is called, supplying its second argument. No other direct global access is identified by the supplied representation. Dependencies may have additional unknown global effects.

No direct allocation, release, or ownership transfer is established.

The payload and flags are function-local 32-bit objects. Passing their addresses does not establish that dependencies copy their contents, avoid retaining their addresses, preserve their values, or take ownership. This function provides no transferable allocation and does not extend either object’s lifetime beyond its invocation.

No rollback, local error recovery, or dependency-error propagation is established. A dependency that does not return normally can prevent later writes, calls, and the normal return.

## Dependencies

| Opaque function ID | Established interaction | Unresolved contract |
|---|---|---|
| F-35c2e4297234 | Called once on every indexed-mode path and on local-payload subindices 0 and 7. Receives the payload address, a directly read global value, and four further inputs. Result ignored. | Payload transformation, global-value interpretation, context/global effects, ownership, retention, and failure behavior. |
| F-1ae7559ffffd | Receives the context after both index fields change under saved mode 2–4. | Internal effects, including mutations visible to later fresh reads. |
| F-681d5ec70de3 | Called whenever `0xB8` changes. Receives context, a freshly read index’s low byte, and a freshly read mode-not-5 predicate. | Notification semantics, mutations, ownership effects, and failure behavior. |
| F-fc9444c74c1d | Called after a nonzero `0xB0` guard read. Receives a separately read handle, low-byte offset, zeros, flags address, and optional payload address. | Handle semantics, payload/flags interpretation, retention, ownership, external effects, and failure behavior. |

No dependency is assumed to preserve context fields or globals, or to own or release memory.

## Acceptance examples

These are conditional static examples, not executed tests. Unless otherwise specified, dependencies return normally and do not mutate context fields.

1. **Rejected values.** Encoded values `0x3F`, `0x80`, and `0xFFFFFFFF` return `0` without context accesses, the direct global read, or dependency calls.

2. **Accepted boundaries.** Values `0x40` and `0x7F` enter accepted processing and return `1` if all required accesses and calls complete normally.

3. **Fixed payload without a direct global read.** With selector `0x11`, encoded value `0x42`, mode 0, current index 7, and zero final guard:
   - Payload becomes `0xFF43FF50`; flags are zero.
   - F-35c2e4297234 is not called, and its global argument is not read.
   - `0xB4` becomes zero and `0xB8` becomes zero.
   - `0xD4` is not inspected.
   - F-681d5ec70de3 receives index byte 0 and a true predicate.
   - `0xD9` is cleared; no final call occurs.

4. **Constraint and synchronization.** With an unrecognized selector, encoded value `0x62`, mode 4, current index 9, and secondary index 8, the unconstrained index 0 is raised to 3. Both index fields become 3. F-1ae7559ffffd precedes F-681d5ec70de3.

5. **No repair when the current index matches.** With selector `0x11`, encoded value `0x42`, mode 2, current index 1, and secondary index 99, the selected index is 1. Neither notification runs, and `0xD4` is not read or changed.

6. **Unknown selector and truncation.** With an unrecognized selector, encoded value `0x42`, mode 5, and current index other than `0xFFFFFFFF`, the function stores `0xFFFFFFFF` at `0xB8`. F-681d5ec70de3 receives index byte 255 and a false predicate.

7. **Indexed mode.** With encoded value `0x55`, the function directly reads the global argument and calls F-35c2e4297234, then writes 8 to `0xB4`. Flags are 4. If the final guard is nonzero and the offset remains unchanged, the final call receives offset 8 and a null payload pointer.

8. **Special local payload.** With encoded value `0x4F`, the global read and F-35c2e4297234 call occur. Flags become 6 and `0xB4` becomes zero. A final call, if entered, receives a non-null payload pointer; the payload contents depend on F-35c2e4297234.

9. **Fresh notification arguments.** If F-1ae7559ffffd changes `0xB8` to `0x1234` and `0xA1` to 5, and those values remain stable through argument evaluation, F-681d5ec70de3 receives index byte `0x34` and a false predicate.

10. **Distinct handle reads.** A zero guard read skips the final call without a handle-argument read. A nonzero guard read enters the branch, where `0xB0` is read again. If the argument read yields a different value, that later value is passed without another nonzero test. This specifies the read distinction, not a guarantee of concurrent-mutation safety.

## Uncertainties

The following remain **documented_only**:

- Domain meanings of the selector, indices, mode, flags, handle, and payload.
- Complete context layout, storage requirements beyond observed accesses, and concurrency rules.
- The global argument’s value, type, width, meaning, and evaluation order relative to other arguments.
- Evaluation order between the fresh context reads within each notification or final call expression.
- Dependency internals, global effects, error signaling, ownership, and pointer retention.
- Whether payload patterns represent colors and what external byte-order convention applies.
- Behavior after invalid memory access, dependency non-return, or concurrent mutation.

## Evidence

This contract uses only the supplied static function representation and opaque dependency mapping. The prior draft and reviewer feedback are not independent behavioral evidence.

Established static evidence includes the unsigned range gate, exact index mapping, payload patterns, field widths and offsets, conditional writes and calls, direct global argument read, saved versus fresh mode values, distinct final guard and handle-argument reads, truncations, and normal return values.

Descriptive field roles and possible packed-color semantics are inferred intent. Unspecified dependency behavior and evaluation order remain unresolved.

No runtime tests have been performed.
