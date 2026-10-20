<!-- Trusted provenance: F-9a4db2e43bd9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9a4db2e43bd9

## Purpose

F-9a4db2e43bd9 conditionally updates a 32-bit field in an indirectly referenced object and, when that field differs from the requested value, writes the byte value `1` to another field.

The defining behavior is asymmetric:

- If the existing 32-bit value equals the requested value, the function returns without writing either field.
- If the values differ, the function writes the requested 32-bit value first, then writes `1` to the one-byte field.

The byte plausibly represents a change or pending-work indicator. That interpretation is **inferred intent**, not an established semantic role. The established behavior is only the conditional field update and subsequent byte assignment.

## Interface

The function receives two arguments and has no specified return value:

| Input | Established role |
|---|---|
| First argument | An integer-represented base address used to locate an indirect object reference. |
| Second argument | The requested value for a 32-bit field. |

The supplied representation uses `int` for both arguments and for the indirect address and comparison-field accesses. This contract describes those integer values and accesses as 32-bit, consistent with the supplied representation. Native pointer portability, a calling convention, and behavior on a different-width target are not established.

The relevant layout is:

| Location | Width | Use |
|---|---:|---|
| First-argument base plus `0xF00` | 32 bits | Holds the address of the indirectly referenced object. |
| Indirect-object base plus `0xB54` | 32 bits | Holds the value compared with, and conditionally replaced by, the second argument. |
| Indirect-object base plus `0xB90` | 8 bits | Receives `1` after a differing value is written. |

The offsets are byte offsets:

- `0xF00` is 3840 bytes.
- `0xB54` is 2900 bytes.
- `0xB90` is 2960 bytes.

For one stable indirect-object base, the byte field begins 60 bytes after the start of the 32-bit field. The two fields do not overlap in that layout.

The caller must provide memory supporting the accesses actually reached. The function contains no visible validation of the first argument, the stored indirect address, object extent, alignment, or write permissions.

## Behavior

### Comparison and early return

The function first obtains the indirect-object address from the 32-bit slot at the first-argument base plus `0xF00`. It then reads the 32-bit field at that address plus `0xB54` and compares the result with the second argument.

If the values are equal, execution returns immediately.

On this path:

- No field is written.
- The byte at offset `0xB90` is not read.
- The byte at offset `0xB90` is not cleared, normalized, or asserted.
- No later indirect-address lookup is performed by the visible body.

Consequently, equality does not establish any relationship between the current byte value and the current 32-bit value. An existing byte value of `0`, `1`, or another value remains unchanged.

### Unequal-value path

If the comparison reports unequal values, the function performs these effects in source-visible order:

1. It reads the indirect-address slot again.
2. It writes the requested 32-bit value at offset `0xB54` from the address obtained by that read.
3. It reads the indirect-address slot again.
4. It writes the one-byte value `1` at offset `0xB90` from the address obtained by that final read.
5. It returns without a specified result.

The byte assignment is unconditional once execution reaches it on the unequal-value path. It does not depend on the prior byte value, and there is no visible read of that byte.

The byte operation replaces the entire byte. It is not a bitwise accumulation operation. For example, an existing byte value of `0x80` becomes `0x01`, not `0x81`.

### Repeated indirect-address lookup

The source does not retain one indirect-object address for all operations. It consults the address slot once for the comparison and twice more on the unequal-value path.

With a stable slot, the comparison, 32-bit write, and byte write all refer to the same indirect object. If the slot changes between those accesses, the visible expressions can address different objects.

This distinction also matters if the caller supplies an overlapping layout in which the 32-bit value write modifies the indirect-address slot itself. The final byte destination is derived from the slot read after that write, not necessarily from the object used for the comparison or the 32-bit write.

These statements describe the access dependencies visible in the supplied source. They do not establish synchronization semantics or guarantee that a compiler emits one separate machine load for every source expression under all compilation conditions.

### Value handling and arithmetic

There is no visible range restriction on the second argument. Zero, negative values, and other representable 32-bit integer values follow the same equality-and-assignment rule.

The function performs no arithmetic transformation of the requested value. It does not increment, clamp, mask, or narrow that value before the 32-bit write. The separate byte write always uses the constant `1`; it does not derive a byte from the requested value.

Address formation uses fixed byte displacements. No explicit overflow check or address-wrap policy appears. Valid address formation and access are preconditions for defined completion; this contract does not infer a recovery or wraparound guarantee for invalid address calculations.

