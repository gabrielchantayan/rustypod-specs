<!-- Trusted provenance: F-a314ad5db9ef; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a314ad5db9ef

## Purpose

F-a314ad5db9ef conditionally updates compact state associated with a current context. It admits a selector, converts the supplied value, validates the target, obtains a parameter object, and chooses an update branch. Every represented rejection uses the same sticky-error rule: write `0x0500` only if the context’s current error field is zero.

The established behavior comprises dependency-call order, represented global reads, comparisons, direct memory accesses, and return conditions. The constants suggest texture-parameter processing, but that is inferred intent, not an additional contract or a basis for importing external enumeration rules.

Processing is not transactional. A dependency may have effects before a later rejection, and the function performs no visible rollback. An existing error neither prevents a valid update nor gets explicitly cleared by success.

## Interface

### Arguments and return

| Input | Width and interpretation | Role |
|---|---|---|
| Target | Signed 32-bit integer | Must equal `0x0DE1` after value conversion to proceed to parameter-object lookup. |
| Selector | Signed 32-bit integer | Controls admission and dispatch. |
| Supplied value | Opaque 32-bit quantity | Passed unchanged to F-c9e0501fef5b after admission. |

The function returns no value or status. Its outcomes are dependency effects, direct state writes, conditional error recording, or an early return.

F-c9e0501fef5b produces a 32-bit converted value. Ordered comparisons involving this result are signed. F-63f3c76ed3e0 also produces a 32-bit result; the caller treats signed `-1` as failure and every other result as eligible for a byte store.

### Storage layout

F-7fba0909c452 supplies a context address. A null result is explicitly handled.

F-1411dcc31aed supplies a parameter-object address. No null, bounds, or lifetime check is represented before its use.

| Storage | Byte offset | Access width | Use |
|---|---:|---:|---|
| Context | `0x000` | 32 bits | Error field, read on rejection and conditionally written |
| Context | `0x90A` | 1 byte | Boolean state |
| Parameter object | `0x104` | 1 byte | First component of the two-byte encoding |
| Parameter object | `0x105` | 1 byte | Single-byte encoding |
| Parameter object | `0x106` | 1 byte | Second component of the two-byte encoding |
| Parameter object | `0x107` | 1 byte | Secondary-conversion result on dispatch equality |
| Parameter object | `0x108` | 1 byte | Secondary-conversion result on the reachable `0x2803` branch |

The parameter-object address is represented as a 32-bit quantity. This does not establish a complete platform pointer model, address-arithmetic overflow rules, or alignment requirements.

These offsets do not establish the full extent or ownership of either object. Context and parameter-object storage are not proven disjoint.

### Global inputs and preconditions

Three distinct 32-bit global quantities participate:

- **Admission pivot:** used before value conversion.
- **Dispatch pivot:** used after parameter-object lookup.
- **Encoding base:** used in value classification.

Their values, initialization, and relationships are unknown. In particular, the two pivots need not be equal.

Ordinary completion requires every reached dependency to return normally and every reached memory access to address valid storage. A nonnull context alone does not establish those conditions. These are execution preconditions, not checks performed by this function.

Repeated reads are described individually below. They are not replaced by an assumed entry-time snapshot. Conversely, a represented read does not by itself establish a volatile access, a synchronization event, or an externally observable machine-level read.

## Behavior

### Context lookup and selector admission

F-7fba0909c452 is called first, without explicit arguments. If its result is null, the function returns immediately: no other dependency is called and no direct state or error write occurs.

For a nonnull context, admission has the following read and decision structure:

| Admission stage | Represented read and decision |
|---|---|
| Initial equality | Read the admission pivot and compare it with the selector. Equality admits immediately. |
| Ordered comparison after inequality | Read the admission pivot again and compare the selector against that value using signed ordering. |
| Below-pivot path | Read the admission pivot a third time, before checking the fixed selectors. Admit only `0x2800` or `0x2801`. |
| Not-below-pivot path | No third admission-pivot read is represented. Admit only `0x2803` or `0x8191`. |

