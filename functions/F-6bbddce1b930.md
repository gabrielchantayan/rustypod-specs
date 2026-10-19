<!-- Trusted provenance: F-6bbddce1b930; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6bbddce1b930

## Purpose

F-6bbddce1b930 conditionally retrieves a 32-bit value through two dependent memory reads.

A zero first input produces a 32-bit zero result without either lookup read. Otherwise, the function reads an address-bearing field four bytes after the supplied record address, uses the second input to select a four-byte displacement from that loaded address, and returns the 32-bit value at the selected location.

**Established static behavior:** the function performs a zero-gated, indexed read. It has no explicit bounds check, loaded-address check, memory write, or dependency call.

**Inferred intent:** the address-bearing field may identify a sequence of four-byte entries. The supplied representation does not establish a collection type, entry count, or valid index range. The returned bits have no established meaning beyond their width.

The indexed read is not restricted to storage separate from the record. It can select the record’s first four bytes, its address-bearing field, another part of the record, or other readable storage.

## Interface

### Inputs

The function accepts two inputs:

| Input | Representation | Role |
|---|---|---|
| Record address | 32-bit integer carrying an address | Zero selects the immediate-return path. A nonzero value is used to locate the address-bearing field at byte offset 4. |
| Index | 32-bit signed integer | Selects a displacement measured in four-byte units from the loaded address. |

There is no explicit rejection of negative indices.

The supplied representation supports 32-bit address-bearing quantities. It does not establish a portable native-pointer interface, compatibility with wider addresses, a calling convention, argument locations, or external symbol binding.

### Output

The result is exactly 32 bits:

- Zero when the record-address input is zero.
- Otherwise, the value obtained by the indexed four-byte read.

No signed interpretation, pointer interpretation, or ownership meaning is established for the result. There is no separate status output. A returned zero does not distinguish the immediate-return path from a successful lookup of zero-valued storage.

### Relevant memory layout

For a nonzero record address, the directly addressed field used to obtain the lookup base is:

| Position relative to the record address | Width | Use |
|---|---:|---|
| Bytes 4–7 | 4 bytes | Address-bearing value used as the base of the indexed read |

Offset 4 is the only directly addressed record field used to obtain that base. This does **not** mean other record bytes are excluded from the function’s reads or result.

The subsequent read selects four bytes beginning at the loaded base plus four times the index, subject to the target’s arithmetic semantics. Its location is unconstrained by any explicit record boundary or separate-storage requirement. It can overlap any readable storage, including:

- Record bytes 0–3.
- The address-bearing field at bytes 4–7.
- Other portions of the record.
- Storage outside the record.

For example, if the loaded base equals the original record address and the index is zero, the returned value comes from record bytes 0–3.

No length, capacity, tag, sentinel, or ownership field is separately consulted. The indexed read could nevertheless land on storage that surrounding code assigns one of those meanings; this function would return its bits without interpreting them.

### Preconditions for a defined successful lookup

On the nonzero-record path, successful completion requires:

- A valid calculation of the address four bytes after the supplied record address.
- A readable four-byte address-bearing field at that location.
- A loaded representation usable as an address in the execution environment.
- Defined index scaling and address addition.
- A readable four-byte selected location satisfying the environment’s access and alignment requirements.

These are caller or environment obligations, not checks performed by the function. Failure behavior is not established.

The zero-record path requires neither lookup region to exist. The index does not cause a lookup access on that path.

## Behavior

### Zero-record path

The function first tests whether the record-address input is zero.

If it is zero, the function returns a 32-bit zero. Neither the read at record offset 4 nor the indexed read occurs. In particular, the function does not read address 4 merely because that would be the field position relative to a zero record address.

Every representable index produces the same result on this path. Index scaling is not required to obtain that result.

### Nonzero-record path

For a nonzero record address, the lookup consists of two dependent reads:

1. Read the four-byte address-bearing field beginning at record offset 4.
2. Use the value obtained from that field as the base for a four-byte read at the index-selected displacement.

The second read supplies the returned bits. There is no explicit comparison, semantic interpretation, or further dereference of those bits.

The first read must supply the base used by the second. The record address itself is not automatically the lookup base. It becomes the lookup base only if that is the value obtained from the field.

This order describes the source-level lookup. It does not establish a machine instruction count, atomic access guarantee, or hardware memory-ordering contract.

### Indexing and arithmetic

The stride is four bytes. It is not one byte and is not an unspecified native word size.

Where the arithmetic and resulting accesses are valid:

- Index zero selects the loaded base.
- Index one selects four bytes after the loaded base.
- Index two selects eight bytes after the loaded base.
- Index negative one selects four bytes before the loaded base.

The function establishes neither a lower bound nor an upper bound. Negative indices are not rejected, but their validity depends on arithmetic rules and the surrounding memory layout.

The address-bearing values and index use 32-bit representations. Arithmetic requiring care includes:

- Adding four to the record-address input.
- Multiplying the index by four.
- Adding that displacement to the loaded base.

The supplied material does not establish wraparound, saturation, widening, or a particular failure behavior when an intermediate calculation exceeds its applicable range. It would therefore be unsupported to define all extreme inputs using modular 32-bit arithmetic or to assume that a wider intermediate makes them safe.

Likewise, representability alone does not prove that a computed address designates valid storage. Arithmetic validity and memory-access validity are separate preconditions.

### Aliasing and overlapping storage

No disjointness check separates the record, the address-bearing field, and the selected entry.

The indexed read can overlap either part or all of the first read’s storage, or select a different portion of the record. Consequently, record bytes outside offsets 4–7 may determine the returned value when selected by the loaded base and index.

Two important cases are:

- If the field contains the record address and the index is zero, the second read returns record bytes 0–3.
- If the field contains the address of the field itself and the index is zero, both logical reads target the same four-byte location.

In the second case, stable storage makes the returned bits equal to the address value obtained by the first read. Without a stability assumption, equality is not guaranteed merely because the locations coincide.

### Results and failure boundaries

A selected entry containing zero produces a zero result after both reads. That result is indistinguishable at the interface from the zero-record result.

A zero **loaded base** does not select the immediate-return path. Only the original record-address input is checked. With a nonzero record address, a loaded base of zero, and index zero, the function attempts the selected read at address zero.

No explicit fallback, error return, exception handler, retry, or recovery path is present. The outcome of unreadable storage, invalid address representations, unsupported alignment, or invalid arithmetic remains dependent on semantics not supplied here.

## Side effects

The function contains no explicit memory writes. It does not assign to the record, the selected entry, a global variable, or an output buffer.

On the nonzero-record path, its explicit lookup activity consists of one four-byte field read followed by one four-byte selected read. Those read regions may coincide, partially overlap, or be disjoint. On the zero-record path, neither lookup read occurs.

There are no explicit allocation, release, reference-count, or ownership-transfer operations. The function does not establish ownership of the record or selected storage. Returning bits that happen to encode an address does not establish that the caller receives ownership of the addressed object.

No synchronization operation is shown. The function does not establish a coherent snapshot across both reads. At the source-level dependency, the second address is derived from the value obtained by the first read, not from a separately refreshed field value. Concurrent modification, access atomicity, visibility, and memory ordering remain unresolved.

The absence of writes does not prove that every possible read is externally side-effect-free. Device mappings or other special-memory rules could attach effects to reads. Such mappings are not established by the supplied material.

## Dependencies

There are no explicit calls to other functions. No additional opaque function IDs are required.

Execution depends on environmental semantics for:

- Interpreting 32-bit address-bearing values.
- Performing four-byte reads.
- Applying target byte order and alignment rules.
- Evaluating address-related arithmetic.
- Handling invalid accesses.
- Handling concurrent or special-memory access.

No callee is invoked whose global-state effects, memory ownership, or release behavior must be assumed.

## Acceptance examples

These examples are conditional consequences of static evidence, not runtime test results. Successful-lookup examples assume valid arithmetic, readable storage, applicable alignment, and stable contents during the lookup unless stated otherwise.

### Zero record address

Given a record-address input of zero and any representable index, the result is a 32-bit zero. Neither lookup read occurs.

This also applies when the index would make scaling or address selection problematic on the nonzero-record path.

### First selected entry

Given a valid nonzero record address whose field at byte offset 4 contains a usable base, index zero returns the four-byte value at that base.

Whether those bytes belong to the record or separate storage does not change the selection rule. Record bytes 0–3 can determine the result if the loaded base selects them.

### Four-byte stride

Suppose the loaded base identifies readable four-byte values of 17, 34, and 51 at byte displacements 0, 4, and 8:

| Index | Result |
|---:|---:|
| 0 | 17 |
| 1 | 34 |
| 2 | 51 |

The function returns the selected value directly; it does not perform another dereference based on that value.

### Negative index

Suppose the four bytes immediately before the loaded base are readable and subtracting four is valid. Index negative one returns those four bytes as the 32-bit result.

Rejecting every negative index would introduce behavior not established by this function.

### Selecting the record’s first four bytes

Suppose a nonzero record has the 32-bit value 73 in bytes 0–3, and its address-bearing field contains the original record address.

With index zero, the first read obtains that record address from bytes 4–7. The second read selects bytes 0–3, and the result is 73.

This case demonstrates why the directly addressed base field must not be confused with the complete set of record bytes the function can read.

### Selecting the address-bearing field

Suppose the address-bearing field contains its own address, and the index is zero.

The two reads target the same four-byte location. With stable storage, the returned bits equal that stored address. No write occurs.

### Stored zero versus loaded zero

If a valid selected entry contains zero, the function returns zero after both reads.

If instead the address-bearing field contains zero and the index is zero, the function attempts a four-byte read at address zero. A guaranteed zero result is not established. If that address is readable, its contents determine the result; otherwise, the outcome depends on the environment.

## Uncertainties

The following remain **documented_only**:

- Calling convention, argument placement, and external binding.
- Target byte order.
- Alignment requirements and unaligned-access support.
- Exact arithmetic semantics at representational limits.
- Address validity rules and invalid-access consequences.
- Atomicity, concurrency semantics, and hardware ordering.
- Logical record size and type.
- Entry type, entry count, and valid index range.
- Object lifetime and ownership requirements imposed by surrounding code.
- Whether any reachable storage is ordinary, shared, volatile, or device-backed memory.

The sequence interpretation is inferred intent, not an established collection contract. No bounds-safety, lifetime-safety, concurrency-safety, or wider-address portability guarantee follows.

No record/storage separation requirement is established. Unknown record layout must not be used to exclude aliasing that the supplied address calculation permits.

## Evidence

The supplied static representation establishes:

- A check of the original record-address input against zero.
- A 32-bit zero result on the zero-input path.
- A four-byte address-bearing read at record offset 4 on the nonzero path.
- Index scaling by four.
- A subsequent four-byte read using the loaded base and scaled index.
- Return of that second read’s 32-bit value.
- No explicit calls, writes, bounds checks, loaded-base zero check, allocation, release, or recovery operation.

The same address calculation establishes the aliasing possibilities: nothing requires the selected location to lie outside the record or outside the first read’s storage.

No runtime tests have been performed. The acceptance examples describe static consequences under explicit preconditions, not observed executions.
