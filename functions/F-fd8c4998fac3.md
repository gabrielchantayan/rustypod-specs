<!-- Trusted provenance: F-fd8c4998fac3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fd8c4998fac3

## Purpose

F-fd8c4998fac3 conditionally installs a caller-supplied address and the four-byte value found at that address into a caller-supplied record.

Installation is permitted only when both of these conditions hold, in this order:

1. The record’s four-byte field at byte offset `0x20` contains zero.
2. The record’s four-byte field at byte offset `0x18`, read only after the first condition succeeds, represents signed 32-bit `-1`.

When installation is permitted, the function copies four bytes from the supplied address into the field at `0x18`, then stores the supplied address itself into the field at `0x20`. It returns true. Otherwise, it returns false without performing either write.

The interpretation of these fields as an available slot, sentinel, value, and registration address is inferred intent. The established behavior is the conditional access and update sequence described below.

## Interface

### Inputs

The function takes two arguments:

- **Record address:** The base address of a memory region containing the fields at offsets `0x18` and `0x20`.
- **Value address:** An address from which four bytes can be read if installation is permitted. This address is also the value installed into the record’s field at `0x20`.

Neither argument has an explicit validation step. There is no explicit null check, bounds check, alignment check, or ownership check.

### Referenced layout

Offsets are byte offsets relative to the record address.

| Offset | Width | Read interpretation | Write behavior |
|---|---:|---|---|
| `0x18` | 4 bytes | Signed 32-bit integer, compared with `-1` | Receives the four bytes read from the value address |
| `0x20` | 4 bytes in the apparent target layout | Integer tested for equality with zero | Receives the value address as a pointer representation |

The second argument identifies a four-byte payload. The function does not interpret that payload numerically or require it to differ from any sentinel.

The source representation indicates a 32-bit target: the record address is carried in a 32-bit integer-shaped argument, the fields are accessed through four-byte types, and the field at `0x20` alternates between integer and pointer interpretations. The behavior should not be silently generalized to a target with wider pointers. Exact ABI details are not independently available.

### Output

The result is Boolean:

- **True:** Both eligibility conditions passed, and the normal execution path performed both stores.
- **False:** At least one eligibility condition failed, and no store was performed.

The result is not the copied payload, the installed pointer, or a status supplied by another function.

The function has no separate error result. A memory-access fault is not represented by a false return.

### Preconditions for normal completion

Required memory permissions depend on the path taken:

- Every normal path requires a readable four-byte field at record offset `0x20`.
- If that field is zero, the path also requires a readable four-byte field at offset `0x18`.
- If both eligibility conditions pass, the value address must permit a four-byte read, and both record fields must permit their respective writes.

The caller must supply addresses and storage compatible with the target’s addressing, alignment, and memory-access rules. No checked address arithmetic is visible for computing either field address.

## Behavior

### Eligibility and conditional reads

The first observable record access is a read of the field at `0x20`.

If that value is nonzero, the function returns false. On this path, it does not read the field at `0x18` and does not dereference the value address.

If the field at `0x20` is zero, the function reads the signed 32-bit field at `0x18`. It compares that value with `-1`, whose 32-bit representation is all bits set.

If the field at `0x18` is anything other than `-1`, the function returns false. On this path, it does not dereference the value address.

These short-circuit properties are part of the contract. An implementation that unconditionally reads both record fields or the input payload would introduce accesses absent from the supplied behavior.

### Successful installation

When the first field read was zero and the subsequent field read was `-1`, the function performs the following memory effects in order:

1. Reads four bytes from the value address.
2. Writes those four bytes into the record at offset `0x18`.
3. Writes the value address into the record at offset `0x20`.
4. Returns true.

There is no intervening dependency call.

The payload copy is bit-preserving at its established width. There is no visible arithmetic, sign extension, truncation of a wider payload, byte-order conversion, or content validation. For example, an all-bits-set payload is accepted even though that pattern is also used in the eligibility test.

The true result depends on the values captured by the eligibility reads. The function does not reread either field to decide its result after writing.

### Aliasing

No non-aliasing requirement is established by the supplied representation. The access sequence matters if the value address overlaps the record.

If the value address is exactly the address of the field at `0x18`, an eligible call reads the existing all-bits-set payload from that field, writes it back, and then installs that field’s address at `0x20`.

If the value address is exactly the address of the field at `0x20`, an eligible call reads the zero payload from that field before replacing the field with the value address. The field at `0x18` therefore receives zero.

These consequences are conditional on the target permitting the described overlapping accesses. More unusual partial overlaps remain subject to the same read-before-write sequence, but their admissibility depends on memory and execution rules not supplied here.

