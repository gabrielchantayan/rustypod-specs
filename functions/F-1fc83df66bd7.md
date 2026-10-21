<!-- Trusted provenance: F-1fc83df66bd7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1fc83df66bd7

## Purpose

F-1fc83df66bd7 conditionally separates fields from an unsigned input word into two one-byte outputs.

Bit 7 determines whether output is produced. When that bit is clear, the function writes bits 6–5 to the first output and bits 4–0 to the second output, then returns a 32-bit zero result. When bit 7 is set, it returns a 32-bit all-ones result without accessing either output location.

The established behavior is a bit-field operation. Interpreting the fields as a category, identifier, length, or other application-level value would be inferred intent, not an established part of this contract.

## Interface

The function has three inputs:

| Input | Representation | Role |
|---|---|---|
| Input word | Unsigned 32-bit value | Supplies the gate bit and both output fields. |
| First output address | Address of a one-byte writable object | Receives the field from bits 6–5 on the successful path. |
| Second output address | Address of a one-byte writable object | Receives the field from bits 4–0 on the successful path. |

It returns one 32-bit result:

| Result bits | Meaning established here |
|---|---|
| `0x00000000` | Bit 7 was clear, and both output stores were reached. |
| `0xFFFFFFFF` | Bit 7 was set; neither output was accessed. |

The return representation is established as a 32-bit value. Treating the all-ones result as signed negative one is a caller interpretation, not a separate return encoding.

The first output contains a value from 0 through 3. The second contains a value from 0 through 31. Each output occupies one byte; the function does not write a wider integer object at either address.

### Preconditions

For the successful path to complete normally, both output addresses must permit a one-byte write when their respective stores occur.

The function does not validate addresses, check capacity, or report invalid storage through its return value. A storage fault is therefore outside the two normal return outcomes.

The two output addresses need not be distinct according to the visible operation. If they designate the same writable byte, store order determines the final value.

When bit 7 is set, neither output address is dereferenced. No valid output storage is needed for the function’s visible operations on that path, subject to the surrounding execution environment’s rules for passing pointer values.

Pointer width, calling convention, and exceptional-memory-access behavior are not supplied.

## Behavior

### Gate and field interpretation

Bits are numbered from zero at the least significant bit.

The gate examines bit 7 alone. It does not require that the complete input fit in eight bits, and it does not reject a value merely because any bit from 8 through 31 is set.

If bit 7 is clear:

- The first output value is the unsigned numerical value represented by bits 6–5, shifted into the low two bit positions.
- The second output value is the unsigned numerical value represented by bits 4–0.
- All other bits in each output byte are zero.
- The normal return value is `0x00000000`.

If bit 7 is set:

- Neither output byte is written.
- Neither output byte is read.
- The normal return value is `0xFFFFFFFF`.

Bits 8–31 affect neither the gate nor either output value. Consequently, inputs differing only in those upper bits have the same normal result and the same output effects.

### Widths and arithmetic

The field selection is bitwise. It does not use signed comparisons or signed right-shift behavior.

The first field is restricted to two bits before its byte store. Its maximum value is 3. The second field is restricted to five bits, with a maximum value of 31. Both values fit in a byte without truncation of any selected field bit.

No arithmetic overflow, saturation, rounding, or sign extension is involved in computing the output values. Although the second field is obtained through a byte-sized conversion in the supplied representation, only the input’s low five bits survive its mask. Upper input bits cannot influence that field.

### Observable ordering

On the successful path, the established source-level order is:

1. Determine that bit 7 is clear.
2. Store the first field to the first output address.
3. Store the second field to the second output address.
4. Return the zero result.

Neither store depends on the previous contents of its destination. There is no output read-modify-write operation.

On the rejection path, the function determines that bit 7 is set and returns the all-ones result without touching output storage.

This ordering matters when the addresses alias. If both designate the same byte, the second store replaces the first. After normal completion, the shared byte contains the second field, not a combination of both fields.

The supplied representation establishes this logical store order. It does not establish atomicity, synchronization, memory barriers, or externally observable ordering for concurrent access or device-backed memory.