## Side effects

Under a stable, valid indirect-address slot and a nonoverlapping layout, successful execution has one of two outcomes:

| Initial comparison | Writes |
|---|---|
| Equal | None. |
| Unequal | Requested 32-bit value at offset `0xB54`, followed by byte `1` at offset `0xB90`. |

No allocation, deallocation, ownership transfer, callback, logging, or external notification is visible. The function does not visibly acquire or release an object reference. The caller remains responsible for the validity and lifetime of all accessed storage.

The two writes are not presented as a transaction. There is no visible rollback mechanism. If execution cannot complete the byte write after completing the 32-bit write, the source supplies no recovery behavior that restores the earlier value.

No locking, atomic operation, or memory barrier is visible. In particular, the source does not establish that another execution context can safely treat the byte as a synchronized publication marker for the 32-bit field.

There is no explicit error result. Invalid memory, insufficient permissions, unsupported alignment, concurrent mutation, and arithmetic outside valid address formation have no defined error-handling path in the supplied body.

## Dependencies

No other function is called in the supplied body. There are therefore no callable dependencies requiring additional opaque function IDs.

The function depends directly on caller-provided memory:

- The outer address slot must be readable for the initial comparison.
- The selected comparison field must be readable.
- If the comparison differs, subsequent address-slot reads must be valid.
- The selected 32-bit and byte destinations must be writable when reached.

These requirements are path-sensitive. With a stable layout, an equal-value invocation does not access the byte field and does not require write access merely to perform the visible operation.

Allocation policy, object construction, lifetime management, and protection against concurrent changes are outside the established contract.

## Acceptance examples

The examples below are conditional specifications derived from static evidence. They are not reports of executed tests. Unless stated otherwise, assume valid addresses, a stable indirect-address slot, no concurrent mutation, and a nonoverlapping layout.

### Equal value leaves a clear byte clear

Given a stored 32-bit value of `7`, a requested value of `7`, and a byte value of `0`:

- The function returns without writes.
- The stored value remains `7`.
- The byte remains `0`.

An implementation that writes `1` merely because the function was invoked would not satisfy this behavior.

### Different value updates both fields in order

Given a stored value of `7`, a requested value of `12`, and a byte value of `0`:

- The 32-bit field is written with `12`.
- The byte field is subsequently written with `1`.

An implementation that writes the byte before the requested value would not preserve the visible write order.

### Equal value preserves an unusual byte

Given a stored value of `12`, a requested value of `12`, and a byte value of `0xA5`, the byte remains `0xA5`. Equality does not trigger byte normalization.

### Different value replaces an unusual byte

Given a stored value of `12`, a requested value of `13`, and a byte value of `0xA5`, the final byte is `0x01`, not a combination of the old byte and a flag bit.

### Negative values are not special cases

Given a stored value of `0` and a requested value of `-1`, the differing-value path writes the 32-bit representation of `-1`, then writes byte `1`. A later invocation requesting `-1` takes the equal-value path if the stored value remains unchanged.

### Repeated invocation does not reassert the byte

After a successful update to `13`, suppose another actor clears the byte to `0` without changing the 32-bit field. A subsequent invocation requesting `13` leaves the byte at `0`.

The function is therefore not an unconditional “assert indicator” operation.

## Uncertainties

The following remain **documented_only**:

- The application-level meanings of the two fields.
- The concrete object types, full object sizes, and required alignment.
- Whether callers guarantee a stable indirect-address slot.
- Whether overlapping storage is possible in supported use.
- Whether any surrounding mechanism provides synchronization.
- Target byte order, calling convention, and exact machine instructions.
- Fault behavior and partial effects when an access cannot complete.
- Whether external components interpret the byte as a Boolean, counter, state code, or something else.

The change-indicator interpretation is inferred from the conditional assignment pattern. It is not needed to specify the observable accesses and must not replace them.

## Evidence

Established static evidence consists of the supplied function body: one equality comparison, an immediate return on equality, a conditional 32-bit assignment, and a subsequent one-byte assignment of `1`. The indirect-address expression appears separately at each of the three access stages.

No runtime tests have been performed. The acceptance examples express consequences of that static evidence under their stated preconditions; they do not claim observed execution, target-level atomicity, or behavior beyond the supplied body.
