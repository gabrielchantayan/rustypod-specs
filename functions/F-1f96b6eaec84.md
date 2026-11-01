<!-- Trusted provenance: F-1f96b6eaec84; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1f96b6eaec84

## Purpose

F-1f96b6eaec84 selects a 32-bit value through one of two paths and, if selection succeeds, passes that value and an opaque caller-supplied payload to F-1365a014da83.

The path depends on a signed 16-bit field at byte offset 2 from the supplied object address:

- If the field equals hexadecimal `4244`, the function derives the selected value directly from another object field and the second input.
- Otherwise, it asks F-bacb3587c147 to produce the selected value. A nonzero status terminates processing without calling F-1365a014da83.

This selection-and-dispatch structure is established by the supplied static evidence. The meaning of the object, the selected value, the status codes, and the payload is not established. In particular, the evidence does not justify describing the selected value as an address, index, block number, or resource identifier.

## Interface

The function has three inputs and no explicit return value.

| Input | Representation | Established use |
|---|---|---|
| Object address | Address-bearing value represented by a 32-bit integer in the supplied type model | Base address for field reads; forwarded to both dependencies where applicable |
| Position value | Unsigned 32-bit integer | Logically shifted right by 12 on the direct path, or by 3 on the delegated path |
| Payload | Opaque 32-bit value | Forwarded unchanged as the third argument to F-1365a014da83 |

The payload is not dereferenced or interpreted by this function. It may represent data or an address, but neither interpretation is established.

### Referenced object layout

All offsets are byte offsets relative to the object address. These are the only object fields accessed directly by this function.

| Offset | Width | Interpretation and access condition |
|---|---|---|
| `0x02` | 16 bits | Signed discriminator, read on every invocation that reaches the initial access |
| `0x10` | 16 bits | Signed value, read only when the discriminator equals `0x4244` |
| `0x15C` | One pointer-sized value in the supplied 32-bit model | Pointer to storage containing a 32-bit word, read only on the delegated path |

On the delegated path, the pointer obtained from offset `0x15C` is dereferenced once to obtain a 32-bit word. That word, rather than the address of its storage, becomes the second argument to F-bacb3587c147.

The function also provides two pieces of temporary storage to that dependency:

- One four-byte scratch region, whose same address is supplied in two argument positions.
- One 32-bit output slot intended to receive the selected value.

Neither temporary is initialized before the dependency call.

### Preconditions

For defined execution, each memory access taken on the selected path must refer to readable storage with alignment and representation acceptable to the execution environment.

The initial discriminator read is unconditional. The direct path additionally requires the signed 16-bit field at offset `0x10` to be readable. The delegated path instead requires both the pointer field at offset `0x15C` and the 32-bit word it designates to be readable.

The supplied function contains no null check, bounds check, alignment check, or recoverable handling for invalid memory. Whether a particular invalid access faults, traps, or has some other consequence is outside the established contract.

## Behavior

### Path selection

The function first reads the signed 16-bit discriminator at object offset `0x02` and compares its numeric value with hexadecimal `4244`, decimal 16964.

This is a numeric comparison after interpreting the field as a signed 16-bit value. It is not established as a comparison against a two-character string. The byte sequence representing that value depends on the target’s byte order.

Exactly one selection path is taken. The direct path does not call F-bacb3587c147. The delegated path does not directly read the signed 16-bit field at offset `0x10`.

### Direct selection

When the discriminator matches:

1. The function reads the signed 16-bit value at object offset `0x10`.
2. It sign-extends that value to the 32-bit integer domain.
3. It derives an unsigned quotient from the position value by logical right shift by 12 bits.
4. It combines the sign-extended field and that quotient using the arithmetic conversions of the supplied type model.
5. It stores the resulting 32-bit value for the final dependency call.

The shifted position ranges from 0 through `0xFFFFF`, inclusive. It is equivalent to unsigned division by 4096 with the remainder discarded.

