<!-- Trusted provenance: F-4c1d92fcf57d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4c1d92fcf57d

## Purpose

F-4c1d92fcf57d dispatches a selector and a value against a state object obtained from a dependency. Supported operations update one 32-bit field, update either member of a pair and conditionally recompute associated state, or encode one of three accepted values into a single byte.

Unsupported selectors and unsupported values for the byte-encoding operation enter an error-recording path. A negative value for one supported selector enters the same path with an externally supplied error value. Error recording preserves an already nonzero error field.

The field roles beyond these operations are not established. Interpreting the paired fields as endpoints and the recomputed field as a scale or reciprocal is plausible intent, not part of the established contract.

## Interface

### Inputs and result

The function takes two integer inputs:

- **Selector:** compared against `0x0B62`, `0x0B63`, `0x0B64`, and `0x0B65`.
- **Value:** interpreted according to the selected operation. It may be stored as a full-width integer, tested for signed negativity, or compared with three specific encodings.

The supplied representation uses 32-bit integer fields and one-byte stores. This contract therefore describes 32-bit integer values and byte offsets based on four-byte integer slots. Independent ABI verification is unavailable.

There is no explicit return value. Observable results are dependency calls, memory accesses, state changes, and any behavior performed by those dependencies.

### State acquisition

Every invocation first calls **F-7fba0909c452**, with no explicit arguments, to obtain a state pointer.

- If the result is null, the function returns immediately.
- In that case, it performs no direct state-field access, selector-specific update, arithmetic-helper call, or error recording.
- The acquisition dependency may itself have side effects; null does not imply that the entire invocation is side-effect-free.

A nonnull result is used directly. No further validation of its extent, alignment, writability, or lifetime is visible.

### State layout

Offsets are relative to the returned state pointer.

| Offset | Width accessed | Contract label | Observable use |
|---|---:|---|---|
| `0x000` | 32 bits | Error field | Read and conditionally written on an error path |
| `0x8B8` | 8 bits | Encoding byte | Written by accepted `0x0B65` operations |
| `0x8BC` | 32 bits | Pair field A | Written by `0x0B63`; read during pair processing |
| `0x8C0` | 32 bits | Nonnegative field | Written by accepted `0x0B62` operations |
| `0x8C4` | 32 bits | Pair field B | Written by `0x0B64`; read during pair processing |
| `0x8C8` | 32 bits | Derived field | Written after pair recomputation |
| `0x8CC` | 8 bits | Mode byte | Written after pair recomputation |

A byte write does not directly overwrite the other bytes in the corresponding four-byte slot. In particular, writes at `0x8B8` and `0x8CC` are not full-width integer assignments.

Storage must be valid for the accesses reached by the selected path. An object covering all listed fields must extend through byte offset `0x8CC`, inclusive. No allocation or capacity check is performed here.

## Behavior

### Common dispatch

State acquisition precedes selector handling, including for unsupported selectors.

Once a nonnull state pointer has been obtained, the function selects one of the operations below. It does not reject an operation merely because the error field is already nonzero.

### Selector `0x0B62`: nonnegative assignment

The supplied representation obtains an external error-code value on this selector’s path before deciding whether the input value is nonnegative.

For a signed value greater than or equal to zero:

1. Store the complete 32-bit value at `0x8C0`.
2. Return without pair recomputation or direct error-field access.

For a negative value:

1. Do not write `0x8C0`.
2. Enter error recording using the externally obtained error-code value.

The external error value is not supplied numerically. It must not be assumed to equal the fixed error code used by other invalid operations. Its storage width, mutability, and synchronization properties are not independently established.

Zero is accepted. Every positive value representable by the described signed 32-bit input is accepted. No additional bound or normalization is visible.

### Selectors `0x0B63` and `0x0B64`: paired-field assignment

For selector `0x0B63`, store the complete input value in pair field A at `0x8BC`.

For selector `0x0B64`, store the complete input value in pair field B at `0x8C4`.

After that assignment, both selectors follow the same processing:

1. Compare pair field A with pair field B.
2. If they are equal, return immediately.
3. Otherwise, form the 32-bit difference **B − A**.
4. Select a recomputation path from the signed high-byte classification of that difference.
5. Write the recomputation result to the derived field.
6. Write the corresponding mode byte.
7. Return.