The third read on the below-pivot path occurs even for selector `0x2800`. Its returned value does not affect the eventual admission decision: for `0x2800`, admission succeeds without using that value; for other selectors on that path, the fixed-selector arithmetic determines admission instead.

Thus, equality admission involves one represented admission-pivot read; admission through the not-below path involves two; and reaching the below path involves three, whether that path ultimately admits or rejects. This is a statement about the supplied representation, not proof that all such reads are externally observable or must survive optimization.

With a stable admission pivot, the accepted set is its equality value, `0x2800` and `0x2801` when below it, and `0x2803` and `0x8191` when above it. If the first two reads differ, equality and ordering need not refer to one common pivot value.

The nonconstant admission checks use 32-bit subtraction: difference `1` from `0x2800`, or difference `0x91` from `0x8100`. Signed-overflow behavior for extreme selector values remains documented_only; no unchecked assumption of wraparound is required for the ordinary fixed-selector cases.

Admission failure proceeds directly to error recording, without value conversion.

### Conversion, target validation, and lookup

After admission, F-c9e0501fef5b receives the supplied 32-bit value.

Only after that dependency returns does the function compare the target with `0x0DE1`. A mismatch triggers error recording. Consequently, an invalid target does not prevent the earlier conversion call, but it does prevent parameter-object lookup and every successful direct update branch.

For a matching target, F-1411dcc31aed receives the context address. After its return, the function reads an encoding-base snapshot. This read occurs before dispatch and also occurs when the eventual branch does not use the snapshot.

The snapshot is retained for the second component of the two-byte encoding. Other encoding decisions may use later, separate base reads.

Neither intervening dependency is assumed to preserve any global, context field, parameter storage, or address validity.

### Dispatch precedence

Dispatch first reads the dispatch pivot and tests equality with the selector. Equality selects offset `0x107`, regardless of whether the selector also equals a fixed selector used elsewhere.

If equality fails, the function reads the dispatch pivot again and performs a signed comparison:

| Second dispatch comparison | Selector | Selected behavior |
|---|---|---|
| Pivot is less than or equal to selector | `0x2803` | Secondary conversion, then possible write at `0x108` |
| Pivot is less than or equal to selector | `0x8191` | Context Boolean update |
| Pivot is less than or equal to selector | Any other value | Error recording |
| Pivot is greater than selector | `0x2800` | Single-byte encoding |
| Pivot is greater than selector | `0x2801` | Two-byte encoding |
| Pivot is greater than selector | Any other value | Error recording |

With a stable dispatch pivot, these are the above-pivot and below-pivot cases because equality has already been handled. With differing reads, the second comparison may observe equality even though the first did not.

Admission does not guarantee an accepting dispatch branch. The pivots may differ, and dependencies execute between their uses.

### Secondary-conversion fields

The dispatch-equality branch and the reachable `0x2803` branch call F-63f3c76ed3e0 with the converted value.

- A result of signed `-1` triggers error recording without a direct parameter-byte write.
- Every other result contributes its low eight bits to the selected byte store.
- The equality branch writes offset `0x107`; the `0x2803` branch writes offset `0x108`.

No additional range validation is represented. For example, `0x100` is successful and writes `0x00`; `-2` is successful and writes `0xFE`.

### Boolean update

The reachable `0x8191` branch stores the Boolean result of testing the full 32-bit converted value for inequality with zero at context offset `0x90A`.

Zero stores false; every nonzero value stores true. In particular, `0x100` stores true even though its low byte is zero.

This branch does not call F-63f3c76ed3e0. The representation establishes a one-byte Boolean store, but does not independently resolve its physical false/true byte encoding.

### Single-byte encoding

For selector `0x2800` on its encoding branch:

| Condition, in precedence order | Result |
|---|---|
| Converted value equals `0x2600` | Write byte `1` at `0x105` |
| Otherwise, converted value equals a fresh encoding-base read | Write byte `2` at `0x105` |
| Neither condition holds | Error recording; no direct parameter-byte write |

The retained early snapshot is not used here. The fresh base read occurs only after the `0x2600` comparison fails.

If that base equals `0x2600`, the constant comparison takes precedence and writes `1`, not `2`.

### Two-byte encoding