The addition has an unsigned operand. In the supplied 32-bit model, the signed field contribution is therefore converted to unsigned 32-bit form before addition, and the addition is modulo \(2^{32}\). Negative field values must not be reinterpreted as unsigned 16-bit values before this operation: the signed 16-bit interpretation and sign extension are material.

The result is assigned to a signed 32-bit slot. For unsigned results above the largest signed 32-bit value, the precise language-level conversion depends on the implementation. Under a conventional two’s-complement, bit-preserving interpretation, such results represent negative signed values. That interpretation is distinguished from the directly established unsigned arithmetic.

No status check follows direct selection. The function proceeds to F-1365a014da83.

### Delegated selection

When the discriminator does not match, the function obtains the indirect 32-bit word described in the interface and calls F-bacb3587c147 with eight arguments, in this positional order:

| Position | Value |
|---|---|
| 1 | Original object address |
| 2 | 32-bit word loaded through the pointer stored at object offset `0x15C` |
| 3 | Integer constant 1 |
| 4 | Address of the four-byte scratch region |
| 5 | Position value logically shifted right by 3 bits |
| 6 | Integer constant 0 |
| 7 | Address of the 32-bit selected-value output slot |
| 8 | The same scratch-region address supplied in position 4 |

The shifted position ranges from 0 through `0x1FFFFFFF`, inclusive. It is equivalent to unsigned division by 8 with the remainder discarded. No additional scaling, rounding, or masking is performed by this function.

The fourth and eighth arguments alias exactly. They do not designate two separate four-byte buffers. Any dependency behavior involving those arguments must account for that aliasing.

The output slot and scratch region have no established initial contents. A dependency that reads either region before establishing its contents cannot rely on initialization performed here.

After F-bacb3587c147 returns, its integer status is compared with zero:

- **Nonzero:** F-1f96b6eaec84 returns immediately. F-1365a014da83 is not called.
- **Zero:** The function reads the selected-value output slot and proceeds to F-1365a014da83.

All nonzero statuses receive the same treatment. The function does not distinguish positive from negative statuses, retry the operation, expose the status to its caller, or select a fallback value.

The zero-status path relies on the output slot containing a usable value. The function does not independently check that F-bacb3587c147 wrote it.

### Final dispatch

After direct selection, or after a zero status from delegated selection, the function calls F-1365a014da83 with five arguments:

| Position | Value |
|---|---|
| 1 | Integer constant 0 |
| 2 | Selected 32-bit value |
| 3 | Original opaque payload |
| 4 | Integer constant 0 |
| 5 | Original object address |

There is no subsequent status test or explicit use of a dependency result. After this call returns normally, F-1f96b6eaec84 returns.

Forwarding the original object address does not imply that the object’s contents are unchanged. On the delegated path, F-bacb3587c147 has already had an opportunity to modify accessible state.

## Side effects

The function’s explicit memory writes are confined to its own temporary storage: it assigns the selected-value slot directly on the matching-discriminator path. On the delegated path, it exposes temporary storage to F-bacb3587c147 for that dependency to access.

There are no explicit stores to the caller’s object in the supplied function. This does **not** establish that the complete invocation leaves the object unchanged. Both dependencies receive the object address, and their side effects are unknown.

The observable ordering established locally is:

- The discriminator read precedes path-specific work.
- On the direct path, the signed field read precedes final dispatch.
- On the delegated path, the pointer-field read precedes the dependent word read; both precede F-bacb3587c147.
- The status returned by F-bacb3587c147 is checked before any final dispatch.
- On a zero status, the selected-value output is consumed after F-bacb3587c147 returns.
- F-1365a014da83, when called, occurs after selection.

These constraints do not establish a general left-to-right evaluation order for all call arguments.

No explicit allocation, release, reference-count adjustment, or ownership transfer appears in this function. Dependency ownership behavior remains unknown. In particular, the function does not establish that either dependency preserves globals or refrains from releasing externally owned storage.

