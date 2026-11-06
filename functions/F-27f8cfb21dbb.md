<!-- Trusted provenance: F-27f8cfb21dbb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-27f8cfb21dbb

## Purpose

F-27f8cfb21dbb writes an indexed value into a memory region using one of three byte layouts selected by a field in a caller-supplied context:

- Selector `0x10`: two bytes per entry, retaining the low 16 bits.
- Selector `0x20`: four bytes per entry, retaining the low 28 bits and clearing the upper four bits of the last byte.
- Every other selector: packed 12-bit entries, with adjacent entries sharing a byte.

The supplied static evidence establishes the memory operations and normal return value. It does not establish the application meaning of these entries, the capacity of the destination region, or a validation policy.

The apparent intent is to update one entry in an externally managed table. That intent is inferred; the byte-level behavior below is the contract supported by the evidence.

## Interface

### Inputs

The function has three 32-bit inputs:

| Input | Interpretation |
|---|---|
| Context address | Address used to locate the layout selector and destination-base field. Its recovered representation is a signed 32-bit integer. |
| Entry index | Unsigned 32-bit index used in address calculations. |
| Value | Unsigned 32-bit value whose selected low-order bits are written. |

Two context fields are accessed:

| Context-relative byte offset | Access width | Role |
|---|---:|---|
| `0x42C` | 32 bits | Layout selector, compared with `0x10` and `0x20`. |
| `0x458` | 32 bits | Destination-base address, initially read before the selector and reread during writing. |

The evidence uses 32-bit address-bearing values. It does not independently establish a portable pointer model, alignment requirements, or the byte order used to decode the context fields.

Destination accesses are one byte wide. Their ordering therefore defines the emitted byte layout independently of the byte order used for multibyte context fields.

### Output

Every path that completes normally returns the 32-bit value `1`.

There is no evidenced alternate return value for an unsupported selector, an excessive index, an invalid address, or a failed memory access. Normal return is not evidence that bounds or address validity were checked.

### Preconditions for a defined, successful call

The caller must provide memory permitting all reads and writes actually selected by execution:

- The initial destination-base field and the selector field must be readable.
- Each later destination-base reread must be valid.
- Every destination byte written must be writable.
- A shared packed byte must also be readable before its read-modify-write operation.
- Address calculations must produce usable addresses under the execution environment’s arithmetic and memory rules.

No length or allocation size is supplied or checked.

A stable, nonoverlapping context and destination region is sufficient for the ordinary layouts described below, but stability and nonoverlap are not enforced. If a write changes the destination-base field, later writes can use the changed address.

## Behavior

### Selection and initial observations

The observable memory sequence begins with a 32-bit read of the destination-base field at context offset `0x458`.

The function then reads the selector at offset `0x42C`:

1. If that read equals `0x10`, it selects the two-byte layout.
2. Otherwise, it reads the selector again and compares that second observation with `0x20`.
3. If the second read equals `0x20`, it selects the four-byte layout.
4. Otherwise, it selects the packed 12-bit layout.

The selector is therefore not necessarily observed only once. Under ordinary stable memory, it has one consistent value. If the observations differ, the branch depends on the first comparison and, when reached, the second comparison. The evidence provides no synchronization contract for concurrent changes.

The byte derived from value bits 8–15 is computed before the selected writes. In the packed path, that byte may instead be replaced with a different value-derived byte before the final store.

### Two-byte layout: selector `0x10`

The entry displacement is the unsigned 32-bit product of the index and two.

Operations occur in this order:

1. Using the initially read destination base, write value bits 0–7 at the entry displacement.
2. Reread the destination-base field.
3. Using that newly read base, write value bits 8–15 at the entry displacement plus one.
4. Return `1`.

Bits 16–31 do not affect the stored bytes.

With a stable base and ordinary in-range arithmetic, the two bytes are the little-endian representation of the value truncated to 16 bits. Neither byte is read before being overwritten.

The base reread is significant: the two stores are not guaranteed to target adjacent addresses if the first store changes the context’s destination-base field or another agent changes it between accesses.

