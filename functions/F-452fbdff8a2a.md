<!-- Trusted provenance: F-452fbdff8a2a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-452fbdff8a2a

## Purpose

F-452fbdff8a2a selects an object through an indexed table and conditionally replaces one of two 32-bit fields in that object.

Two selector values are supported:

- Selector **0** replaces the field at object offset **0xA90** with the low 27 bits of the supplied payload.
- Selector **1** replaces the field at object offset **0xB10** with the complete 32-bit payload.

Both supported operations return the 32-bit value **0**. Every other selector returns **0xFFFFFF06** without either field write.

The supplied representation places the indexed object-pointer read before selector validation. Consequently, an unsupported selector does not establish that an invalid table reference is safe.

These are static behavioral findings. The meanings of the objects, fields, and status values are not established.

## Interface

### Inputs

| Input | Width and interpretation | Role |
|---|---|---|
| Index | Signed 16-bit integer | Selects a table slot using a stride of 396 bytes. |
| Selector | Unsigned 32-bit integer | Chooses a field update or the unsupported-selector return path. |
| Payload | Unsigned 32-bit integer | Supplies the replacement field value. |

The index has the representable range **−32,768 through 32,767**. The selector and payload each have the range **0 through 4,294,967,295**.

The interface does not establish how a caller’s wider values are converted into these input widths. Acceptance conditions apply to values after they have acquired the stated widths and signedness.

### Result

The result is a 32-bit value:

| Condition | Result bit pattern |
|---|---|
| Selector is 0 or 1, and execution completes normally | 0x00000000 |
| Selector is any other unsigned 32-bit value, and execution completes normally | 0xFFFFFF06 |

The second bit pattern has unsigned value **4,294,967,046** and signed two’s-complement interpretation **−250**. Its designation as a particular error code is unknown. Consumers must not assume that its signed interpretation is the declared return type.

### Implicit state and layout

The operation uses a shared table-base value. Relative to that base:

- Slot addresses are separated by **0x18C**, or **396 bytes**.
- The selected slot begins at the base plus the signed index multiplied by 396.
- A 32-bit value read from the start of that slot supplies the target object address.

For supported selectors, the target layout includes:

| Object-relative offset | Decimal offset | Access width | Use |
|---|---:|---:|---|
| 0xA90 | 2,704 | 32 bits | Selector 0 replacement |
| 0xB10 | 2,832 | 32 bits | Selector 1 replacement |

The two field starts are **128 bytes apart**. The evidence establishes neither other table-slot contents nor other object fields. A 396-byte stride does not mean that this function reads a complete 396-byte record.

## Behavior

### Target lookup

The function obtains the shared table base and uses the signed index to locate a slot. It reads the object-address value from the beginning of that slot before testing whether the selector is supported.

The index is not range-checked. A negative index addresses a location before the base; it is not rejected or converted into a nonnegative table index by this function. Whether that location belongs to a valid table allocation is an external precondition.

The signed index multiplication itself fits within a signed 32-bit integer over the full input range:

- Index **−32,768** produces displacement **−12,976,128 bytes**.
- Index **32,767** produces displacement **12,975,732 bytes**.

That arithmetic fact does not establish that adding the displacement to a particular base produces a valid address. Address validity, allocation boundaries, and any address-space wraparound remain separate concerns.

### Selector 0

For selector 0, the function replaces the 32-bit field at object offset 0xA90.

The replacement value is the payload masked with **0x07FFFFFF**. Therefore:

- Payload bits 0 through 26 are retained.
- Payload bits 27 through 31 are cleared.
- The stored value is between **0** and **134,217,727**, inclusive.

This is a full 32-bit replacement, not an update that preserves some bits of the old field. The prior field value is not used to construct the replacement.

After the write, the function returns zero.

### Selector 1

For selector 1, the function replaces the 32-bit field at object offset 0xB10 with the payload unchanged.

All 32 payload bits are significant, including the most significant bit. The function does not apply the selector-0 mask on this path and does not use the old field value.

After the write, the function returns zero.

### Unsupported selectors

Selectors **2 through 0xFFFFFFFF**, inclusive, take the unsupported-selector path.

For this path:

- The indexed object-address read still precedes the selector decision in the supplied representation.
- Neither designated object field is read or written by the shown operation.
- The payload does not affect the result.
- Normal completion returns **0xFFFFFF06**.

Because the selector is unsigned, values with their highest bit set belong to this unsupported range. They are not negative selectors.

### Preconditions and abnormal execution

Defined normal operation requires readable storage for the shared base and selected slot. A supported selector additionally requires that the selected object address, plus the chosen field offset, designate writable storage suitable for a 32-bit access.

There are no visible checks for:

- A valid table index.
- A null or otherwise invalid table base.
- A null or otherwise invalid object address.
- Writable destination storage.
- Suitable alignment.