For selector `0x2801` on its encoding branch, three thresholds are formed before classification, in this order:

1. A fresh encoding-base read plus `0x100`.
2. Another encoding-base read plus `0x101`.
3. Another encoding-base read plus `0x102`.

Each addition is represented as 32-bit integer arithmetic. Overflow semantics are unresolved. The reads are separate, so the resulting thresholds are not necessarily consecutive or numerically ordered.

The first component uses these rules:

| Region and condition | First component |
|---|---:|
| Equal to the first threshold | `2` |
| Below the first threshold, equal to `0x2600` | `1` |
| Below the first threshold, not `0x2600`, equal to a further encoding-base read | `2` |
| Below the first threshold, matching neither preceding case, equal to `0x2700` | `1` |
| Above the first threshold, equal to the second threshold | `1` |
| Above the first threshold, not equal to the second threshold, equal to the third threshold | `2` |
| Otherwise | Rejection |

The further base read in the lower region occurs only when the converted value is below the first threshold and is not `0x2600`. It precedes the `0x2700` check.

Rejection records the sticky error before either component is directly written. Otherwise, the first component is written at offset `0x104`.

The second component is then classified using the retained early snapshot and already computed thresholds:

| Region and condition | Second component |
|---|---:|
| Equal to the first threshold | `1` |
| Below the first threshold, equal to `0x2600` or the early base snapshot | `0` |
| Below the first threshold, matching neither preceding case, equal to `0x2700` | `1` |
| Above the first threshold, equal to the second or third threshold | `2` |
| Otherwise | `0xFF` |

The second component is written at offset `0x106`, after the first write. Its `0xFF` fallback is successful output, not a rejection. No further encoding-base read is represented during second-component classification.

All precedence rules matter when constants, snapshots, or thresholds collide. In particular, equal second and third thresholds select first component `1`, not `2`, in the above-first-threshold region.

### Error recording

Every represented rejection reads the current 32-bit context error field at offset zero:

- If it is zero, write `0x0500`.
- If it is nonzero, leave it unchanged.
- Return without a status value.

The error is read at rejection time, not captured at entry. Earlier dependency effects can therefore determine whether the direct error write occurs.

Successful branches do not explicitly inspect or clear this field.

## Side effects

Direct writes are limited to the selected state byte, the ordered pair at offsets `0x104` and `0x106`, or the conditional 32-bit error write. A rejected encoding does not directly write a parameter byte.

The represented reads include the otherwise decision-irrelevant third admission-pivot read on the below-pivot path. Neither that read nor the other global reads establishes volatility, device access, synchronization, or external observability by itself.

There is no visible allocation, release, or ownership-transfer operation. Returned addresses are used without establishing ownership. Dependencies may have allocation, retention, release, replacement, or other effects; none are inferred away.

No rollback, locking, explicit memory barrier, or atomic two-byte update is established. Aliasing may make a selected state write affect storage described under another role, including the error field. Statements about leaving another field unchanged concern explicit accesses, not a guarantee of nonaliasing.

Invalid addresses, storage invalidation, abnormal dependency exits, and concurrent interference have no established recovery behavior.

## Dependencies

| Opaque ID | Invocation and caller interpretation | Unresolved behavior |
|---|---|---|
| F-7fba0909c452 | First call; no explicit arguments; supplies context address | Ownership, lifetime, side effects, abnormal exits |
| F-c9e0501fef5b | After admission; receives supplied value; returns converted 32-bit value | Conversion rules and effects on globals or memory |
| F-1411dcc31aed | After target validation; receives context; returns unchecked parameter-object address | Address validity, ownership, retention, allocation, release, other effects |
| F-63f3c76ed3e0 | Only for dispatch equality or reachable `0x2803`; receives converted value | Conversion rules and effects; caller recognizes only `-1` as failure |

No dependency is assumed to preserve global values, object contents, or address validity.

## Acceptance examples

These are conditional deductions, not runtime observations.

Unless overridden, assume a valid nonnull context, valid nonoverlapping storage, normal dependency returns, no relevant dependency mutations or concurrent interference, target `0x0DE1`, both pivots stable at `0x2802`, and every encoding-base read equal to `0x2601`. Values described as converted are outputs of F-c9e0501fef5b, not interpretations of the supplied input.

