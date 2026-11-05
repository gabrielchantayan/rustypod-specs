<!-- Trusted provenance: F-47f33f8d1eca; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-47f33f8d1eca

## Purpose

F-47f33f8d1eca conditionally resets selected fields of a caller-supplied memory object.

A four-byte field at offset `0x30` controls entry. If that field contains zero, the function returns without calling its dependency or performing any direct write. Otherwise, it calls F-da8ea439d275, performs four reset writes, reads two primary-object fields, and writes two four-byte outputs.

With valid storage, a normally returning dependency, and no intervening changes to the primary object, the active path leaves:

- Four-byte zero values at `0x30`, `0x38`, and `0x3C`.
- Four-byte all-one patterns at `0x40` and `0x44`.
- A zero byte at `0x59`.

The supplied representation also contains a conditional calculation using a secondary object. Selection depends on a reread of `0x30` after that same field has been cleared. Ordinary stable execution therefore does not select this calculation. Its represented behavior is specified without asserting that a supported execution mechanism can reach it.

Deactivation of an association and resetting associated range or status information are **inferred intent**, not established domain semantics.

## Interface

### Input and primary layout

The function accepts one object-base argument represented through a 32-bit integer-shaped address carrier. A public object type, calling convention, and complete address-model contract are not supplied.

All offsets below are byte offsets from that base.

| Offset | Access width | Established use |
|---|---:|---|
| `0x30` | 4 bytes | Entry gate, reset destination, and later reread used as a secondary-object address when nonzero |
| `0x34` | 4 bytes | Unsigned input from which a half-width is derived |
| `0x38` | 4 bytes | First final output |
| `0x3C` | 4 bytes | Second final output |
| `0x40` | 4 bytes | Reset destination receiving `0xFFFFFFFF` |
| `0x44` | 4 bytes | Reset destination receiving `0xFFFFFFFF` |
| `0x59` | 1 byte | Reset destination receiving zero |

This table specifies accesses, not a complete object declaration. The meaning and required contents of intervening bytes are unknown.

### Conditional secondary layout

A nonzero post-reset reread of `0x30` supplies the base for two secondary reads:

| Secondary offset | Access width | Arithmetic interpretation |
|---|---:|---|
| `0x2B4` | 4 bytes | Signed 32-bit center-like input |
| `0x2B8` | 4 bytes | Signed 32-bit bound-like input |

“Center” and “bound” name arithmetic roles only. They do not establish coordinate units, container sizes, or valid domain ranges.

### Outputs and errors

No explicit return value or status result is supplied. Observable results consist of direct memory writes and whatever effects F-da8ea439d275 produces.

There is no visible validation failure path, error report, or recovery mechanism. Behavior after an invalid access or abnormal dependency exit is **documented_only**.

### Preconditions

Every invocation requires a readable four-byte entry field at `0x30`. If that field is zero, no other primary or secondary field is explicitly accessed.

An active invocation additionally requires:

- Writable storage at all six direct write destinations.
- Readable storage at `0x30` and `0x34` when the post-reset reads occur.
- Continued validity of the primary object after the dependency returns.
- Readable secondary fields if the conditional secondary branch is selected.
- Address calculations and alignment compatible with the execution environment.

No null-base check, allocation-extent check, secondary-address validation, or alignment recovery is shown. A zero base is not specially accepted; the entry access would still be attempted at its corresponding offset.

No explicit operation transfers ownership.

## Behavior

### Entry gate

The first explicit memory access reads four bytes at primary offset `0x30`.

If the value is zero, the function returns immediately:

- F-da8ea439d275 is not called.
- No direct write occurs.
- `0x34` is not read.
- Neither secondary field is read.

Every nonzero entry pattern selects the active path. This test establishes neither address validity nor any domain-specific state. The original entry value is not retained as the address for the later secondary reads; those reads depend on a separate post-reset reread.

### Dependency call and reset writes

On the active path, F-da8ea439d275 is called once. No explicit arguments are shown, and no result from the call is used.

After normal return, the function performs these writes in the represented order:

1. Four-byte `0xFFFFFFFF` at `0x40`.
2. Four-byte zero at `0x30`.
3. Four-byte `0xFFFFFFFF` at `0x44`.
4. One-byte zero at `0x59`.

The all-one values are established bit patterns. Their interpretation as negative-one sentinels, invalid indices, or other status values is not established.

Neither the object base nor the original entry value is visibly passed as an explicit dependency argument. This does not exclude implicit inputs or shared-state access.

If the dependency does not return normally, the subsequent direct writes are not established to occur.

### Post-reset reads

After the four reset writes, the function performs two primary reads in this order:

1. Four bytes at `0x30`, captured for the branch and possible secondary address.
2. Four bytes at `0x34`, interpreted as unsigned and shifted right by one bit.