The selected pair field is therefore updated even when the equality check causes an early return. On that early return, this function does not refresh either the derived field or the mode byte. Their previous contents may remain inconsistent with a newly equal pair.

No ordering, range, or nonnegativity constraint is applied to the pair inputs.

#### Difference classification

The represented operation arithmetic-right-shifts the difference by 24 bits and distinguishes three cases:

- The shifted result is zero.
- The shifted result is negative one.
- The shifted result is anything else.

The first two cases take the unscaled path. The third takes the scaled path.

For a defined signed 32-bit difference, the unscaled interval is:

**−16,777,216 through +16,777,215, inclusive.**

The asymmetry is intentional in this description: `−16,777,216` has the accepted all-one high byte, while `+16,777,216` does not have the accepted all-zero high byte.

Subtraction overflow is a separate uncertainty. The supplied representation does not establish a portable language-level result when mathematical B − A exceeds the signed 32-bit range. Wrapping must not be silently assumed for such inputs.

#### Scaled path

When the signed high-byte classification is neither zero nor negative one:

1. Arithmetic-right-shift the difference by eight bits.
2. Call **F-dc13818ac82a** with that shifted value as its one explicit argument.
3. Store the returned 32-bit result at `0x8C8`.
4. Store the single byte `8` at `0x8CC`.
5. Return.

The argument is the shifted difference, not the original input value or either pair field individually. Arithmetic right shift of a negative value is not interchangeable with signed division rounded toward zero.

The dependency runs before either of the final two stores. Its side effects, if any, can therefore occur before the derived field and mode byte are overwritten.

#### Unscaled path

When the signed high-byte classification is zero or negative one:

- If the difference is nonzero, call **F-829ee7540d02** with no explicit arguments and use its returned 32-bit result.
- If the difference is zero, use `0x7FFFFFFF` without calling that dependency.

Then:

1. Store the selected result at `0x8C8`.
2. Store the single byte `0` at `0x8CC`.
3. Return.

No explicit difference argument is passed to F-829ee7540d02. Its result must not be specified as a particular function of the difference without additional evidence.

Under stable memory and ordinary nonoverflowing arithmetic, the earlier equality return prevents the zero-difference alternative from being reached. The alternative is nevertheless present in the supplied representation. It is not the behavior for an ordinary equal-pair update.

### Selector `0x0B65`: byte encoding

Exactly three values are accepted:

| Input value | Byte stored at `0x8B8` |
|---|---:|
| `0x0800` | `1` |
| `0x0801` | `2` |
| `0x2601` | `0` |

Each accepted value causes one direct byte write followed by return. No pair processing or direct error-field access occurs.

Every other value enters error recording with code `0x0500`. An invalid value does not directly change the encoding byte.

### Unsupported selectors

Every selector other than the four listed above enters error recording with code `0x0500`, after successful state acquisition.

### Error recording

Given a selected error code:

1. If the code is zero, return without reading or writing the error field.
2. Otherwise, read the 32-bit error field at offset `0x000`.
3. If that field is zero, write the selected code there.
4. If that field is nonzero, leave it unchanged.
5. Return.

This is a conditional “preserve an existing error” policy. The visible read-then-write sequence is not evidence of an atomic first-error operation.

Successful operations do not directly clear a preexisting error.

## Side effects

Direct writes are limited to the fields listed in the layout table. Dependency side effects are additional and are not bounded by that list.

Important ordering properties are:

- State acquisition occurs before all direct state access.
- A pair assignment occurs before comparison and any arithmetic-helper call.
- On recomputation paths, the helper returns before the derived-field store.
- The derived-field store precedes the mode-byte store.
- Error recording reads the existing error before conditionally replacing zero.
- A zero selected error code avoids direct access to the error field.

The source establishes the comparison and subsequent subtraction as separate uses of the pair fields. It does not establish a coherent snapshot, synchronization, or a precise ordering between the two operand reads within an expression.

The state pointer is retained across arithmetic-helper calls. There is no visible reacquisition afterward. Those calls must not be assumed to preserve globals, other state fields, or the state allocation itself.

For a normally completing recomputation, the retained pointer must still permit the final writes after the helper returns. If a dependency invalidates that storage, this function contains no visible recovery mechanism.

