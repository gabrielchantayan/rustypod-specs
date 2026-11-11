<!-- Trusted provenance: F-da275d1ce8ce; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-da275d1ce8ce

## Purpose

F-da275d1ce8ce accepts a 32-bit input word and produces a nominal 32-bit result through classification, delegation, shared-storage reads, word conversion, and exceptional indirect dispatch.

Its established behavior includes:

- Direct handling of two distinguished input encodings.
- Delegation to F-90d91794378e for inputs whose bits 23–30 are not all ones.
- Conditional notification through F-77dd9f40f9b6.
- Two distinct sentinel reads on the nonzero-shaped delegated-result path.
- Separately located reads of two intermediate words on the zero-shaped delegated-result path.
- Conversion of those words into one result word.
- An unresolved indirect continuation for certain exceptional inputs or intermediate values.

The layouts resemble binary32 input/output and a binary64 intermediate. This is inferred intent, not a numerical contract: the delegated operation is unknown, and some visible results differ from conventional floating-point conversion behavior. The contract therefore specifies bit patterns and observable operations.

## Interface

### Explicit inputs

The supplied interface has four 32-bit argument positions.

| Position | Established use |
|---|---|
| First | Input word used for classification and passed to F-90d91794378e when applicable. |
| Second | No visible use. |
| Third | No visible use. |
| Fourth | No effective influence on a reachable result in the supplied body. On the all-ones classification path, it is overwritten before relevant use. |

These observations do not establish whether an unavailable machine-level calling convention or indirect continuation can observe additional incoming state.

### Result

The nominal result is an unsigned 32-bit word, obtained from one of these sources:

- A direct constant or bitwise construction.
- The result returned by F-90d91794378e.
- The first saved sentinel value, following a successful comparison against a separately read sentinel value.
- The result of an unresolved indirect transfer, if that transfer returns normally.

Normal return is not established for every dependency or indirect transfer.

### Widths and layouts

Word operations use 32-bit values. Addition and left shifts retain the low 32 bits; complements cover all 32 bits. Right shifts described below are logical unless explicitly identified otherwise. Signed comparisons interpret a 32-bit pattern as a two’s-complement signed integer.

The input fields are:

- Bit 31: sign-shaped bit.
- Bits 23–30: eight-bit classification field.
- Bits 0–22: fraction-shaped field.

The intermediate consists of separately read lower and upper 32-bit words. The upper word supplies the result sign from bit 31. “Lower” and “upper” describe their roles in conversion, not a guarantee that the reads come from one stable object.

The supplied addressing uses a shared locator and two additive offsets differing by four bytes. It does not establish portable byte order, alignment requirements, or a coherent 64-bit load.

### Implicit inputs and preconditions

Relevant implicit state includes:

- Sentinel storage.
- Shared locator storage.
- Payload storage addressed through that locator.
- Exceptional-dispatch companion and control storage.
- Incoming link-register-derived target state.
- An incoming status bit used by one dispatch form.

For normal execution, every reached read and transfer must be valid under the execution environment. Initialization, lifetime, access permissions, concurrency guarantees, and target validity are not established. There is no visible validation that supplies safe behavior when these requirements are unmet.

## Behavior

### Input classification

If bits 23–30 are all ones:

1. Input `0xff800000` returns `0x00000000` immediately.
2. For every other input in this class, the reachable path prepares dispatch control `0x0000000b` and a second companion value of `1`, then reads a shared exceptional-data word as the first companion.
3. If bits 0–22 are zero, the original input is returned.
4. Otherwise, execution enters indirect dispatch.

Accordingly, `0x7f800000` returns unchanged, but only after the exceptional-data read. Input `0xff800000` returns before that read. Every all-ones-class input with a nonzero fraction-shaped field reaches dispatch with low control nibble `0xb`.

The supplied body contains additional conditional alternatives inside this branch. Under the established classification condition and the shown flag calculations, they do not provide additional reachable input cases.

If bits 23–30 are not all ones, execution follows the delegated path below.

### Delegation and sentinel handling

F-90d91794378e is called with the input word and a second argument of zero.

After that call returns, sentinel storage is read once and the value is saved. This first read occurs regardless of whether the delegated result is zero-shaped.

A result is **zero-shaped** when all bits except bit 31 are zero: either `0x00000000` or `0x80000000`.

For a result that is not zero-shaped:

1. Sentinel storage is read a second time for comparison with the delegated result.
2. If the delegated result differs from this second value, the delegated result is returned immediately.
3. If they are equal, F-77dd9f40f9b6 is called with `2`, and the first saved sentinel value is returned.

The comparison does **not** use the saved first value. No stability between the two reads is established. Consequently, a successful comparison can lead to returning a word different from both the delegated result and the second sentinel value.

The value returned after notification is the first saved value. Changes to sentinel storage during F-77dd9f40f9b6 do not replace that saved result.

For either zero-shaped delegated result:

1. F-77dd9f40f9b6 is called with `2`.
2. Execution reads the intermediate words as described next.