The second read occurs even when the captured reread is zero.

The shifted quantity is called the **half-width** here. It is the unsigned width input divided by two with any remainder discarded. Its range is zero through `2,147,483,647`, so its signed 32-bit interpretation is nonnegative.

The branch uses the captured reread. No additional read of `0x30` is shown before secondary access.

### Zero post-reset reread

If the captured reread is zero:

- No secondary field is read.
- The first output is zero.
- The second output is zero.

The function writes the first output to `0x38`, then the second to `0x3C`.

This is the ordinary stable path because the function has just written zero to `0x30`. A dependency change to that field before returning is insufficient to select the secondary branch: the later direct clear overwrites it.

### Nonzero post-reset reread

If the captured reread is nonzero, it is treated as a secondary-object base. The function reads the signed center at `0x2B4`, then the signed bound at `0x2B8`. Both reads occur before either final output write.

The first output is:

- Zero when the center is less than or equal to the half-width.
- The center minus the half-width when the center is strictly greater.

The selected subtraction yields a positive value within signed 32-bit range.

For the second output, the function first forms a candidate from the center plus the half-width, subject to the arithmetic qualifications below. Once a signed candidate is available, it applies these rules:

| Relationship | Second output |
|---|---|
| Bound greater than candidate | Candidate |
| Bound less than or equal to candidate, and bound zero | Zero |
| Bound less than or equal to candidate, and bound nonzero | Bound minus one |

Consequently, a zero bound does not necessarily force a zero output. If conversion produces a negative candidate, a zero bound is greater than that candidate and the candidate is retained. Whether a particular unsigned sum becomes such a negative candidate can remain unresolved.

The first output is written to `0x38`, followed by the second output at `0x3C`.

There is no final ordering check between the outputs. Negative secondary inputs are not rejected. The first output can exceed the second.

### Integer widths and arithmetic boundaries

The gate, address-bearing field, width input, secondary inputs, and final outputs are all four bytes wide in the supplied representation.

The width reduction is unsigned. For example, width input `0xFFFFFFFF` produces half-width `2,147,483,647`, not a negative value.

The first-output comparison uses the nonnegative signed interpretation of the half-width. Although the subtraction combines the signed center and unsigned half-width, its selection condition ensures a positive result no greater than the signed maximum. That selected result therefore has no out-of-range signed conversion.

The center-plus-half-width calculation uses unsigned 32-bit arithmetic:

- The center participates through its unsigned 32-bit value.
- Addition is reduced modulo \(2^{32}\).
- The unsigned sum is then converted to a signed 32-bit candidate.

An unsigned sum no greater than `2,147,483,647` is representable as a signed 32-bit value. For larger unsigned sums, the applicable conversion behavior is **documented_only**. The supplied evidence does not justify an unconditional two’s-complement reinterpretation.

This limitation applies even when the mathematical addition itself appears harmless. With center `−3` and half-width zero, the unsigned sum is `0xFFFFFFFD`. Converting that sum to the signed candidate remains subject to the unresolved conversion rule; adding zero does not remove the uncertainty.

The bound decrement has a separate boundary. If the bound is `−2,147,483,648` and its decrement branch is selected, the mathematical result is outside signed 32-bit range. The effective outcome is **documented_only**; wrapping, trapping, and normal continuation are not established.

Neither arithmetic uncertainty affects the zero-reread path.

## Side effects

### Direct writes

A normally completing active invocation performs six direct writes to six distinct destinations:

| Order | Primary offset | Width | Value |
|---:|---|---:|---|
| 1 | `0x40` | 4 bytes | `0xFFFFFFFF` |
| 2 | `0x30` | 4 bytes | Zero |
| 3 | `0x44` | 4 bytes | `0xFFFFFFFF` |
| 4 | `0x59` | 1 byte | Zero |
| 5 | `0x38` | 4 bytes | First output |
| 6 | `0x3C` | 4 bytes | Second output |

The first four writes precede the post-reset reads. The final two follow all reads used to determine their outputs.

Only one byte is directly cleared at `0x59`; adjacent bytes are not included in that write. No direct write targets `0x34` or either secondary field.

These statements describe this function’s direct effects. They do not guarantee preservation of other memory because the dependency’s effects are unknown. Final retained values also require absence of subsequent interference.

### Ownership and lifetime

No explicit allocation or release occurs in this body. Clearing the address-bearing field does not establish that any associated resource has been released.

The dependency may affect memory, ownership, or object lifetime. Its unavailable contract must not be replaced with an assumption that the primary object, secondary object, globals, or allocations are preserved.

### Ordering and synchronization

The stated order is the order expressed by the supplied representation. No atomic-access qualification, lock, memory barrier, or synchronization protocol is shown.

