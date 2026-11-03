<!-- Trusted provenance: F-a4b9590c0eba; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a4b9590c0eba

## Purpose

F-a4b9590c0eba performs a signed-index lookup through a table associated with an object supplied by dependency F-ecd921e533b1. It returns the selected entry as a 32-bit integer, or returns zero when the index is negative or is not strictly below the object’s signed bound.

The established behavior is a bounded lookup, not validation of the underlying object or table. For an index admitted by the bound, the function uses the stored table address and returns the selected 32-bit entry without checking either value for nullness or validity.

Interpreting the object as a collection, the bound as its element count, and the returned entry as an object pointer is plausible intent. Those meanings are not necessary to the contract: the visible behavior requires only the field layout, signed comparison, address calculations, and reads described below.

## Interface

**Input**

One signed 32-bit integer, used as an index.

- Negative values are rejected.
- Zero is eligible for lookup only when the stored bound is positive.
- Positive values are eligible only when strictly less than the stored bound.
- The index is not normalized, clamped, or converted into a different numbering scheme.

**Output**

One 32-bit integer.

- For a rejected index, the result is zero.
- For an admitted index, the result is the exact 32-bit value read from the selected table slot.
- A selected slot containing zero also produces zero. The return value therefore does not distinguish rejection from a successful read of a zero-valued entry.
- A selected slot whose highest bit is set is returned with those bits intact. Under a signed interpretation of the return type, that result is negative; it is not treated as a lookup error.

**Dependency result**

F-ecd921e533b1 is called once, without explicit arguments, before the input index is rejected or used for lookup. The supplied representation exposes its result as 64 bits:

- The low 32 bits provide the base address used by this function.
- The high 32 bits do not affect the established lookup result or the function’s subsequent memory-read decisions.

This does not establish the dependency’s original source-level return type or calling convention. It establishes only which portion of the supplied result is operationally relevant here.

**Required memory layout**

For a nonnegative input, the base address identifies an object with the following fields:

| Offset from base | Width | Interpretation in this function |
|---|---:|---|
| `0x78` | 32 bits | Address of a table of 32-bit entries |
| `0x7c` | 32 bits | Signed bound used to admit or reject the index |

The table has a four-byte slot stride. Slot zero begins at the stored table address.

These are access requirements, not a complete object definition. No meaning is established for other object fields or for the contents of a returned entry.

## Behavior

### Dependency acquisition and rejection

The dependency call precedes every possible return, including rejection of a negative index. Consequently, rejection does not bypass any effects or failures of F-ecd921e533b1.

After the dependency returns:

- A negative index produces zero without dereferencing the object’s bound field, table-address field, or any table entry.
- A nonnegative index causes a 32-bit read at base offset `0x7c`. That value is interpreted as signed.
- The input is admitted only if it is strictly less than that signed bound.
- Equality with the bound is rejected. An input greater than the bound is also rejected.

For a nonnegative input, a bound of zero or any negative bound rejects every index. The function does not reinterpret a negative bound as a large unsigned capacity.

No separate success status, error code, exception object, or diagnostic is explicitly produced.

### Admitted lookup

For an admitted index, the read order is:

1. Read the signed bound at base offset `0x7c`.
2. Read the 32-bit table address at base offset `0x78`.
3. Read the 32-bit entry at the table address plus four times the input index.

The selected entry becomes the return value. There is no further dereference of that entry.

The table-address field is not read on ordinary rejection. In particular, an index equal to the bound does not require the table address or table storage to be valid.

The function does not reread the bound after acquiring the table address. No lock, version check, atomic snapshot, or other synchronization is visible. If the underlying memory can change concurrently or through an external actor, this function alone does not establish that the bound, table address, and selected entry belong to one consistent state.

### Widths and address arithmetic

The represented address operations use 32-bit quantities:

- The object base is taken from the low 32 bits of the dependency result.
- The field addresses are formed using offsets `0x78` and `0x7c`.
- The selected slot address uses a four-byte stride.
- The returned entry is 32 bits wide.

The lookup performs a signed comparison before calculating and reading the selected slot. That comparison is not a check that the address calculation fits without wraparound or that the resulting address remains inside an allocation.

Under the represented 32-bit machine-address interpretation, additions and the four-times-index offset retain their low 32 bits. For example, an admitted index of `0x40000000` has a four-byte offset whose low 32 bits are zero. With a sufficiently large positive bound, that index can therefore address the same location as slot zero.

This is an arithmetic consequence of the represented width, not evidence of an intended circular table. No explicit overflow rejection is present. Exact fault behavior and any distinctions imposed by the original language or execution environment remain outside the available evidence.

### Memory and validity preconditions

A negative input does not require readable object fields after the dependency returns. The dependency itself must still complete.

A nonnegative input requires the bound field to be readable, even when the input will be rejected.

An admitted input additionally requires:

- A readable table-address field.
- A readable four-byte location at the computed slot address.
- Whatever address validity and alignment the execution environment requires for these accesses.

The function does not establish these conditions before attempting the reads. In particular, a positive bound does not prove that the table has corresponding storage.