There is no second sentinel read on this path. The first saved sentinel value is not used to determine its result.

### Intermediate-word reads

After notification on the zero-shaped path, reads occur in this order:

1. Read the shared locator.
2. Read the lower payload word at the address formed by adding `0x08031f14` to that locator value.
3. Read the shared locator again.
4. Read the upper payload word at the address formed by adding `0x08031f18` to this second locator value.

The address additions use 32-bit address arithmetic in the supplied representation.

If both locator evaluations produce the same value, the upper payload address is four bytes beyond the lower payload address, modulo the address width. If they differ, that relationship is not guaranteed.

Neither a single captured locator nor a coherent pair is established. The two words may reflect different locator values or different storage states. Notification precedes both locator evaluations, so any locator or payload changes made by F-77dd9f40f9b6 can affect them.

The sign of the delegated zero-shaped result is discarded. Subsequent sign handling uses the upper payload word.

### Ordinary conversion branch

From the upper word, form:

- A saved sign containing bit 31 only.
- A magnitude-shaped word with bit 31 cleared.
- A probe equal to that magnitude-shaped word XOR the upper word shifted left by one.

The ordinary branch applies when the probe has any bit set under mask `0x7fe00000`.

Add `0xc8000000` to the magnitude-shaped word, retaining 32 bits. Interpret the adjusted value as signed.

If it is less than `0x00100000`, return positive zero. This test also rejects negative signed values. The saved sign is not included in this return.

Otherwise, rounding depends on the lower word:

- Bit 28 is the rounding trigger.
- Bits 0–27 identify nonzero trailing bits.
- Bit 29 resolves the exact-halfway case.

Add `0x20000000` to the lower word exactly when bit 28 is set and either bits 0–27 are nonzero or bit 29 is set. Retain 32 bits. Increment the adjusted upper value by one if the lower addition carries; this occurs when the pre-addition lower word exceeds `0xdfffffff`.

After rounding:

- If the adjusted upper value, interpreted as signed, is less than `0x0ff00000`, return the OR of the saved sign, the adjusted upper value shifted left three, and the lower word shifted right 29.
- Otherwise, return the saved sign OR `0x7f800000`.

The rounding rule is consistent with nearest, ties-to-even at the retained low bit. That interpretation is inferred; the masks, increments, carry condition, and packing define the established behavior.

This branch does not construct gradual-underflow results. Its initial rejection returns positive zero directly.

### Special conversion branch

If the probe has no bits set under `0x7fe00000`, inspect bit 20 of the upper word.

When bit 20 is clear:

- If the lower word is zero and the upper word has no non-sign bits set, return the saved sign.
- Otherwise, return positive zero.

An exact zero-shaped pair therefore preserves sign, while a nonzero pair in this branch does not.

When bit 20 is set:

- If the lower word is zero and bits 0–19 of the upper word are zero, return the upper word AND `0xff8fffff`.
- Otherwise, prepare a first companion value of zero, read the shared second companion, then read the shared dispatch control and enter indirect dispatch.

In particular, upper words `0x7ff00000` and `0xfff00000`, with lower word zero, return `0x7f800000` and `0xff800000`, respectively.

### Indirect dispatch

The transfer target is formed from incoming link-register-derived state by clearing its lowest two bits. Dispatch selects a recovered argument form using the low four bits of the control word.

The underlying dispatch structure is unresolved. The following are the explicit argument forms in the supplied representation, not a complete ABI guarantee.

| Control condition | Recovered explicit arguments |
|---|---|
| Low nibble `9`, bit 20 clear | Zero; second companion |
| Low nibble `9`, bit 20 set, any bit under `0x00070000` set | `8`; the result of shifting control left 15, complementing that 32-bit result, then shifting left once |
| Low nibble `9`, bit 20 set, mask `0x00070000` clear | `0x30000000`; second companion; `0x30000000` OR the low incoming status bit shifted left 27 |
| Low nibble `10`, bit 6 set | `0x80000000` |
| Low nibble `10`, bit 6 clear | First companion |
| Low nibble `8`, bit 6 set | Zero; zero |
| Low nibble `8`, bit 6 clear, bit 4 clear | Newly read shared first companion; existing second companion |
| Low nibble `8`, bit 6 clear, bit 4 set | Zero; newly read shared second companion |
| Any other low nibble | No recovered explicit arguments |

For low nibble `8` with bit 6 clear, the shared first companion is read before bit 4 is tested. If bit 4 is set, that first value is discarded and the shared second companion is then read.

The input-classification exceptional path uses control `0xb`, so it takes the no-recovered-explicit-arguments form. Its prepared companions are not established as explicit transfer arguments.

Each represented transfer returns its resulting word if it returns normally. Whether these are genuine returning calls remains unresolved.

## Side effects

The supplied body shows no direct memory stores, explicit allocation, release, or ownership transfer.

Its visible effects are dependency calls, shared reads, and exceptional indirect transfers. Dependency effects are unknown; no dependency is assumed to preserve globals, retain ownership, or avoid allocation or release.

