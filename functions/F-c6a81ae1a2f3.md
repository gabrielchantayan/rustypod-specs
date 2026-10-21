<!-- Trusted provenance: F-c6a81ae1a2f3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c6a81ae1a2f3

## Purpose

F-c6a81ae1a2f3 conditionally updates a byte in an indirectly referenced state object and, on the update path, writes a second byte to the value one.

The decision compares the existing byte with the **full-width unsigned input**, not with its byte-truncated form. Consequently, an input outside the byte range always takes the update path, even when its low eight bits already match the stored byte.

Interpreting the first field as a selection or setting, and the second as a change notification, is plausible intent only. Their application-level meanings are not established by the supplied evidence.

## Interface

The function has two inputs and no return value:

| Input | Representation | Role |
|---|---|---|
| Object address | Recovered 32-bit address-bearing integer | Locates an indirect state pointer at byte offset `0xF00`. |
| Requested value | Recovered 32-bit unsigned integer | Compared in full with the existing byte; reduced to one byte for storage on the update path. |

The relevant memory layout is:

| Location | Access width | Use |
|---|---:|---|
| Object address + `0xF00` | 32 bits | Contains the state address. |
| State address + `0xB88` | 8 bits | Byte compared with the requested value and conditionally overwritten. |
| State address + `0xB90` | 8 bits | Byte written to one on the update path. |

Offsets are measured in bytes. The two state-field offsets differ by eight bytes; the evidence does not establish the contents or purpose of the intervening storage.

The supplied representation supports this 32-bit pointer/integer and 8-bit byte interpretation. It does not independently establish the complete ABI, endianness, alignment requirements, or language-level signed-character rules.

### Preconditions

For normal completion:

- The object address must support reading the state-pointer slot.
- Every state address used for the comparison or a subsequent write must identify memory valid for that particular access.
- The comparison field must be readable.
- On the update path, the destination fields must be writable.
- Address calculations must resolve to valid locations under the execution environment’s addressing rules.

There is no visible null check, bounds check, allocation check, or validation of the requested value. The function exposes no error result.

## Behavior

### Comparison and early return

The function first reads the state address from the object’s pointer slot. It then reads the byte at offset `0xB88` from that state address.

That byte is interpreted as unsigned, giving a value from zero through 255. It is compared numerically with the entire unsigned requested value.

If the values are equal, the function returns immediately. On this path:

- Neither state field is written.
- The notification byte at offset `0xB90` is not read.
- Its existing value is preserved, including values other than zero or one.
- No visible dependency is invoked.

Equality is possible only when the requested value lies between zero and 255 inclusive.

### Update path

If the comparison is unequal, the function performs two ordered byte stores:

1. It writes the requested value’s low eight bits to the field at offset `0xB88`.
2. It writes the exact byte value one to the field at offset `0xB90`.

The first store precedes the second. The second operation is an assignment, not a bitwise update: a previous notification value of any kind is replaced by one.

There is no second comparison after truncation. In particular, an out-of-range request can cause the function to store the same byte already present and still write the notification byte.

For the stored byte, the observable reduction is modulo 256. For example, requested values 256 and 511 produce stored byte values zero and 255 respectively. This reduction applies to storage, not to the initial comparison.

### Indirect-address resolution and ordering

The supplied representation resolves the state pointer separately for the comparison and for each store. It does not establish a single captured state address reused throughout the operation.

The represented memory-access order is therefore:

1. Read the state-pointer slot.
2. Read the comparison byte through that pointer.
3. If unequal, read the state-pointer slot again.
4. Write the requested byte through that pointer.
5. Read the state-pointer slot again.
6. Write one through that pointer.

Under a stable, ordinary object layout, all three pointer reads yield the same address. The contract must not silently require that result for unusual aliasing arrangements or externally changing memory.

In particular, the final destination is derived after the first store. If that store can overlap the pointer slot and alter its representation, the final pointer resolution may differ. Whether such an arrangement is valid in the actual execution environment is not established.

This ordering describes the supplied static representation. It is not a guarantee of atomicity, synchronization, or externally observable processor-level transactions.