### Four-byte layout: selector `0x20`

The entry displacement is the unsigned 32-bit product of the index and four.

Operations occur in this order:

1. Using the initially read base, write value bits 0–7 at the entry displacement.
2. Reread the destination-base field, then write value bits 8–15 at the entry displacement plus one.
3. Reread the destination-base field, then write value bits 16–23 at the entry displacement plus two.
4. Reread the destination-base field, then write value bits 24–27 at the entry displacement plus three, with the upper four bits of this byte cleared.
5. Return `1`.

No destination byte is read before replacement.

The resulting stable-base layout represents the low 28 bits of the input in four little-endian bytes. It is not an unrestricted 32-bit store. In particular, value bits 28–31 are discarded, and the upper nibble of the fourth byte is overwritten with zero rather than preserved.

The selector is not reread after this path has been selected.

### Packed layout: every other selector

The value is first reduced to its low 12 bits.

The displacement calculation has a width-sensitive intermediate:

1. Multiply the unsigned 32-bit index by three, retaining a 32-bit result.
2. Interpret that result as signed 32-bit.
3. Divide the signed result by two, truncating toward zero.

For ordinary small indices, this equals the floor of three times the index divided by two. It must not be generalized to that mathematical formula for the entire unsigned 32-bit input range: multiplication can wrap, and conversion to signed form can make the dividend negative.

The index’s low bit selects one of two layouts.

#### Even index

Operations occur in this order:

1. Using the initially read base, write value bits 0–7 at the computed displacement.
2. Reread the destination-base field.
3. At that newly read base plus the displacement plus one, read one byte.
4. Preserve the read byte’s upper nibble and replace its lower nibble with value bits 8–11.
5. Write the merged byte back to that address.
6. Return `1`.

The preserved upper nibble is taken from the byte observed after the first store and after the base reread. It is not necessarily the byte’s value at function entry.

#### Odd index

Operations occur in this order:

1. At the initially read base plus the computed displacement, read one byte.
2. Preserve that byte’s lower nibble and replace its upper nibble with value bits 0–3.
3. Write the merged byte back to the same address.
4. Reread the destination-base field.
5. At that newly read base plus the displacement plus one, write value bits 4–11.
6. Return `1`.

The second byte is replaced completely; it is not read before writing.

#### Shared-byte relationship

With a stable base and indices in the ordinary nonwrapping range, each pair of consecutive entries occupies three bytes:

- The even entry occupies the first byte and the lower nibble of the middle byte.
- The odd entry occupies the upper nibble of the middle byte and the final byte.

Each packed update preserves the neighboring entry’s nibble in the shared byte. That preservation is implemented by a separate byte read and byte write, not by an evidenced atomic masked update.

### Arithmetic boundaries

The index is not range-checked.

The products by two, three, and four are formed from unsigned 32-bit operands. Their wraparound matters before subsequent address formation or, in the packed case, signed interpretation and division.

The evidence does not justify treating all later signed address additions as portable, defined arithmetic for every input. Exact behavior when address-bearing signed additions overflow is documented_only. The ordinary layout guarantees apply when those calculations remain valid in the target environment.

## Side effects

The function directly modifies destination memory:

| Selected layout | Explicit byte stores | Destination-byte reads |
|---|---:|---:|
| Two-byte | 2 | 0 |
| Four-byte | 4 | 0 |
| Packed, even index | 2 | 1 |
| Packed, odd index | 2 | 1 |

These counts describe explicit operations on normally completed paths, not unique addresses. Aliasing or arithmetic behavior can make accesses overlap.

The context is explicitly read, not explicitly written as a context object. Nevertheless, a destination address can overlap the context, so destination stores may change its fields indirectly. This matters because the destination base is repeatedly loaded from memory.

There is no evidenced allocation, deallocation, ownership transfer, reference-count change, callback, logging, locking, or rollback operation. Memory ownership remains unspecified; the function merely accesses addresses supplied through the context.

If an access fails after an earlier store, the evidence contains no mechanism that restores prior contents. Partial updates are therefore a risk. The exact fault mechanism and caller-visible failure handling are not established.

