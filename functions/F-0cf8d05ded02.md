<!-- Trusted provenance: F-0cf8d05ded02; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0cf8d05ded02

## Purpose

F-0cf8d05ded02 conditionally prepares geometry and submits object-associated data to a target. It selects between direct-source submission and a three-value cache path that may allocate storage and invoke a conversion dependency.

The established contract consists of memory accesses, dependency calls, branch conditions, and explicit state changes. “Geometry,” “format,” “allocation,” “conversion,” and “submission” describe inferred roles, not independently established external API semantics.

The function has no explicit return value or status result. Several unsupported-format paths return after publishing shared state; one can also return after populating all three cache fields. Those returns do not perform the normal final clearing operation.

## Interface

### Object argument

The sole argument is a 32-bit address-like value identifying an object. No null, bounds, or storage-validity checks are visible.

| Object offset | Width | Established use |
|---|---:|---|
| `0x20`, `0x24` | 32 bits | Subtrahends for conversion width and height. |
| `0x28`, `0x2C` | 32 bits | Minuends for conversion width and height. |
| `0x30`, `0x34` | 32 bits | Horizontal and vertical placement, interpreted with signed comparisons. |
| `0x3F` | 8 bits | Argument to F-460b2abbbbbe on selected paths. |
| `0x50` | 32 bits | Direct-source value captured before category selection. |
| `0x54` | 32 bits | Enable gate: zero suppresses processing. |
| `0x5C` | 32-bit address | Source-record address. |
| `0xF4` | 32 bits | First target-lookup argument. |
| `0xF8` | 8 bits | Selects clipping bounds by zero versus nonzero. |
| `0xFC` | 32 bits | Target selector; only its low eight bits are used. |
| `0x100` | 32 bits | Mode; equality to `4` selects special handling. |
| `0x104` | 32 bits | First cache value and sole cache-initialization gate. |
| `0x108`, `0x10C` | 32 bits | Second and third cache values. |

The source record provides:

| Source offset | Width | Established use |
|---|---:|---|
| `0x04` | 32 bits | Source value supplied to conversion. |
| `0x14` | 16 bits, unsigned | Format identifier, zero-extended before comparison. |
| `0x98` | Embedded-region address | Argument to both dimension queries. |

Reached fields must be readable, and cache fields must be writable when initialization occurs. Referenced storage and shared manager storage must be valid for the selected path. These are preconditions, not enforced validation.

### Shared state and captured values

Relevant shared state comprises a manager address and two 32-bit format constants, designated **format A** and **format B**.

The manager address is read at entry, even for a disabled object. Publication uses a separate, fresh manager-address read. Normal final clearing uses the entry-captured address.

After descriptor initialization, the function captures, in order:

1. Shared format B.
2. Shared format A.
3. The object’s direct-source value at `0x50`.

These captured format values are used for later conversion and direct-submission dispatch. They are **not** used for initial descriptor-category selection.

For initial category selection in a mode other than `4`, the function reads the source identifier, then freshly reads shared format A for comparison. Only if that comparison fails does it freshly read shared format B. Initial category selection can therefore disagree with later dispatch even if the source identifier remains unchanged.

All format comparisons compare a zero-extended unsigned 16-bit identifier with a full 32-bit value. Values outside `0` through `65535` cannot match.

### Temporary descriptor

The function passes a stack-resident descriptor to F-abc658b0bf5e, then explicitly writes selected fields.

| Descriptor offset | Width | Explicitly supplied value |
|---|---:|---|
| `0x00` | 8 bits | Category: `0`, `2`, or `3`. |
| `0x01` | 8 bits | Constant `3`. |
| `0x04` | 32 bits | Destination horizontal coordinate. |
| `0x08` | 32 bits | Destination vertical coordinate. |
| `0x0C` | 32 bits | Clipped width. |
| `0x10` | 32 bits | Clipped height. |
| `0x14` | 32 bits | Result of F-19fde5f06ccc. |
| `0x18` | 32 bits | Result of F-b01436b9ce2b. |
| `0x24` | 32 bits | Vertical source displacement. |
| `0x28` | 32 bits | Horizontal source displacement. |
| `0x30` | 16 bits | Zero. |