## Side effects

On the equality path, there are no explicit memory writes.

On the update path, there are exactly two explicit byte stores in the supplied representation:

- The requested value reduced to eight bits.
- The byte value one.

There is no explicit write to the pointer slot itself. An overlapping memory layout could nevertheless cause a field store to affect that slot; non-overlap is not established by the source alone.

The function does not visibly allocate, release, transfer, or retain ownership of memory. It accesses caller-provided storage through the supplied address and the pointer found there. The caller remains responsible for ensuring that this storage stays accessible during the operation.

No logging, I/O, callbacks, or synchronization operations are visible. No notification consumer or mechanism for clearing the second byte is shown.

The operation is not transactional. If an access fails after the first store, the supplied representation contains no rollback or recovery path. It does not establish the surrounding environment’s fault-handling behavior.

## Dependencies

No called-function dependencies appear in the supplied body. The private dependency mapping identifies only F-c6a81ae1a2f3 itself; it supplies no additional behavioral contracts.

The function depends on the memory layout described above and on the execution environment’s rules for address resolution and byte access. It does not rely on any visible callee for validation, memory ownership, or preservation of global state.

## Acceptance examples

These examples are conditional specifications, not observed runtime results. Except where noted, assume valid accessible storage, a stable state pointer, and non-overlapping pointer and state fields.

| Initial byte at `0xB88` | Requested value | Initial byte at `0xB90` | Required result |
|---:|---:|---:|---|
| 7 | 7 | 0 | Return without writes; fields remain 7 and 0. |
| 7 | 7 | 42 | Return without writes; notification remains 42. |
| 7 | 8 | 0 | Store 8, then store notification value 1. |
| 0 | 255 | 9 | Store 255, then store notification value 1. |
| 0 | 256 | 0 | Comparison is unequal; store 0, then store notification value 1. |
| 255 | 511 | 0 | Comparison is unequal; store 255, then store notification value 1. |
| 255 | 255 | 1 | Return without writes; both fields remain unchanged. |

Additional boundary and ordering requirements:

- **Maximum unsigned input:** For requested value `0xFFFFFFFF`, the comparison cannot equal any unsigned byte. The update path stores `0xFF`, then one.
- **No sticky-bit preservation:** If the notification byte begins as `0x80`, taking the update path replaces it with `0x01`, not `0x81`.
- **Equality does not repair notification state:** Matching values do not cause the notification byte to be normalized or cleared.
- **Separate pointer resolutions:** If a valid execution permits the pointer slot to change between its reads, the stores use the addresses obtained at their respective resolution points. A single stable destination cannot be assumed from this representation.
- **Invalid storage:** A missing or invalid pointer does not produce a specified error return. Normal completion is outside the established contract.

## Uncertainties

The following are **documented_only**, not filled in by assumptions:

- Application-level names and meanings of either byte.
- The complete object and state-object layouts, including allocation sizes.
- Whether callers restrict requests to the range zero through 255.
- Whether the pointer slot is immutable during the function.
- Whether the layout guarantees non-overlap between the pointer slot and writable fields.
- Whether any accessed memory is volatile, memory-mapped, or shared.
- Alignment, endianness, and exact fault behavior.
- Thread-safety, atomicity, and memory-ordering guarantees.
- Language-level treatment of out-of-range conversion to plain character types, independently of the represented byte store.
- Whether another component consumes, clears, or assigns meaning to the notification byte.

Calling the second byte a “dirty flag” would be inferred intent. The established behavior is only that an unequal comparison leads to a write of one.

## Evidence

**Established static evidence:** The supplied body contains one unsigned-byte comparison against the unsigned input, an equality early return, and two ordered byte assignments on the unequal path. It contains repeated indirect-pointer resolution, no visible calls, and no explicit validation or recovery.

**Inferred intent:** The two writes resemble updating a byte-sized setting and marking associated state as changed. That interpretation is not required to explain the observable behavior.

**Verification limit:** No runtime tests have been performed. The examples specify outcomes derived from the supplied static representation under their stated preconditions; they are not execution evidence.