The multiple stores do not constitute an evidenced atomic entry update. Packed updates also have a read-modify-write interval during which an unsynchronized competing update could invalidate the intended preservation of the neighboring nibble. No concurrency guarantee is established.

## Dependencies

No callee invocation appears in the supplied body of F-27f8cfb21dbb. There are therefore no additional opaque function dependencies to enumerate.

Execution depends on the environment’s handling of:

- 32-bit field reads and address representations.
- Unsigned 32-bit multiplication.
- Signed interpretation and division in the packed displacement calculation.
- Byte reads and writes.
- Address validity, alignment, memory faults, and any concurrent access.

No unknown callee is assumed to preserve state or manage memory.

## Acceptance examples

The following are conditional acceptance criteria derived from static evidence. They are not reports of executed tests.

Unless otherwise stated, assume a stable destination base, stable selector, valid accessible memory, no overlap with the context, no concurrent modification, and valid address arithmetic.

### Two-byte truncation

Given selector `0x10`, index `3`, and value `0x1234ABCD`:

- Displacement is six bytes.
- The byte at displacement six becomes `0xCD`.
- The byte at displacement seven becomes `0xAB`.
- The function returns `1`.

The input’s upper 16 bits do not contribute to the stores.

### Four-byte high-nibble clearing

Given selector `0x20`, index `2`, and value `0xFEDCBA98`:

- Displacement is eight bytes.
- Bytes at displacements eight through eleven become, in order, `0x98`, `0xBA`, `0xDC`, and `0x0E`.
- The function returns `1`.

A fourth byte of `0xFE` would violate the evidenced masking behavior.

### Packed even entry

Given a selector other than `0x10` or `0x20`, index `0`, value `0x1ABC`, and an existing byte `0xD7` at displacement one:

- The retained value is `0xABC`.
- The byte at displacement zero becomes `0xBC`.
- The byte at displacement one becomes `0xDA`.
- The existing upper nibble `0xD` is preserved.
- The function returns `1`.

### Packed odd entry

Given a fallback selector, index `1`, value `0xABC`, and an existing byte `0x5D` at displacement one:

- The byte at displacement one becomes `0xCD`.
- The byte at displacement two becomes `0xAB`.
- The existing lower nibble `0xD` is preserved.
- The function returns `1`.

### Unrecognized selector

Given a stable selector of zero and otherwise valid inputs, the function uses the packed 12-bit path. It does not reject the selector or return a selector-specific error.

### Wrapped packed displacement

Given fallback selection and index `0x80000000`:

- The 32-bit product by three is `0x80000000`.
- Its signed interpretation is −2147483648.
- Division by two gives displacement −1073741824.

This criterion establishes the displacement calculation only. It does not establish that the resulting address is accessible or that the call completes.

### Destination-base aliasing

If the first store changes bytes of the destination-base field, a later store must use the field as subsequently read, subject to the environment’s field-read semantics. Reusing the original base for every store would not reproduce the evidenced access sequence.

## Uncertainties

The following are documented_only:

- The context’s application-level type and the semantic meaning of an entry.
- Why selector values other than `0x10` and `0x20` select the packed layout.
- The intended valid index range and destination capacity.
- Whether overlapping context and destination memory is supported or accidental.
- The target’s exact pointer, alignment, field-endianness, and signed-overflow rules.
- Fault delivery, recovery, and behavior after an invalid access.
- Any synchronization obligations beyond the absence of visible synchronization here.

A 16-/28-/12-bit table interpretation is consistent with the stores, but no domain-specific table identity is established.

## Evidence

Established static evidence consists solely of the supplied function representation: three 32-bit inputs, two context-field offsets, selector comparisons, width-sensitive arithmetic, byte reads and writes, repeated base-field loads, and the constant normal return value.

The layout and access-order requirements above follow from those operations. Application intent and portability limits are identified separately rather than assumed.

No runtime tests have been performed. No implementation execution, external dependency inspection, or independent machine-level verification is claimed.