The evidence does not define a recoverable result for failed memory accesses. The unsupported-selector result must not be treated as an invalid-address result. Likewise, the zero result is conditional on reaching normal completion; it is not a guarantee that invalid memory is tolerated.

## Side effects

The shown operation performs one selected-slot read and, for a supported selector, one designated 32-bit field write. Access to the shared base is also required to calculate the slot address.

At the level of the supplied representation, the relevant order is:

1. Obtain the table base and read the selected slot’s object-address value.
2. Decide which selector path applies.
3. For a supported selector, perform the corresponding field replacement.
4. Return the path’s result.

This ordering describes the supplied operation. It does not establish machine-level memory barriers, volatile semantics, atomicity, or synchronization with other execution contexts.

No prior destination-field read is shown. No ordinary second field write, allocation, release, callback, or explicit table update is shown.

The function does not establish ownership of either the table or the selected object. It neither visibly acquires nor relinquishes a reference. Storage lifetime must be maintained externally.

No non-aliasing guarantee is established. If a destination address overlaps table storage, shared state, or another object, the designated write may change that overlapping storage. Statements about an “unchanged other field” therefore require distinct storage.

Repeated supported calls repeat the lookup and replacement. There is no visible cache, accumulated value, or history-dependent branch.

## Dependencies

No other function calls are present in the supplied body. No callable dependency IDs are therefore established.

The operation depends on externally maintained memory:

- The shared table-base value.
- Readable selected-slot storage containing an object address.
- Writable target-field storage for supported selectors.
- Storage lifetime and any synchronization required by callers.

No assumptions about an unknown callee preserving globals or releasing memory are needed: no callee is shown.

## Acceptance examples

These examples are conditional static acceptance cases, not results of executed tests. Supported-selector examples assume valid, suitably accessible storage and no concurrent interference. Examples involving unaffected fields additionally assume nonoverlapping storage.

| Inputs and setup | Required normal behavior |
|---|---|
| Selector 0; payload 0xFFFFFFFF | Replace offset 0xA90 with **0x07FFFFFF**; return zero. |
| Selector 0; payload 0xF8000000 | Replace offset 0xA90 with **0**; return zero. |
| Selector 0; payload 0x01234567 | Replace offset 0xA90 with **0x01234567**; return zero. |
| Selector 0; old destination is 0xFFFFFFFF; payload is zero | Replace the complete field with zero. No old high bits survive. |
| Selector 1; payload 0xFFFFFFFF | Replace offset 0xB10 with **0xFFFFFFFF**; return zero. |
| Selector 1; payload 0x80000000 | Preserve the payload’s high bit in the replacement; return zero. |
| Selector 2; readable selected slot | Perform no designated object-field write; return **0xFFFFFF06**. |
| Selector 0xFFFFFFFF; readable selected slot | Take the same unsupported-selector path; return **0xFFFFFF06**. |

Additional boundary conditions:

- **Index −1:** the slot address is 396 bytes before the table base. The function does not reject the negative index.
- **Index 32,767:** the slot displacement is 12,975,732 bytes. Normal completion still depends on valid storage at the resulting address.
- **Unsupported selector with an unreadable selected slot:** no normal return is guaranteed, because the slot read precedes selector validation.
- **Selector 0 followed by selector 1:** each call performs its own lookup. If both resolve to the same object and the fields are distinct, the second call does not itself replace the first field.
- **Payloads differing only in their upper five bits:** selector 0 stores the same value for both; selector 1 preserves the difference.

## Uncertainties

The following remain **documented_only**:

- The business meaning of either field.
- Whether selector 0’s mask represents a bounded integer, packed flags, or another format.
- The symbolic meaning of 0xFFFFFF06.
- Table extent and permitted index range.
- Table and object initialization, lifetime, and ownership rules.
- Endianness and concrete byte order of the 32-bit accesses.
- Alignment requirements and the runtime response to invalid accesses.
- Atomicity, thread safety, and behavior during concurrent table or object changes.
- Whether malformed addressing wraps, faults, or has another outcome in the execution environment.
- Machine-level enforcement of the represented access order.

None of these unknowns changes the established selector mapping, replacement widths, mask, or normal return bit patterns.

## Evidence

This contract is based solely on the supplied static representation of F-452fbdff8a2a.

Established evidence consists of the input widths and signedness, signed-index table addressing with a 396-byte stride, the selected-slot read before selector branching, two 32-bit replacement destinations, the 27-bit mask on selector 0, the unchanged payload on selector 1, and the two normal return bit patterns.

The description of this operation as a field setter is inferred intent. Field semantics, error-code naming, and broader subsystem behavior are not established.

No implementation access, runtime execution, or runtime tests were used.