The reset is not established as atomic. External mutation is not established as a supported means of reaching the nonzero-reread branch. Other observers are not guaranteed a synchronized view of the listed accesses.

## Dependencies

### F-da8ea439d275

Established call behavior:

- Called exactly once if the entry field is nonzero.
- Not called if the entry field is zero.
- Receives no explicit arguments in the supplied representation.
- Returns before the first direct write on an ordinarily continuing active path.
- Supplies no result used by F-47f33f8d1eca.

Its purpose, implicit inputs, global effects, ownership behavior, failure behavior, and normal-return guarantees are **documented_only**.

No preservation or release contract is inferred from the absence of explicit arguments.

## Acceptance examples

These are conditional criteria derived from static evidence, not executed tests. Secondary-calculation examples assume an otherwise valid execution supplies a nonzero post-reset reread; they do not establish a supported way to reach that state.

### Inactive entry

Given a readable entry field containing zero:

- The dependency is not called.
- No other listed field is explicitly read.
- No direct write occurs.

Other fields need not already contain reset values.

### Ordinary active reset

Given a nonzero entry value, valid storage, a normally returning dependency, and no intervening changes after the reset writes:

- The dependency is called once before every direct write.
- `0x40` and `0x44` receive `0xFFFFFFFF`.
- `0x30` and the byte at `0x59` receive zero.
- `0x34` is read despite not affecting the zero-reread outputs.
- `0x38` and `0x3C` receive zero.
- No secondary field is read.

### Dependency changes at reset destinations

If the dependency changes any reset destination and returns with the primary object still valid, the subsequent direct writes overwrite those destinations as specified, absent later interference.

This criterion places no restriction on dependency changes elsewhere.

### Interior calculation

With width input `8`, center `10`, and bound `100`:

- Half-width is `4`.
- First output is `6`.
- The unsigned sum is `14`, which is representable as the signed candidate.
- Second output is `14`.

### Equality boundaries

With width input `8`, center `4`, and bound `8`:

- Half-width is `4`.
- First output is zero because the lower comparison is strict.
- Candidate is `8`.
- Second output is `7` because equality selects the bound decrement.

### Zero bound with a representable nonnegative candidate

With width input `8`, center `10`, and bound zero:

- First output is `6`.
- Candidate is `14`.
- Second output is zero, not `−1`.

This demonstrates that the outputs need not be ordered.

### Zero bound with a negative center

With width input zero, center `−3`, and bound zero:

- Half-width is zero.
- First output is zero.
- The unsigned addition result is `0xFFFFFFFD`.
- That result exceeds the signed maximum, so the candidate and resulting second output are not unconditionally determined by the supplied conversion evidence.

**Conditional refinement:** If the applicable conversion maps `0xFFFFFFFD` to signed `−3` and execution continues normally, the candidate is `−3`. The zero bound is greater than that candidate, so the second output is `−3`.

The refinement is conditional on the conversion rule, not an unconditional acceptance requirement.

### Unsigned high-bit width input

With width input `0xFFFFFFFF`, center zero, and bound `2,147,483,647`:

- Half-width is `2,147,483,647`.
- First output is zero.
- Candidate is `2,147,483,647`, without an out-of-range conversion.
- Second output is `2,147,483,646`.

This distinguishes the unsigned width shift from a signed shift.

## Uncertainties

- **documented_only:** Complete object types, storage extents, alignment requirements, byte order, and domain meanings.
- **documented_only:** Address arithmetic and the relationship between the 32-bit address carrier and the execution environment.
- **documented_only:** Dependency implementation, implicit inputs, shared-state changes, lifetime effects, ownership effects, and abnormal exits.
- **documented_only:** Any supported mechanism for obtaining a nonzero reread immediately after the explicit clear.
- **documented_only:** Conversion of unsigned sums above `2,147,483,647` into the signed candidate, including negative-center cases with zero half-width.
- **documented_only:** Effective behavior of a selected decrement at the signed minimum.
- **documented_only:** Invalid-access, hardware-fault, exception, and recovery behavior.
- **Inferred intent:** Deactivation and reset of association, range, or status state.
- **Not established:** Thread safety, atomicity, reentrancy, resource release, or idempotence of dependency effects.

## Evidence

Established claims come from the supplied static function representation: an entry read and guard, one opaque dependency call, four initial writes, two primary rereads, conditional secondary reads and arithmetic, and two final writes.

The unsigned width and mixed signed/unsigned arithmetic establish the conversion boundary described above. In particular, a negative center plus zero half-width still produces an unsigned sum requiring conversion before the signed comparison.

The supplied material provides no dependency body, public object declaration, caller contract, conversion-environment specification, or runtime observations. Comments about omitted material do not establish additional behavior.

All access-order claims describe the supplied representation, not a verified hardware trace. No runtime tests have been performed.