### Subsequent calls

Under ordinary, noninterfering execution, a successful call using a nonzero value address leaves the field at `0x20` nonzero. A subsequent call therefore returns false at the first eligibility check, regardless of the newly stored payload.

This is not an unconditional one-shot guarantee. Other code may modify the record, and the supplied function establishes no protection against such modification.

## Side effects

The only explicit writes are the two successful-path record updates:

- Four bytes at offset `0x18`.
- The target pointer representation at offset `0x20`.

There are no explicit writes on either rejection path.

The function does not explicitly allocate, free, retain, release, or otherwise manage the lifetime of the supplied storage. Installing an address does not establish an ownership transfer. The caller must not infer that the function extends the lifetime of the addressed object.

The function copies a payload snapshot and stores an address. It does not arrange synchronization between future changes through that address and the copied field at `0x18`.

No explicit global access, I/O, callback, or external call appears in the supplied behavior. Memory aliasing could nevertheless make record writes externally visible, including through other references to the same storage.

The operation is not established as atomic or thread-safe. Eligibility checks and writes are separate accesses. In particular, the payload write precedes the address write; there is no rollback logic if execution fails between them. The representation supplies no lock, atomic primitive, memory barrier, or concurrency protocol.

## Dependencies

No opaque function dependency is invoked by F-fd8c4998fac3.

Its behavior relies directly on target memory access and representation rules:

- Four-byte integer and payload access.
- Zero comparison for the field at `0x20`.
- Signed 32-bit comparison with `-1` for the field at `0x18`.
- Storage of the supplied address in the target pointer representation.

No unknown callee behavior is needed to explain the function’s normal path.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed test results. Unless stated otherwise, assume valid, suitably aligned storage, the apparent 32-bit layout, and no concurrent modification.

### Occupied address field

Initial state:

- The field at `0x20` is nonzero.
- The other field and the value address may have arbitrary contents.

Expected behavior:

- Return false.
- Do not read the field at `0x18`.
- Do not read from the value address.
- Perform no writes.

An unreadable value address does not by itself prevent this path from returning normally, because it is not dereferenced.

### Zero address field, wrong sentinel

Initial state:

- The field at `0x20` is zero.
- The field at `0x18` is `0`.

Expected behavior:

- Read the two fields in eligibility order.
- Return false.
- Do not dereference the value address.
- Leave the record unchanged.

The same result applies to every 32-bit value at `0x18` other than signed `-1`.

### Successful distinct-storage installation

Initial state:

- The field at `0x20` is zero.
- The field at `0x18` is signed `-1`.
- The value address identifies a separate readable four-byte payload.

Expected behavior:

- Read the payload only after both eligibility checks pass.
- Copy its exact four-byte contents to `0x18`.
- Store the value address at `0x20` after the payload write.
- Return true.

### All-bits-set input payload

Use the successful initial state, with all four input bytes set to `0xFF`.

Expected behavior:

- Return true.
- Leave an all-bits-set payload at `0x18`.
- Install the value address at `0x20`.

The copied payload is not required to clear the sentinel pattern.

### Input aliases the address field

Use the successful initial state, with the value address pointing to the record field at `0x20`.

Expected behavior:

- Read the existing zero payload from that location.
- Store zero at `0x18`.
- Replace the field at `0x20` with its own address.
- Return true.

### Inaccessible payload on an eligible path

Both record eligibility conditions pass, but the value address does not permit the required read.

Expected behavior:

- The function attempts the four-byte payload read.
- No false-return fallback is specified.
- Fault handling and recovery are outside the established contract.

## Uncertainties

The following remain **documented_only**:

- The record’s application-level type and the semantic names of its fields.
- Whether zero and `-1` represent availability, initialization, ownership, or another lifecycle state.
- The precise calling convention and physical Boolean return representation.
- Independent confirmation of target pointer width and pointer–integer representation rules.
- Endianness and alignment requirements.
- Whether address zero could designate readable storage in the intended environment.
- Caller-enforced aliasing restrictions, object lifetimes, and synchronization requirements.
- Platform behavior for invalid addresses, address arithmetic outside valid storage, or interrupted writes.

No allocation policy, ownership transfer, concurrency guarantee, or recovery behavior should be invented to resolve these unknowns.

## Evidence

The supplied static representation establishes the ordered eligibility reads, conditional four-byte payload read, two ordered stores, Boolean result, and absence of dependency calls.

The available-slot interpretation is inferred intent, not an additional behavioral requirement. Target-wide ABI properties and environmental fault behavior are not established independently.

No runtime tests have been performed. The acceptance examples describe consequences of the supplied static behavior, not observed executions.