## Side effects

On a normal successful call, the function performs two one-byte stores. With distinct destinations, both resulting bytes remain available to the caller. With identical destinations, only the second stored value remains in that byte after completion.

On rejection, the function leaves output storage unchanged through its own operations.

There are no visible global-variable accesses, allocations, releases, callbacks, logging operations, or dependency calls. This does not imply that the addresses cannot point into global storage or another caller-selected memory region. If an output address refers to such storage, the corresponding byte is modified in the same way as any other output destination.

The function does not retain either address in visible state and performs no ownership transfer. The caller remains responsible for providing appropriate storage and managing its lifetime.

The operation is not transactional. In the established order, the first store precedes the second. If the execution environment permits the first store to succeed and the second to fault, the first modification may already have occurred. No rollback or fault recovery is shown. The exact consequences of invalid memory access remain outside the established normal behavior.

## Dependencies

No opaque function dependencies are present in the supplied body.

F-1fc83df66bd7 performs the gate, field extraction, stores, and result selection directly. There is no unknown callee whose behavior must be assumed to explain those operations.

External requirements are limited to the execution environment and caller-provided addresses. Memory validity, protection, and any special semantics of the addressed storage are not defined by this function.

## Acceptance examples

These examples describe conditional acceptance criteria derived from static evidence. They are not reported runtime observations.

Unless stated otherwise, each example assumes two distinct, ordinary writable byte locations and normal completion.

| Input word | First output after call | Second output after call | Return value |
|---|---:|---:|---|
| `0x00000000` | 0 | 0 | `0x00000000` |
| `0x0000001F` | 0 | 31 | `0x00000000` |
| `0x00000020` | 1 | 0 | `0x00000000` |
| `0x00000045` | 2 | 5 | `0x00000000` |
| `0x0000007F` | 3 | 31 | `0x00000000` |
| `0x00000080` | Unchanged | Unchanged | `0xFFFFFFFF` |
| `0x000000FF` | Unchanged | Unchanged | `0xFFFFFFFF` |
| `0x00000100` | 0 | 0 | `0x00000000` |
| `0xFFFFFF7F` | 3 | 31 | `0x00000000` |

Additional boundary conditions:

- **Rejection preserves sentinels:** If the outputs initially contain `0xA5` and `0x5A`, an input of `0x00000180` must leave those bytes unchanged and return `0xFFFFFFFF`. Bit 7, rather than the input’s overall magnitude, causes rejection.
- **Aliased outputs:** For input `0x00000045`, if both addresses designate the same writable byte, the first store places 2 there and the second replaces it with 5. The final byte is 5 and the return value is zero.
- **Old contents are irrelevant on success:** For input zero, both distinct output bytes become zero regardless of their prior contents.
- **Rejected calls do not dereference outputs:** For input `0x00000080`, output-address validity is not consulted by the visible body. Any claim about whether particular invalid pointer values can be passed legally remains environment-dependent.
- **Invalid successful-path storage has no defined status result here:** A clear gate bit does not turn an unwritable destination into a normal rejection. The function contains no corresponding validation branch.

## Uncertainties

The following are **documented_only**:

- The application-level meaning of the gate and fields.
- Whether callers interpret `0xFFFFFFFF` as a signed error value.
- Pointer width, ABI details, and target-specific fault handling.
- Whether callers provide ordinary memory, shared memory, global storage, or device-backed addresses.
- Concurrent observation, atomicity, and hardware ordering.
- Any stronger caller-side requirements concerning distinct addresses, null addresses, or storage lifetime.
- Whether the function participates in a larger validation or decoding process.

The likely intent is to split a gated low-byte representation into two fields. That interpretation adds no established validation beyond the bit-7 check.

## Evidence

This contract is based solely on static inspection of the supplied function representation.

The visible body establishes a bit-7 gate, two bounded field extractions, first-output-before-second-output store order, zero and all-ones 32-bit results, and the absence of dependency calls.

No runtime tests, caller inspection, ABI verification, or memory-environment experiments have been performed. The acceptance examples are derived expectations, not execution evidence.