There is no explicit null-base check, null-table check, allocation-size check, or address-overflow check. If a required read is invalid, the supplied body does not specify a normal recovery path or guaranteed zero result.

## Side effects

The visible body performs no memory writes, allocations, releases, or explicit updates to the object or table.

This absence of direct writes does **not** establish that the entire invocation is side-effect-free:

- F-ecd921e533b1 runs on every invocation.
- Its effects on memory, global state, resources, or ownership are unknown.
- Its result may depend on state not supplied as an explicit argument.
- The read locations may have execution-environment-specific behavior not described by the supplied material.

No ownership transfer is established for the base address, table address, or returned entry. The function does not explicitly acquire or release ownership, but that does not determine what the dependency does or what obligations apply to the caller.

If the returned bits represent an address, this function does not prove that the referenced object exists, remains alive, or may be modified. It also does not establish whether the caller must eventually release it.

## Dependencies

### F-ecd921e533b1

**Established interaction**

- Called exactly once by this function.
- Receives no explicit arguments.
- Runs before any index-dependent rejection.
- Supplies the result whose low 32 bits determine the object base.
- Has no explicit failure-result handling in this function.

**Unknown contract**

The available evidence does not establish whether F-ecd921e533b1:

- Always returns normally.
- Always supplies a valid base address.
- Returns a stable or shared object.
- Mutates global state or the object it returns.
- Allocates, retains, transfers, or releases resources.
- Provides any synchronization or lifetime guarantee.

These properties are documented_only unknowns. They must not be inferred from this caller’s use of the returned bits.

## Acceptance examples

The following are conditional acceptance cases derived from the static evidence. They are not runtime test results. Unless stated otherwise, the dependency returns normally and every required read is valid.

1. **Negative input**

   Input is `-1`. The dependency is called once. No bound, table-address, or slot read follows. The result is zero.

   The same result and read suppression apply to the minimum signed 32-bit input, `-2147483648`.

2. **Zero input with a zero bound**

   Input is zero and the bound is zero. The bound is read, equality causes rejection, and the result is zero. Neither the table-address field nor a slot is read.

3. **Zero input with a positive bound**

   Input is zero and the bound is three. The function reads the bound, then the table address, then the first four-byte slot. If that slot contains `0x12345678`, those exact bits are returned.

4. **Last admitted ordinary index**

   Input is two and the bound is three. The selected location is eight bytes after the stored table address. Its 32-bit contents are returned.

5. **Input equal to the bound**

   Input is three and the bound is three. The result is zero after the bound read. An invalid table address does not affect this path because the table-address field is not read.

6. **Negative bound**

   Input is zero and the bound is `-1`. Signed comparison rejects the input. The function returns zero without reading the table address.

7. **Zero-valued selected entry**

   Input is admitted and the selected slot contains zero. The function performs the table lookup and returns zero. The caller cannot distinguish this case from rejection using the return value alone.

8. **Entry with its highest bit set**

   Input is admitted and the selected slot contains `0x80000000`. Those bits are returned, corresponding to `-2147483648` under signed 32-bit interpretation. No additional error branch is taken.

9. **Nonzero upper half of the dependency result**

   Two otherwise equivalent invocations receive dependency results with identical low 32 bits but different high 32 bits. With identical relevant memory contents, they perform the same subsequent reads and return the same result.

10. **Large admitted index**

    Input is `0x40000000` and the signed bound is `0x40000001`. The index passes the signed bound check. Under the represented 32-bit address arithmetic, its four-byte offset wraps to zero, so the selected read occurs at the stored table address.

11. **Invalid selected storage**

    Input is admitted, but the computed slot address is not readable. No guaranteed zero result follows from this contract. The resulting failure behavior depends on the execution environment.

## Uncertainties

**Inferred intent**

The layout suggests an indexed collection with an entry-table address and element count. The slot contents may be pointers. These interpretations are useful descriptions, not established ownership or type guarantees.

**Documented_only unknowns**

- The complete object layout and allocation size.
- The semantic meaning of the bound beyond its use in signed comparison.
- The semantic type and lifetime of table entries.
- The original dependency return convention.
- Byte order, alignment requirements, and memory-fault handling.
- Concurrency guarantees and whether relevant storage can change during a call.
- Any caller-side restrictions that prevent large indices or wrapped addresses.
- Dependency side effects, failure modes, and resource-management obligations.

No unknown is resolved by assuming that the dependency preserves globals, supplies valid storage, or manages memory on behalf of this function.

## Evidence

The contract is based solely on the supplied static representation of F-a4b9590c0eba and its dependency mapping.

Established observations include the unconditional dependency call, use of its low 32 result bits as the base, conditional signed-bound read at offset `0x7c`, table-address read at offset `0x78`, strict signed upper-bound rule, four-byte slot stride, zero rejection result, and absence of explicit memory writes or resource operations in the visible body.

The collection interpretation is inferred intent. Dependency internals, ownership rules, runtime address validity, and environmental failure behavior are unknown.

No runtime tests have been performed.
