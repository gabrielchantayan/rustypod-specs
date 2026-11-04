<!-- Trusted provenance: F-ad43d78eb6b0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ad43d78eb6b0

## Purpose

F-ad43d78eb6b0 constructs a fixed 13-byte record in caller-provided storage. The record combines fixed marker bytes with values obtained through three indirect calls.

The third input controls only the returned integer. It does **not** limit the number of bytes written: every normal completion writes bytes at offsets 0 through 12, even when the third input is zero.

The record’s protocol meaning is unknown. Describing it as a header, message, or device command would be inferred intent rather than established behavior.

## Interface

### Inputs

The function receives three inputs:

1. **Dispatch root.** A pointer to storage whose first 32-bit word supplies an object address. The first 32-bit word at that object address supplies a dispatch-table address. Three indirect call targets are obtained from that table.
2. **Destination.** A pointer to writable byte-addressed storage. Normal completion requires that all 13 destination bytes be writable.
3. **Unsigned control value.** A 32-bit unsigned integer used to determine the signed 32-bit return value.

These widths follow the supplied static representation. Its pointer traversal is consistent with a 32-bit address model; portability to another address width is not established.

The input names above are descriptive labels, not claims about an original public interface.

### Return value

For unsigned control value \(n\), the normal return value is:

| Condition | Returned signed 32-bit integer |
|---|---:|
| \(0 \le n < 14\) | \(n + 1\) |
| \(14 \le n \le 4{,}294{,}967{,}295\) | 13 |

The addition is evaluated only for values from 0 through 13. Its result therefore lies from 1 through 14 and cannot overflow either a 32-bit unsigned integer or a 32-bit signed integer.

The return is not consistently the record size. In particular, control value 13 returns 14, while control value 14 returns 13. No interpretation such as “bytes consumed,” “bytes available,” or “required capacity” is established.

### Preconditions for normal completion

The caller must provide:

- A dispatch root that can be read whenever the function resolves an indirect call.
- An object and dispatch table reachable through that root at each resolution.
- Callable targets at the three required table offsets.
- Execution conditions compatible with those targets’ actual calling conventions.
- A destination supporting writes at every offset from 0 through 12.
- Continued validity of all required storage across the indirect calls.

There are no visible null checks, destination-size checks, dispatch-target checks, or explicit error returns. A small control value does not relax the destination requirement.

The actual argument and receiver conventions of the indirect targets are not recoverable from the supplied representation. The contract does not assert that they receive an explicit object argument, receive no machine-level arguments, or preserve any particular register or global state.

## Behavior

### Indirect value acquisition

For this document, the three unresolved indirect dependencies have local opaque function IDs:

- **F-opaque-A:** target selected at dispatch-table byte offset `0x28`.
- **F-opaque-B:** target selected at dispatch-table byte offset `0x24`.
- **F-opaque-C:** target selected at dispatch-table byte offset `0x5C`.

These labels distinguish call roles. They are not recovered identities and do not establish that the targets are distinct.

The observable sequence is:

1. Resolve and invoke F-opaque-A.
2. Retain a 32-bit result from that call.
3. Resolve and invoke F-opaque-B.
4. Retain a 16-bit result from that call.
5. Write destination offsets 0 through 9, in increasing order.
6. Resolve and invoke F-opaque-C.
7. Retain an 8-bit result from that call.
8. Write destination offset 10.
9. Write destination offset 11.
10. Determine the return value from the unsigned control value.
11. Write destination offset 12.
12. Return normally.

Each call target is resolved through the dispatch root anew. The function does not visibly retain one object address or dispatch-table address for all three calls.

Consequently:

- Changes made by F-opaque-A can affect resolution of F-opaque-B.
- Changes made by either earlier dependency can affect resolution of F-opaque-C.
- The first ten destination writes can also affect resolution of F-opaque-C if the destination overlaps dispatch-related storage.

The values retained from F-opaque-A and F-opaque-B are used for serialization after their respective calls return. Their later source state is not consulted to reconstruct those values.

### Destination layout

Let:

- \(A\) be the retained 32-bit result from F-opaque-A.
- \(B\) be the retained 16-bit result from F-opaque-B.
- \(C\) be the retained 8-bit result from F-opaque-C.

The function performs these byte writes:

| Destination offset | Byte written | Meaning established by the evidence |
|---:|---:|---|
| 0 | `86` | Fixed hexadecimal byte |
| 1 | `0B` | Fixed hexadecimal byte |
| 2 | `01` | Fixed hexadecimal byte |
| 3 | Bits 15–8 of \(B\) | Most significant byte of retained 16-bit value |
| 4 | Bits 7–0 of \(B\) | Least significant byte of retained 16-bit value |
| 5 | `00` | Fixed zero byte |
| 6 | Bits 31–24 of \(A\) | Most significant byte of retained 32-bit value |
| 7 | Bits 23–16 of \(A\) | Second byte of retained 32-bit value |
| 8 | Bits 15–8 of \(A\) | Third byte of retained 32-bit value |
| 9 | Bits 7–0 of \(A\) | Least significant byte of retained 32-bit value |
| 10 | \(C\) | Retained 8-bit value |
| 11 | `02` | Fixed hexadecimal byte |
| 12 | `00` | Fixed zero byte |

The 16-bit and 32-bit fields are serialized most-significant byte first. Their output byte order is established by explicit extraction, not by assuming the processor’s native byte order.

The byte assignments concern bit patterns. Whether an intermediate character type is signed does not change the intended eight bits stored at each destination offset.

The dependency result widths described here are the widths retained and consumed by F-ad43d78eb6b0. The evidence does not establish the dependencies’ complete return signatures or the significance of any wider return state.

### Write extent and final contents

On normal completion, F-ad43d78eb6b0 itself issues exactly 13 destination-byte writes. It performs no visible destination read, terminator search, conditional truncation, or bulk copy.

No direct write by this function targets an offset beyond 12. This is not a guarantee that dependencies leave adjacent memory untouched.

The layout table specifies the function’s writes. To treat it as a guarantee of the destination’s complete final contents, an additional condition is needed: dependencies and external actors must not overwrite those bytes after the relevant writes.

This matters especially for offsets 0 through 9. F-opaque-C runs after those bytes have been written and may alter them. F-ad43d78eb6b0 does not restore them afterward.

## Side effects

### Direct effects

The established direct effects are:

- Reads used to traverse the dispatch root and select three indirect targets.
- Three indirect calls, in the stated order.
- Thirteen destination-byte writes, in the stated order.
- A return value derived solely from the unsigned control value after normal dependency completion.

No explicit allocation, deallocation, reference-count adjustment, ownership transfer, or global-variable assignment is visible in the function body.

The caller supplies the destination. The function does not visibly replace its address or return an independently allocated record.

### Dependency effects and ownership limits

The indirect dependencies may have additional effects. Their behavior is not supplied, so this contract does not assume that they:

- Preserve globals or dispatch state.
- Leave the destination untouched.
- Preserve the lifetime of the object or destination.
- Avoid allocation or deallocation.
- Return successfully.
- Return values stable across calls.

In particular, the absence of a direct release operation in F-ad43d78eb6b0 does not establish that no memory can be released during its execution.

### Partial progress and failures

There is no visible recovery or rollback mechanism.

If execution stops during the first or second dependency call, this function has not yet written any destination bytes. Those dependencies may nevertheless have produced their own side effects.

If execution reaches F-opaque-C, this function has already written offsets 0 through 9. If that call fails to return normally, this function does not reach its writes to offsets 10 through 12.

A memory-access failure during serialization can leave a partially written destination. Exact fault behavior, exception propagation, and resulting process state are outside the supplied evidence.

## Dependencies

| Opaque function ID | Resolution point | Retained result | Known role |
|---|---|---:|---|
| F-opaque-A | First call; table offset `0x28` | 32 bits | Supplies destination offsets 6–9 |
| F-opaque-B | Second call; table offset `0x24` | 16 bits | Supplies destination offsets 3–4 |
| F-opaque-C | After writes to offsets 0–9; table offset `0x5C` | 8 bits | Supplies destination offset 10 |