The complete descriptor size, packing, and meanings of other bytes are not established. Unwritten bytes depend on initialization and any dependency effects.

## Behavior

### Entry, publication, and initialization

The function first captures the manager address, then tests the object’s enable field.

If the enable field is zero, it returns. It makes no dependency calls and performs no explicit writes, including no clearing of manager state.

Otherwise, the observable sequence is:

1. Read the shared manager address again and store the object address at manager offset `0x14`.
2. Call F-5b4ae75440ab with the object’s `0xF4` value and its `0xFC` value masked to eight bits.
3. Retain the returned 32-bit target value.
4. Call F-abc658b0bf5e with the descriptor address.
5. Capture format B, format A, and the direct-source value, in that order.

Target lookup and descriptor initialization precede all format rejection.

### Initial category selection

The function reads the mode at `0x100`.

- If it equals `4`, write descriptor category `0`; no source-format comparison is performed at this stage.
- Otherwise, read the source-record address and its unsigned 16-bit identifier.
  - Compare that identifier with a fresh shared-format-A read. On equality, write category `2`.
  - Only on mismatch, compare it with a fresh shared-format-B read. On equality, write category `3`.
  - If both comparisons fail, return immediately.

Thus, the fresh format-A comparison has precedence. The captured constants remain reserved for later dispatch.

This unsupported-format return leaves publication in place and performs no dimension calls or final manager clearing.

### Descriptor writes at dimension-call boundaries

After writing the category, the function rereads the source-record address and computes the embedded-region address at `0x98`. It then performs these ordered operations:

1. Write descriptor byte `0x01` to `3`.
2. Call F-19fde5f06ccc with the computed embedded-region address.
3. Store that call’s 32-bit result at descriptor offset `0x14`.
4. Call F-b01436b9ce2b with the same retained embedded-region address.
5. Store that call’s 32-bit result at descriptor offset `0x18`.

At the first dimension-call boundary, category and byte `0x01` have already been explicitly written; neither dimension-result store has occurred. At the second boundary, the first dimension-result store has occurred, but the second has not.

These are ordered writes, not an unordered descriptor-population phase. Because F-abc658b0bf5e received the descriptor address, its retention behavior cannot be assumed harmless. Subsequent dependencies might observe or modify the descriptor through an escaped reference; no preservation guarantee is established.

### Placement and clipping

After both dimension-result stores, horizontal placement is read and written into descriptor offset `0x04`.

- If the read value is negative, the destination coordinate is overwritten with zero. The object’s horizontal placement is read again, negated, and stored as horizontal source displacement.
- Otherwise, horizontal source displacement is set to zero.

Vertical placement is then handled analogously, using offsets `0x08` and `0x24`.

For stable fields, no intervening descriptor mutation, and representable signed arithmetic:

- Initial visible width is the first dimension result minus horizontal source displacement.
- Initial visible height is the second dimension result minus vertical source displacement.

Width is stored before height. The viewport-selector byte is then read:

| Selector | Horizontal bound | Vertical bound |
|---|---:|---:|
| Zero | 720 | 576 |
| Nonzero | 320 | 240 |

Horizontal clipping precedes vertical clipping. For each axis, a signed comparison checks whether destination coordinate plus extent exceeds the bound. If so, the extent is replaced by bound minus destination coordinate.

There is no explicit zero clamp. A destination beyond a bound can produce a negative extent; a source displacement larger than the original dimension can also produce one. Equality with the bound does not trigger replacement.

After clipping, the function writes zero to descriptor offset `0x30` as a 16-bit store, then calls:

1. F-dcb90cca1e28 with the retained target and descriptor address.
2. F-c9bd3b9286da with the retained target and literal zero.

### Conversion/cache path

After those calls, the function rereads mode. This second read independently determines the submission path.

If mode equals `4`, it reads the first cache field at `0x104`. Only zero triggers initialization.

#### Cache initialization

The allocation-size calculation uses:

- Width difference: object `0x28` minus object `0x20`.
- Height difference: object `0x2C` minus object `0x24`.
- Their product, assigned to a 32-bit unsigned size value.

The established order is:

1. Call F-638beded7699 with the full size and `0x35`.
2. Logically shift the retained unsigned size right by two bits.
3. Store the first return value at `0x104`.
4. Call F-638beded7699 with the shifted size and `0x35`; store its return at `0x108`.
5. Call F-638beded7699 with that same shifted size and `0x35`; store its return at `0x10C`.

The quarter-size requests round down. There is no visible alignment adjustment, size validation, or allocation-result check.

The source-record address and identifier are then read again. Dispatch precedence is:

| Identifier comparison | Selected dependency |
|---|---|
| Literal `0x65` | F-9e62624e89e5 |
| Otherwise, captured format A | F-28199ab2af9a |
| Otherwise, captured format B | F-c0ab5b5d32b8 |
| None match | Immediate return |

Each conversion receives six arguments:

1. Height difference.
2. Width difference.
3. The newly selected source record’s `0x04` value.
4. The currently read first cache value.
5. The currently read second cache value.
6. The retained return value from the third allocation call.

Conversion dimensions are recomputed from object fields; allocation-time dimensions are not reused. The sixth argument is not a reread of `0x10C`.

Unsupported conversion format returns after all three allocation calls and stores, without submission or final manager clearing.

#### Cached submission

If the first cache value was already nonzero, allocation, conversion-format checking, and conversion are all skipped. The other cache values are not independently validated.

Both initialized and previously cached paths call, in order:

1. F-25dd705719a1 with the target.
2. F-460b2abbbbbe with the target and a current read of byte `0x3F`.
3. F-307b5e43b9f5 with the target, current reads of all three cache fields, and literal zero.

### Direct submission

If the second mode read is not `4`, the function rereads the source identifier and compares it with the **captured** format values.

- Captured format A matches: call F-25dd705719a1, then F-460b2abbbbbe with the target and current byte `0x3F`.
- Otherwise, captured format B matches: call F-f3f14c3b9ccd with the target.
- Neither matches: return without final manager clearing.

After either supported branch, call F-1d891874186d with the target and the direct-source value captured before category selection.

### Normal completion

Following either completed submission path, store zero at offset `0x14` of the entry-captured manager.

This overwrites rather than restores the previous value. Publication and clearing can address different managers if the shared manager address changes.

## Side effects

Explicit writes comprise manager publication, descriptor initialization and population, up to three object-cache stores, and normal-completion manager clearing.

Every dependency may have additional effects. The evidence does not establish preservation of globals, object fields, descriptor contents, or referenced memory across calls.

The function contains no explicit release operation for the target or allocation-like results. Cache stores establish persistence of values in the object, not ownership transfer or release responsibility.

Descriptor storage is local to this invocation. Whether dependencies copy, retain, or asynchronously access its address is unknown. No persistent replacement storage is explicitly allocated here.

No diagnostic, rollback, status return, or ordinary-return failure check is visible. An allocation-like return of zero is still stored and passed onward; later invocations gate initialization solely on the first cache field.

## Dependencies

All identifiers below are opaque. Descriptions establish call-site interactions only.

| Dependency | Interaction |
|---|---|
| F-5b4ae75440ab | Target lookup using a 32-bit value and low-eight-bit selector. |
| F-abc658b0bf5e | Descriptor initialization before explicit field writes. |
| F-19fde5f06ccc | First dimension query; return stored before the second query. |
| F-b01436b9ce2b | Second dimension query using the same computed region address. |
| F-dcb90cca1e28 | Receives target and populated descriptor address. |
| F-c9bd3b9286da | Receives target and zero immediately afterward. |
| F-638beded7699 | Three allocation-like requests during initialization. |
| F-9e62624e89e5 | Six-argument conversion for literal identifier `0x65`. |
| F-28199ab2af9a | Six-argument conversion for captured format A. |
| F-c0ab5b5d32b8 | Six-argument conversion for captured format B. |
| F-25dd705719a1 | Target operation on cached submission and direct format A. |
| F-f3f14c3b9ccd | Target operation on direct format B. |
| F-460b2abbbbbe | Receives target and the current one-byte object value. |
| F-307b5e43b9f5 | Receives target, three current cache values, and zero. |
| F-1d891874186d | Receives target and the captured direct-source value. |