1. **Null context.** A null result from F-7fba0909c452 ends processing before every other dependency and direct write.

2. **Admission read sequence for `0x2800`.** The initial equality fails, the second pivot read selects the below-pivot path, and a third admission-pivot read occurs before fixed-selector admission. This third read occurs despite `0x2800` being accepted directly.

3. **Unused third-read value.** Keep the first two admission-pivot reads at `0x2802`, but let the third yield `0x1000`. Selector `0x2801` remains admitted. This changes neither its fixed-selector admission nor its subsequent conversion input. No external observability of the third read is implied.

4. **Admission rejection.** Selector `0x2804` takes the not-below path and is rejected after two admission-pivot reads, without conversion or parameter-object lookup. A zero error becomes `0x0500`.

5. **Invalid target after conversion.** Selector `0x2800` with target zero is admitted and converted before rejection. Parameter-object lookup does not occur.

6. **Single-byte success and failure.** For selector `0x2800`, converted `0x2601` writes `2` at `0x105`. Converted `0x2700` instead rejects without a parameter-byte write.

7. **Stable-base two-byte mapping.** Selector `0x2801` produces the following ordered pairs at `0x104`, then `0x106`:

   | Converted value | First byte | Second byte |
   |---:|---:|---:|
   | `0x2600` | `1` | `0` |
   | `0x2601` | `2` | `0` |
   | `0x2700` | `1` | `1` |
   | `0x2701` | `2` | `1` |
   | `0x2702` | `1` | `2` |
   | `0x2703` | `2` | `2` |

8. **Secondary-conversion failure.** Selector `0x2803` with F-63f3c76ed3e0 returning `-1` triggers sticky error recording and no direct write at `0x108`.

9. **Successful truncation.** Selector `0x2802` takes dispatch equality. A secondary-conversion result of `0x100` writes `0x00` at `0x107`.

10. **Full-width Boolean test.** Selector `0x8191` with converted value `0x100` stores true at context offset `0x90A`, without secondary conversion.

11. **Existing error.** With error `0x0502`, a valid update still occurs. A rejection also leaves that existing error unchanged.

12. **Equality overrides fixed-selector meaning.** Set both pivots to stable `0x8191`. That selector takes dispatch equality, not the Boolean branch. A secondary-conversion result of `2` writes `2` at `0x107`.

13. **Successful second-component fallback.** For selector `0x2801` and converted value `0x2602`, retain early snapshot `0x2601` and all threshold-producing base reads `0x2601`, but let the later first-component base comparison read `0x2602`. The function writes `2` at `0x104`, then `0xFF` at `0x106`, without error recording.

## Uncertainties

The following remain documented_only:

- Actual global values, initialization, stability, and relationships.
- Volatility and external observability of represented global reads.
- Supplied-value interpretation and dependency conversion rules.
- Dependency effects, ownership, lifetime, address guarantees, and abnormal exits.
- Signed-overflow semantics for threshold additions and selector subtraction.
- Pointer model, address arithmetic, alignment, and memory-fault behavior.
- Physical Boolean byte encoding.
- Aliasing, concurrency, reentrancy, atomicity, and visibility guarantees.
- Any surrounding restrictions on valid arguments or object layouts.

Inferred texture-related intent does not resolve these unknowns or justify substituting familiar API validation and error behavior.

## Evidence

Established static evidence is limited to the supplied function representation: input widths, dependency order, comparisons, represented global-read placement, accessed offsets, direct writes, and shared rejection handling.

That representation includes a third admission-pivot read after the below-pivot comparison succeeds and before fixed-selector checks, including for selector `0x2800`. Its value does not determine admission. This establishes read placement in the representation, not a volatile or externally observable access guarantee.

Comments, removed-block notices, prior prose, and reviewer assertions are not independent behavioral evidence. The tables and examples above are deductions from the represented control and data dependencies under their stated assumptions.

No runtime tests have been performed. Dependency implementations, actual global values, and platform execution semantics have not been independently established.
