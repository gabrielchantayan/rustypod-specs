<!-- Trusted provenance: F-2adbf0d1a289; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2adbf0d1a289

## Purpose

F-2adbf0d1a289 classifies a bit pattern in a single unsigned field associated with a caller-supplied address. It returns true only when two separate two-bit groups have specific values.

The established behavior is a predicate, not a state transition: the function reads the field, evaluates the captured value, and returns a Boolean. It does not visibly update the field, allocate storage, release storage, or invoke another function.

Any interpretation of the bits as a lifecycle state, object category, permission set, or other domain-specific condition is unknown. The available evidence supports the numeric condition only.

## Interface

**Input:** one address-bearing argument identifying the base of a readable memory region.

The recovered interface represents this argument as an integer rather than a typed object pointer. Its role as an address is established by its use to locate a memory field. The original public parameter type, address-space rules, and full ABI are not established.

**Relevant memory layout:**

| Location relative to the supplied base | Width | Interpretation |
|---|---:|---|
| `+0x48`, decimal 72 | 32 bits | Unsigned value containing the tested bits |

The function requires access to the four-byte field at that offset. It does not establish that any other part of the containing region is readable, initialized, or meaningful.

**Output:** a logical Boolean result:

- **True:** both required bit-group conditions hold.
- **False:** at least one condition does not hold.

The physical return-register width and Boolean ABI representation are not established. Consumers should rely on the logical result, not an assumed register encoding.

**Preconditions for a defined, ordinary return:**

- The supplied value must identify an address from which the field at offset `0x48` can be read.
- The field must be accessible as the evidenced unsigned 32-bit read under the applicable platform’s alignment and memory-access rules.
- Address calculation must identify the intended field under the applicable address model.

There is no visible null check, bounds check, type check, or alignment check. A null or otherwise invalid base is not specified to produce false.

## Behavior

### Decision rule

Number the bits of the unsigned field from zero at the least significant bit. The result is true exactly when all four of these conditions hold:

| Bit | Required value |
|---|---:|
| 9 | 1 |
| 10 | 0 |
| 11 | 1 |
| 12 | 0 |

All other bits are irrelevant to this predicate.

Equivalently, the combined mask is `0x00001E00`, and the required masked value is `0x00000A00`. The function therefore accepts every 32-bit value whose intersection with that mask equals the required value.

This equivalence does not mean that the entire field must equal `0x00000A00`. Bits outside the mask may be set or clear without changing the result.

### Observable read and evaluation order

The visible sequence is:

1. Read the unsigned 32-bit field at base offset `0x48`.
2. Test the captured value against mask `0x00000600`, requiring the masked result to equal `0x00000200`.
3. If that first condition holds, evaluate the captured value against mask `0x00001800`, requiring the masked result to equal `0x00000800`.
4. Return the conjunction of the two conditions.

Only one memory-field read is present in the supplied body. The second condition uses the captured value; it does not fetch the field again.

If the first condition fails, the function returns false without needing the second bit-group condition to succeed. If the first condition succeeds, the second condition determines the result.

The conditional evaluation introduces no additional visible memory access or external call. It is not evidence of an externally observable timing guarantee.

### Width and arithmetic

The field and masks are treated as unsigned 32-bit quantities. No signed comparison is involved in deciding the result. In particular, setting bit 31 does not make an otherwise accepted pattern fail.

The classification uses masking and equality comparison only. It performs no arithmetic on the field value, so field-value overflow, carry, and saturation do not arise.

Address calculation is separate from the classification. The fixed displacement is 72 bytes; it is not multiplied by an element size. The supplied evidence does not establish how an invalid or overflowing address calculation behaves on the target platform.

### Stability and concurrency

For an ordinary readable memory location, repeated calls that capture the same 32-bit value produce the same Boolean result.

If another actor changes the field after the function has read it, the visible body contains no reread that would incorporate that later change. This is not a synchronization guarantee: atomicity, memory ordering, race semantics, and behavior for device-backed memory are not established.