The temporary addresses are usable as local storage during the delegated call. They do not provide a valid persistent-storage guarantee after this function returns.

## Dependencies

### F-bacb3587c147

Called once only on the nonmatching-discriminator path.

Its locally observable contract is an eight-argument call with a zero/nonzero status result, an output-slot address, and two exactly aliasing scratch-buffer arguments. The meanings of the constant arguments and indirect word are undocumented.

A usable selected value on zero status is an operational requirement of the caller, not independently proven behavior of this dependency. Its initialization discipline, side effects, ownership behavior, and status meanings are unknown.

### F-1365a014da83

Called once after direct selection or successful delegated selection; not called after a nonzero delegated status.

Its locally observable contract is the five-argument sequence described above. The meaning of those arguments, any returned value, and all dependency-side effects remain unknown. F-1f96b6eaec84 does not inspect a result from this call.

## Acceptance examples

These are conditional static acceptance examples, not runtime test results. They assume valid storage and dependencies that return normally.

1. **Matching discriminator with discarded low position bits.**  
   With discriminator `0x4244`, signed field 7, and position `0x2FFF`, the shifted position is 2. F-bacb3587c147 is not called. F-1365a014da83 receives selected value 9, the unchanged payload, and the original object address, with zero in its first and fourth positions.

2. **Direct-path shift boundary.**  
   With a matching discriminator and signed field 7, position `0x0FFF` produces selected value 7; position `0x1000` produces 8.

3. **Signed field contribution.**  
   With a matching discriminator, signed field −1, and position `0x1000`, the unsigned addition produces zero. Treating the field as unsigned 16-bit would produce a different result and would not satisfy this behavior.

4. **Negative-result representation condition.**  
   With a matching discriminator, signed field −1, and position zero, the unsigned intermediate is `0xFFFFFFFF`. Under a bit-preserving two’s-complement conversion, the selected signed value is −1. The conversion assumption must remain explicit.

5. **Successful delegated selection.**  
   With a nonmatching discriminator and position 15, F-bacb3587c147 receives 1 in its fifth position. If it writes 123 to the output slot and returns zero, F-1365a014da83 subsequently receives selected value 123. The scratch addresses in delegated argument positions 4 and 8 must be identical.

6. **Delegated failure.**  
   If F-bacb3587c147 returns any nonzero status, final dispatch does not occur, even if that dependency wrote the output slot. Any earlier dependency side effects are not rolled back by this function.

7. **Path-specific memory requirements.**  
   A matching discriminator does not cause this function to read the pointer at offset `0x15C`. A nonmatching discriminator does not cause it to read the signed field at offset `0x10`. Neither statement restricts what dependencies may access.

## Uncertainties

The following items are **documented_only**:

- Semantic identities of the object, discriminator, selected value, payload, and indirect word.
- Target byte order and exact alignment requirements.
- Exact target behavior for unsigned-to-signed conversion outside the signed range.
- Whether F-bacb3587c147 initializes the output slot whenever it returns zero.
- Whether either dependency reads the initially uninitialized scratch storage.
- Dependency effects on object contents, globals, external resources, and memory ownership.
- Whether dependencies retain temporary addresses.
- Concurrency, atomicity, synchronization, and volatile-access requirements.
- Exceptional or nonlocal control flow from memory accesses or dependencies.

The interpretation of the discriminator as a format or object-kind tag is inferred intent only. The numeric branch condition itself is established.

## Evidence

This contract is based solely on the supplied static function body and its supplied dependency-ID mapping.

The static evidence establishes the field offsets, indicated widths and signedness, two branch paths, shift counts, call argument positions, scratch-address aliasing, zero/nonzero status gate, and absence of explicit local recovery or cleanup.

Dependency implementations and an independent target ABI description were not supplied. Claims about their semantics or target-specific representation have therefore not been promoted to established behavior.

No runtime tests have been performed. The acceptance examples are deductions from the static evidence, subject to their stated preconditions and representation assumptions.