For each resolution, the visible traversal reads the dispatch root’s first word, reads the first word at the resulting object address, and selects a callable entry at the indicated byte offset.

The offsets identify dispatch locations, not semantic API names. Actual target identities, target aliasing, receiver requirements, side effects, and error behavior remain unknown.

## Acceptance examples

These examples are conditional acceptance criteria derived from static evidence. They are not reports of executed tests.

### Representative serialization

Given:

- F-opaque-A returns a retained value of `12345678` hexadecimal.
- F-opaque-B returns a retained value of `9ABC` hexadecimal.
- F-opaque-C returns a retained value of `DE` hexadecimal.
- All calls return normally.
- The dependencies do not overwrite the destination or invalidate required storage.
- The destination has at least 13 writable bytes.

The destination after completion must contain:

**86 0B 01 9A BC 00 12 34 56 78 DE 02 00**

For unsigned control value 12, the function must return 13.

### Control-value boundaries

Under otherwise identical valid conditions:

| Unsigned control value | Required return | Direct bytes written |
|---:|---:|---:|
| 0 | 1 | 13 |
| 1 | 2 | 13 |
| 12 | 13 | 13 |
| 13 | 14 | 13 |
| 14 | 13 | 13 |
| 4,294,967,295 | 13 | 13 |

A destination shorter than 13 bytes is not made safe by selecting any of these control values.

### Zero-valued dependency results

If all three retained dependency results are zero and the other conditions above hold, the destination must contain:

**86 0B 01 00 00 00 00 00 00 00 00 02 00**

The fixed bytes remain present; zero-valued results do not suppress any write.

### Visibility at the third call

If F-opaque-C observes the destination without modifying it, and the earlier dependencies do not modify that storage, it must be able to observe the already-written bytes at offsets 0 through 9.

Offsets 10 through 12 have not yet been written by this invocation at that point.

### Dispatch mutation

If F-opaque-A changes the dispatch root to another valid object before returning, selection of F-opaque-B must use the root state read after that return, not a table cached before the first call.

This criterion requires the replacement object and target to satisfy the actual calling convention and lifetime requirements.

### Intervening destination mutation

If F-opaque-C overwrites destination offset 0 and returns normally without invalidating storage, F-ad43d78eb6b0 does not rewrite offset 0. It proceeds with writes to offsets 10, 11, and 12.

A consumer must not assume that the initial fixed prefix survives arbitrary dependency side effects.

## Uncertainties

The following are **documented_only**:

- **Protocol semantics:** The meanings of the fixed bytes and serialized fields are unknown.
- **Control-value meaning:** Its relationship to capacity, progress, length, or another caller concept is unknown.
- **Dependency signatures:** Actual parameter passing, receiver passing, and full return types are unknown.
- **Dependency effects:** Global mutation, memory mutation, allocation, release, failure, and reentrancy behavior are unknown.
- **Aliasing policy:** No non-overlap check is visible. The evidence does not establish whether overlapping destination and dispatch storage is intended or supported.
- **Concurrency:** No synchronization or cross-thread visibility guarantee is established.
- **Fault semantics:** Invalid addresses and abnormal dependency completion have no documented recovery behavior here.
- **Portability:** The supplied representation supports the stated 32-bit traversal model, not a platform-independent interface guarantee.

An inferred intent is that the function constructs a compact externally consumed record. That inference is unnecessary to the byte-level contract and must not be used to infer field names or validation rules.

## Evidence

The established behavior comes solely from the supplied static function representation:

- Three indirect calls appear in the documented order and at the documented dispatch offsets.
- Their consumed results occupy 32-bit, 16-bit, and 8-bit retained values.
- Explicit byte assignments establish the 13-byte layout and serialization order.
- An unsigned comparison against 14 selects between the bounded increment and constant return value.
- The final zero-byte write occurs after return-value selection and before normal return.
- No visible bounds validation, rollback, direct allocation, or direct release is present.

No runtime tests have been performed. The acceptance examples express required behavior under their stated conditions, not observed execution results.