## Side effects

The only explicit memory operation is the read at base offset `0x48`.

There are no visible:

- Writes to the input region.
- Writes to global state.
- Allocations or deallocations.
- Ownership transfers.
- Reference-count updates.
- External calls.
- Error-status updates or diagnostic emissions.

The caller retains whatever ownership and lifetime responsibilities it had before the call. The function does not visibly acquire or release ownership, and it does not visibly retain the supplied address.

The Boolean result reports a bit-pattern match, not memory validity. A failed read is not converted into false by any visible handling.

A read from memory with platform-specific read effects could have consequences outside this body. Such effects are not established by the supplied evidence and are not part of an ordinary-memory guarantee.

## Dependencies

No function dependencies are visible. F-2adbf0d1a289 invokes no opaque function IDs.

Its non-call dependencies are:

- A readable unsigned 32-bit field at the specified offset.
- The execution environment’s address representation and memory-access rules.
- The execution environment’s representation of the loaded bytes as an unsigned integer.

No assumption about an unknown callee preserving globals, retaining memory, or releasing memory is necessary because no call appears in the supplied body.

## Acceptance examples

These examples are conditional specifications, not results of executed tests. Each assumes a valid, appropriately accessible field at the required offset and uses the numeric value obtained by the read.

| Captured field value | Expected result | Reason |
|---|---|---|
| `0x00000A00` | True | Both two-bit groups have their required values. |
| `0x00000A01` | True | Bit 0 lies outside the tested mask. |
| `0x80000A00` | True | Bit 31 lies outside the tested mask; the field is unsigned. |
| `0xFFFFEBFF` | True | Every unconstrained bit is set, while the four constrained bits match. |
| `0x00000000` | False | Required set bits are absent. |
| `0x00000800` | False | Bit 9 is clear, so the first condition fails. |
| `0x00000E00` | False | Bit 10 is set, violating the first condition. |
| `0x00000200` | False | The first condition passes, but bit 11 is clear. |
| `0x00001A00` | False | The first condition passes, but bit 12 is set. |
| `0xFFFFFFFF` | False | Both constrained groups contain prohibited set bits. |

Additional behavioral acceptance conditions:

- Changing only bits outside `0x00001E00` must not change the result for the captured value.
- Starting from an accepted value, clearing bit 9 or bit 11 must make the result false.
- Starting from an accepted value, setting bit 10 or bit 12 must make the result false.
- For ordinary memory, the function must leave the field unchanged.
- An inaccessible field has no specified Boolean outcome; it must not be treated as an ordinary negative classification.

## Uncertainties

The following remain **documented_only**:

- **Semantic meaning:** no domain meaning is established for the field or its bits.
- **Containing layout:** only the accessed field’s offset and width are established. The complete object size and other fields are unknown.
- **Original interface:** the original pointer type, calling convention, address width, and Boolean return representation are not established.
- **Byte order:** examples describe loaded numeric values, not a required byte sequence in memory.
- **Alignment and access faults:** platform requirements and failure behavior for invalid reads are unknown.
- **Concurrent access:** atomicity, synchronization, and permitted interaction with concurrent writers are unknown.
- **Special memory:** volatility and device-read semantics are not established.
- **Caller obligations:** no caller evidence is supplied beyond the access requirements implied by this body.

Calling the accessed value a “flags field” is a reasonable inferred description of its use, not an established domain contract.

## Evidence

The supplied static body establishes one unsigned field read at offset `0x48`, a first masked comparison using `0x600` and `0x200`, a conditional second masked comparison using `0x1800` and `0x800`, and a Boolean conjunction as the return condition.

The combined-mask formulation and per-bit requirements are derived directly from those comparisons. The absence of explicit writes, calls, allocation, release, and error handling is established within the supplied body only; it does not establish behavior for unseen callers or platform fault machinery.

No runtime tests have been performed. All acceptance examples are mathematical consequences of the visible predicate under the stated preconditions.