No direct allocation, deallocation, ownership transfer, or reference-count operation is shown. This does not establish equivalent restrictions on dependencies. Thread safety, reentrancy, fault recovery, and rollback are not provided by the visible body.

## Dependencies

| Opaque function ID | Explicit interface used here | Established role |
|---|---|---|
| **F-7fba0909c452** | No explicit arguments; returns a pointer-like result | Supplies the state pointer; null causes immediate return |
| **F-dc13818ac82a** | One shifted 32-bit difference argument; returns a 32-bit result | Supplies the derived value on the scaled path |
| **F-829ee7540d02** | No explicit arguments; returns a 32-bit result | Supplies the derived value on the ordinary nonzero unscaled path |

The names above convey no mathematical, ownership, or purity guarantee. In particular, the no-argument dependency may consult hidden state, and neither arithmetic dependency is shown to preserve the state object.

The external error-code datum used by selector `0x0B62` is an additional dependency whose value is unknown.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Except where stated otherwise, assume valid stable storage, normally returning dependencies, no concurrent mutation, and representable signed subtraction.

1. **Null state:** If F-7fba0909c452 returns null, even an unsupported selector causes no direct error-field write and no arithmetic-helper call.

2. **Accepted zero:** With selector `0x0B62` and value zero, `0x8C0` becomes zero. A preexisting nonzero error field is not directly cleared.

3. **Negative-value error:** With selector `0x0B62`, a negative value, external error code `E ≠ 0`, and an initially zero error field, the error field becomes `E`; `0x8C0` remains unchanged by this function.

4. **Zero external error:** With the same negative input but external error code zero, neither `0x8C0` nor the error field is directly written, and the error field is not read on that path.

5. **Equal-pair early return:** If pair field B is `25` and selector `0x0B63` supplies `25`, pair field A becomes `25`. Neither arithmetic dependency is called, and the derived field and mode byte retain their prior contents.

6. **Unscaled difference:** If the updated pair yields A = `3` and B = `10`, F-829ee7540d02 is called with no explicit arguments. If it returns `R`, the final direct stores place `R` at `0x8C8` and byte zero at `0x8CC`.

7. **Positive scaling boundary:** With A = `0` and B = `16,777,216`, F-dc13818ac82a receives `65,536`. Its result is stored before byte `8` is written to the mode byte. B = `16,777,215` instead selects the unscaled dependency.

8. **Negative scaling boundary:** A difference of `−16,777,216` selects the unscaled path. A difference of `−16,777,217` selects the scaled path, whose arithmetic-shifted argument is `−65,537`.

9. **Byte preservation:** With selector `0x0B65` and value `0x0801`, only byte `0x8B8` is directly set to `2`; neighboring bytes in that four-byte slot are not part of the store.

10. **Invalid encoding and sticky error:** An invalid `0x0B65` value records `0x0500` if the error field is zero. If the field already contains another nonzero value, that value remains unchanged.

11. **Overflow boundary:** A pair whose mathematical difference exceeds signed 32-bit range has no portable expected recomputation result established here. Such an example cannot be assigned a wrapping outcome without additional evidence.

## Uncertainties

The following are **documented_only**, not guessed behavior:

- Exact ABI, pointer width, byte order, and independently verified integer semantics.
- Defined subtraction behavior outside the signed 32-bit range.
- The external error-code value and its storage properties.
- Mathematical meaning and internal behavior of both arithmetic dependencies.
- Ownership, lifetime guarantees, and side effects of the returned state pointer.
- Whether dependencies can mutate, replace, or release relevant storage.
- Concurrency guarantees and permissible asynchronous state mutation.
- Exact read counts and instruction-level ordering beyond the represented control and data dependencies.
- Whether the zero-difference alternative is reachable in the actual execution environment.

The paired-field and derived-value terminology is descriptive. Any domain-specific interpretation remains inferred intent.

## Evidence

Established behavior comes solely from the supplied static function representation: explicit comparisons, dependency calls, field-access widths and offsets, branch conditions, arithmetic operations, and store order.

Offsets are derived from four-byte indexed integer accesses; the encoding and mode writes are explicitly one byte wide. No dependency implementation, runtime trace, executable behavior, or external ABI documentation was supplied.

No runtime tests have been performed.