## Acceptance examples

These are conditional static predictions, not executed tests. Unless an example specifies mutation, assume valid storage, normally returning dependencies that do not modify relevant state, stable shared values, and representable arithmetic.

1. **Disabled object.** Enable zero causes only the entry manager capture and enable test among the described operations. No dependency is called and no explicit write occurs.

2. **Initial unsupported format.** Enabled, non-`4` mode and an identifier matching neither freshly read constant cause publication, target lookup, and descriptor initialization, followed by return. No dimension query occurs, and publication is not cleared.

3. **Dimension-boundary visibility.** If the initializer retains a usable descriptor reference without modifying it later, the first dimension dependency can observe category and byte `0x01 = 3`. The second can additionally observe the first dimension result at `0x14`; this invocation has not yet stored its second dimension result at `0x18`.

4. **Fresh versus captured formats.** Suppose captured A is `17`, captured B is `34`, and the unchanged source identifier is `34`. If shared A becomes `34` before the initial fresh comparison, category is `2` and shared B is not read for that comparison. If later mode is non-`4`, direct dispatch nevertheless selects captured format B and calls F-f3f14c3b9ccd.

5. **Negative placement.** Dimensions 100 by 80 and placement −10 by −20 produce destination `(0, 0)`, source displacements 10 and 20, and extents 90 by 60.

6. **Reduced viewport.** A nonzero viewport selector, dimensions 100 by 100, and placement `(300, 220)` produce extents 20 by 20.

7. **No zero clamp.** With reduced bounds, width 10 and horizontal placement 330 yield horizontal extent −10. Descriptor delivery still occurs.

8. **Allocation rounding.** Mode `4`, zero first cache, and differences 5 by 3 produce allocation requests 15, 3, and 3, each with `0x35`.

9. **Unsupported conversion.** Mode `4`, zero first cache, and an identifier matching neither `0x65` nor either captured constant cause all three allocation requests and stores before return. No conversion or submission occurs; publication remains uncleared.

10. **Existing cache bypass.** Mode `4` and nonzero first cache skip allocation and conversion even with an unsupported source identifier or zero remaining cache values. Submission uses current reads of all three cache fields.

11. **Precedence.** Equal captured constants favor format A in later dispatch. During new conversion, literal `0x65` takes precedence over either captured constant.

## Uncertainties

The following remain **documented_only**:

- **Overflow:** signed minimum-value negation, overflowing differences, coordinate sums, and dimension products lack an established portable overflow contract. The later unsigned logical shift does not resolve earlier signed-overflow behavior.
- **Mutation and reentrancy:** dependencies may change reachable state. Mode reads can disagree; category uses fresh constants while dispatch uses captured constants. No synchronization guarantee is established.
- **Descriptor aliasing:** retained descriptor access could alter values used after dimension calls. Ordered explicit writes do not establish dependency preservation.
- **Manager identity:** publication and final clearing can affect different storage.
- **Failure and ownership:** allocation failure conventions, capacity requirements, target validity, release obligations, and nonlocal exits are unknown.
- **Descriptor semantics:** downstream handling of negative extents, unwritten bytes, and retained addresses is unknown.
- **Argument-read ordering:** dependency-call order and the explicit boundaries described above are established; independent argument reads within a call do not have an established relative order.

## Evidence

This contract derives from the supplied static representation of F-0cf8d05ded02 and the supplied opaque dependency mapping. No runtime tests have been performed.

Established evidence includes access widths and offsets, separate captured and fresh format reads, conditional format-B reads, descriptor stores surrounding dimension calls, branch precedence, cache gating, call order, and cleanup-bypassing returns.

Image-processing terminology expresses inferred intent. Unknown dependency behavior, ownership rules, concurrent behavior, and arithmetic outside representable cases are not promoted to guarantees.