For a nonzero-shaped delegated result, the required supplied-body ordering is:

1. F-90d91794378e.
2. First sentinel read, saved for a possible return.
3. Second sentinel read, used for comparison.
4. Either immediate return of the delegated result, or F-77dd9f40f9b6 followed by return of the first saved sentinel.

For a zero-shaped delegated result, ordering is:

1. F-90d91794378e.
2. First sentinel read.
3. F-77dd9f40f9b6.
4. First locator read.
5. Lower payload read using that locator.
6. Second locator read.
7. Upper payload read using that locator.

These read occurrences must not be collapsed into a stable sentinel snapshot or a stable locator snapshot without an external guarantee. The evidence establishes neither atomicity nor synchronization. It also does not establish that concurrent modification has well-defined behavior in every possible execution environment.

## Dependencies

### F-90d91794378e

Receives the original input and zero. Its returned 32-bit word selects the direct-return, sentinel-notification, or intermediate-conversion path.

Its mathematical operation, errors, memory effects, ownership effects, and relationship to intermediate storage are unknown. Sentinel reads occur after it returns and can therefore observe state it changes.

### F-77dd9f40f9b6

Receives `2`; its return value is unused.

It is called when:

- A nonzero-shaped delegated result equals the **second** sentinel read; or
- The delegated result is either zero-shaped word.

On the first path, the return remains the **first** saved sentinel value. On the second path, both locator reads and both payload reads occur after notification. The meaning of argument `2` is unknown.

### Indirect dependency: opaque ID unavailable

Target identity depends on incoming machine state rather than a resolved dependency identity. Argument forms vary with dispatch control.

Its complete ABI, state preservation, memory ownership effects, errors, and return guarantees remain documented_only.

## Acceptance examples

These are conditional static consequences, not runtime observations. Payload examples assume execution has reached the conversion path and that the ordered reads produce the stated words.

1. **Direct distinguished inputs.** Input `0xff800000` returns positive zero without either named dependency or the exceptional-data read. Input `0x7f800000` performs the exceptional-data read and returns unchanged.

2. **Exceptional input fraction.** Input `0x7fc00001` reaches indirect dispatch with control `0xb` and no recovered explicit arguments. Its final result requires the continuation’s contract.

3. **Second sentinel read controls rejection.** Suppose delegation returns `0x3f800000`, the first sentinel read is also `0x3f800000`, and the second is `0x40000000`. Return `0x3f800000` without notification. Equality with the first read is insufficient.

4. **Successful comparison returns a different saved word.** Suppose delegation returns `0x3f800000`, the first sentinel read is `0x40000000`, and the second is `0x3f800000`. Notify with `2`, then return `0x40000000`.

5. **Saved return survives later mutation.** Under example 4’s conditions, changing sentinel storage during notification does not change the selected return.

6. **Zero-shaped delegation.** Either `0x00000000` or `0x80000000` requires the first sentinel read, notification, then two locator evaluations interleaved with their payload reads. There is no second sentinel read.

7. **Locator changes.** If the first locator value is zero, the lower address is `0x08031f14`. If the second locator value is `0x100`, the upper address is `0x08032018`. A four-byte relationship cannot be required in this case.

8. **Signed zero and tiny nonzero pair.** Lower zero with upper `0x80000000` returns `0x80000000`. Lower `1` with that same upper word returns positive zero.

9. **Packing and rounding.** With upper `0x3ff00000`, lower words `0`, `0x10000000`, `0x10000001`, and `0x30000000` produce, respectively, `0x3f800000`, `0x3f800000`, `0x3f800001`, and `0x3f800002`.

10. **Threshold behavior.** Upper `0xb8000000` with lower zero returns positive zero. Upper `0xc7f00000` with lower zero returns `0xff800000`.

## Uncertainties

The following remain documented_only:

- The combined mathematical operation.
- Sentinel meaning and stability between reads.
- Locator initialization, stability, and storage ownership.
- Payload coherence, byte order, alignment, and concurrency requirements.
- Whether dependencies populate or relocate intermediate storage.
- The indirect target’s identity and actual dispatch structure.
- Complete calling-convention and incoming-status requirements.
- Attainability of individual shared dispatch-control values.
- Dependency exceptions, nonlocal exits, and nonreturning behavior.
- Behavior for invalid storage or transfer targets.

No explicit error result, invalid-state validation, or recovery mechanism is visible. No safe fallback is assigned to these unresolved conditions.

## Evidence

Established evidence is static: the supplied body’s field tests, constants, word operations, signed comparisons, dependency calls, separate shared-storage references, ordered payload-address evaluations, direct returns, and unresolved indirect transfers.

The sentinel comparison references storage separately from the first saved read. The lower and upper payload expressions separately evaluate the locator, with additive offsets differing by four bytes. Neither storage stability is established.

The simplified all-ones classification behavior and rounding predicate follow from the shown operations. Floating-point terminology describes matching layouts and inferred intent only.

No runtime tests have been performed. Acceptance examples are static consequences under their stated conditions, not measured executions.
