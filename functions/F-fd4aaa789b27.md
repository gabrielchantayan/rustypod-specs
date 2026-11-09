<!-- Trusted provenance: F-fd4aaa789b27; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fd4aaa789b27

## Purpose

F-fd4aaa789b27 mutates memory reachable through incoming machine state, performs additional writes to arithmetic-derived addresses, and ends at an unresolved indirect control transfer.

The established contract is an ordered memory transformation. No application-level purpose—such as initialization, linking, serialization, or dispatch preparation—is established. Address-derived bytes and constants are written into apparent fields, but those patterns do not establish object types or valid object boundaries.

Observable behavior includes intermediate writes, repeated accesses, and fresh reads of previously accessed locations. A final-state description alone is insufficient: overlapping storage can make an early write determine a later address, payload, or transfer argument.

## Interface

### Inputs

The supplied representation exposes no ordinary parameter list. The operation nevertheless depends on four incoming values:

| Role | Representation and use |
|---|---|
| Primary base | A 32-bit integer-like address used for field access, address arithmetic, and byte-sized address tags. |
| Secondary base | A 32-bit integer-like address used to locate records and receive an incoming-word store. |
| Incoming word | A 32-bit integer stored in full, narrowed for a byte store, and later used in address arithmetic. |
| Incoming byte | A one-byte value written to two primary-base fields. |

These are descriptive roles, not a recovered calling convention.

Three global cells supply additional inputs:

- **First global cell:** a signed integer-like 32-bit value used as a halfword payload and an address component.
- **Dispatch-word cell:** a signed integer-like 32-bit value used in two final destination calculations and as a transfer argument.
- **Dispatch-payload cell:** a four-byte payload copied to a computed destination.

Each global is read at its specified point in execution. Its value is not assumed to equal an entry-time snapshot.

### Widths and arithmetic

The visible representation distinguishes 8-bit, 16-bit, and 32-bit accesses. Address-bearing field loads and stores occupy four bytes. This establishes the widths used by this contract, not an independently verified native ABI.

Offsets are byte offsets. “Low byte” and “low halfword” identify the low eight and sixteen bits of a captured value. Address-derived byte stores do not preserve a complete address.

The character-derived index has an unsigned 32-bit representation. Its additions with 0x44 and 0x45 therefore use unsigned 32-bit arithmetic in the supplied model. This does **not** establish wrapping behavior for the other address calculations, which use signed integer-like operands.

Native byte order, alignment requirements, signed narrowing details, and the validity of interpreting integer values as addresses remain unresolved.

### Output and completion

No returned value or status result is established.

The terminal operation is represented as an indirect invocation with three arguments and a return path afterward. The supplied material does not establish whether the actual control transfer is an ordinary returning call, a nonreturning transfer, or another calling-convention construct. Completion after that transfer is conditional on the target’s behavior.

## Behavior

### Memory roles and capture rules

The following names describe values obtained during execution. They are not independent objects unless the execution environment makes them so.

| Role | How its value is obtained |
|---|---|
| Selected destination | Read the four-byte address at primary base + 0x44, then read a four-byte address at that address + 0x14. |
| Saved primary payload | Read four bytes at primary base + 0x54 before that field receives the primary base. |
| Linked target | Read the address at secondary base + 0x04, then the address at that address + 0x74. |
| Linked payload | Read four bytes at linked target + 0x64 after the initial linked-target byte writes. |
| Reloaded record | A later, separate four-byte address read at secondary base + 0x04. |
| Record payload | Read four bytes at reloaded record + 0x50 after its first address-tag write. |
| Secondary displacement | Read a signed integer-like word at secondary base + 0x74. |
| Container | Read an address at secondary base + 0x14. |
| Child | Read an address at container + 0x04, after the reloaded record receives its self-address field. |
| Primary displacement | Read a signed integer-like word at primary base + 0x74. |
| Character index | Read one plain character at primary base + secondary displacement, then convert it to unsigned 32-bit form. |
| Dispatch snapshot | A late four-byte read at primary base + 0x44. |

Captured values remain the values of their respective reads. Overwriting a field from which an address was obtained does not retroactively change that captured address.

The two reads at secondary base + 0x04 are distinct. The late read at primary base + 0x44 is also distinct from its initial use in selecting a destination.

### Field-level effects

The following table describes nominal destinations. It is not a guarantee of final contents when destinations overlap.

| Destination | Width | Required value or ordered replacements |
|---|---:|---|
| Primary base + 0x1D | 1 byte | Incoming byte. |
| Secondary base + 0x34 | 4 bytes | Incoming word. |
| Primary base + 0x14 | 4 bytes | Selected destination, later replaced by linked payload. |
| Selected destination + 0x09 | 1 byte | Low byte of primary base. |
| Linked target + 0x15 | 1 byte | Low byte of incoming word. |
| Primary base + 0x54 | 4 bytes | Primary base, later replaced by decimal 100. |
| Primary base + 0x11 | 1 byte | Low byte of secondary base. |
| Linked target + 0x11 | 1 byte | Low byte of saved primary payload. |
| Primary base + 0x09 | 1 byte | Incoming byte. |
| Reloaded record + 0x09 | 1 byte | Low byte of primary base, written twice. |
| Reloaded record + 0x11 | 1 byte | Low byte of linked payload. |
| Primary base + 0x0D | 1 byte | Low byte of reloaded record, later replaced by low byte of linked payload. |
| Reloaded record + 0x00 | 1 byte | Low byte of record payload. |
| Reloaded record + 0x54 | 4 bytes | Reloaded record’s own address. |
| Child + 0x10 | 1 byte | Decimal 100. |
| Child + 0x40 | 4 bytes | Decimal 100. |
| Container + 0x09 | 1 byte | Low byte of child address. |
| Primary base + 0x24 | 4 bytes | Linked payload. |
| Container + 0x11 | 1 byte | Low byte of primary base. |

Decimal 100 is 0x64. Its one-byte and four-byte stores are separate operations with different widths.

### Ordering within the record transformation

The layout above is constrained by the following access chronology. These boundaries matter even if a particular execution uses disjoint storage.

**Initial selection boundary.** The incoming-byte write at primary base + 0x1D is the first visible memory operation. Destination selection completes before the incoming word is written at secondary base + 0x34. The selected address is then installed at primary base + 0x14 before the saved primary payload is captured.

**Initial linked-target boundary.** The selected destination receives its primary-base byte tag before the linked-target address chain is read. The linked target receives the incoming-word byte before primary base + 0x54 receives the primary base. Next come the secondary-base byte at primary base + 0x11 and the saved-payload byte at linked target + 0x11. Only then is the linked payload captured. The incoming-byte write at primary base + 0x09 follows that capture.

**Reload boundary.** The fresh secondary-base + 0x04 read occurs after the preceding primary-byte write. The reloaded record receives its first primary-base byte tag before the record payload is captured. Its linked-payload byte is written next, followed by the repeated primary-base tag. The replacement at primary base + 0x14 follows that repeated tag. The reloaded-record byte at primary base + 0x0D precedes the record-payload byte at the reloaded record’s starting address.

**Container boundary.** The secondary displacement is captured before the container address. Both reads precede the reloaded record’s four-byte self-address store. The child address is read only after that store; overlap with container + 0x04 can therefore change the child selected.

After child selection, the required order is: the child’s one-byte 100 field, the primary four-byte 100 replacement, and the child’s four-byte 100 field. The linked-payload byte then replaces primary base + 0x0D. The container’s child-address byte precedes the four-byte linked-payload store at primary base + 0x24, which precedes the container’s primary-base byte.

No validation or dependency invocation is visible between these boundaries.

### Arithmetic-derived memory effects

After the record transformation, the first global cell is captured before the primary displacement. The remaining halfword and character operations have these requirements:

- The address primary base + primary displacement receives the low halfword of the primary displacement.
- Twice the child address receives the low halfword of the captured first-global value.
- The character index is captured after both preceding halfword writes.
- First-global value + primary displacement then receives the low halfword of the primary displacement.
- The halfword write at primary base + primary displacement is repeated afterward.

Thus the character read occurs between the two nominally identical halfword writes. The first write can affect the character read if their storage overlaps. The intervening first-global-derived write can also affect storage used later.

The read is a **plain-character** read, not an explicitly unsigned-byte read. Under signed eight-bit character semantics, a byte with its high bit set is interpreted as negative before conversion to unsigned 32-bit form. Under unsigned-character semantics, the index remains between 0 and 255. The applicable character interpretation is unresolved.

The first-global capture is not refreshed if any subsequent store changes that global cell.

### Dispatch preparation and transfer

The character index determines two adjacent nominal destinations:

| Destination | Width | Payload |
|---|---:|---|
| Character index + 0x45 | 4 bytes | 0x00000045 |
| Character index + 0x44 | 1 byte | 0x44 |

The four-byte write occurs first. Between these two writes, three captures occur in this exact order:

1. Dispatch-word cell.
2. Dispatch-payload cell.
3. Dispatch snapshot at primary base + 0x44.

Consequently, the first character-indexed write can affect any of these reads through aliasing. The later one-byte write cannot affect their already captured values.

After the one-byte write, the captured dispatch payload is stored as four bytes at twice the captured dispatch word. The final explicit store writes the low halfword of the dispatch word at incoming word + dispatch word.

The terminal transfer to **F-unknown-1** receives these ordered arguments:

1. Character index bitwise-OR 0x44.
2. Captured dispatch word.
3. Dispatch snapshot.

The first argument uses bitwise OR, not addition. The third argument is the late capture, not the address used during initial destination selection.

## Side effects

### Aliasing and observable access order

All described reads use the memory contents present at their respective execution points. No disjointness assumption is established for records, global cells, arithmetic-derived locations, or incoming bases.

Repeated stores remain part of the contract even when their values match. Coalescing them, caching a fresh read, or moving a read across a store can change behavior under aliasing or externally observable memory access.

A multi-byte operation has its stated width and value. The supplied material does not establish its physical bus transaction count, atomicity, or bytewise observation order.

### Ownership and lifetime

No explicit allocation, deallocation, retention, or release is visible before the terminal transfer. Address stores do not establish ownership transfer or extend the lifetime of any referenced storage.

F-unknown-1 has unresolved ownership behavior. It may not be assumed to preserve memory or globals, nor may it be assumed to release anything.

### Preconditions and failures

There are no visible null, bounds, alignment, overflow, or access-result checks.

Reaching the terminal transfer requires every preceding operation to be supported by the actual environment. This includes stores at doubled addresses, the character-indexed destinations, and incoming word + dispatch word.

No recoverable error result or rollback is specified. If an access or arithmetic operation prevents continued execution, earlier effects may already have occurred. The supplied representation does not establish a particular fault, exception, signal, or language-level outcome.

No synchronization protocol is visible.

## Dependencies

### F-unknown-1

This is a contract-local opaque identifier for the unresolved terminal indirect target, not an established target identity.

Its invocation representation has three arguments, described above, after all explicit memory operations. The target’s address acquisition and validation are not established.

The following properties are **documented_only**:

- Actual calling convention and formal parameter types.
- Whether the transfer returns.
- Return representation, if any.
- Additional reads, writes, and global-state changes.
- Ownership and lifetime effects.
- Failure behavior.

No other function dependency is visible in the supplied representation. That does not establish the absence of environment-level effects associated with memory accesses.

## Acceptance examples

These are conditional static consequences, not executed tests.

### Stable, disjoint record fields

Assume all accesses are valid and the relevant fields and computed destinations do not overlap. Immediately before the terminal transfer:

- Primary base + 0x14 and primary base + 0x24 contain the captured linked payload.
- Primary base + 0x54 contains four-byte decimal 100.
- Primary base + 0x0D contains the linked payload’s low byte.
- Primary base + 0x09 and +0x1D contain the incoming byte.
- Reloaded record + 0x54 contains the reloaded record’s address.
- Child + 0x10 contains one-byte 100; child + 0x40 contains four-byte 100.

These expectations do not extend across unknown effects of F-unknown-1.

### Full-width and narrowed copies

For incoming-word bit pattern 0x123456AB, the secondary-base + 0x34 store carries all 32 bits. The linked-target + 0x15 store carries the low byte, 0xAB, under the represented byte-narrowing behavior.

The later address involving the incoming word uses its full integer value, not the stored byte.

### Read after an overlapping write

If the reloaded record’s self-address store changes the storage at container + 0x04, child selection uses those changed contents. Reading the child before the self-address store would violate the ordering requirement.

Likewise, an overlapping earlier store can make the second secondary-base + 0x04 read differ from the first.

### Character interpretation

For loaded byte 0x80:

- Signed eight-bit interpretation followed by unsigned 32-bit conversion produces 0xFFFFFF80.
- Unsigned interpretation produces 0x00000080.

The four-byte 0x45 destination is respectively 0xFFFFFFC5 or 0x000000C5 in the represented unsigned arithmetic. An acceptance environment must resolve the character interpretation rather than silently choose one.

### Unsigned wrap versus signed arithmetic

If signed-character interpretation of byte 0xFF produces character index 0xFFFFFFFF, the unsigned destination additions yield:

- Four-byte destination: 0x00000044.
- One-byte destination: 0x00000043.
- First transfer argument: 0xFFFFFFFF.

This unsigned result does not authorize wrapping twice the child address or incoming word + dispatch word; those expressions have different operand types.

### OR is not addition

For character index 0x40, the first transfer argument is 0x44. It is not 0x84.

### Late snapshot

If the four-byte character-indexed write modifies primary base + 0x44, the dispatch snapshot reflects that modification, subject to native representation and overlap. Substituting the initial primary-base + 0x44 value is not acceptable.

## Uncertainties

The following remain **documented_only**:

- Application-level intent and object schemas.
- Actual function boundary, ABI, and incoming-state convention.
- Native byte order and alignment restrictions.
- Plain-character signedness and signed narrowing behavior.
- Signed overflow and integer-to-address semantics.
- Validity, lifetime, overlap, volatility, and concurrent modification of accessed storage.
- Physical atomicity and transaction ordering of multi-byte accesses.
- Global-cell locations beyond their described roles.
- Actual terminal transfer form and F-unknown-1 behavior.

In particular, the signed integer-like additions and doublings cannot be assigned universal modulo-2³² semantics merely because the representation uses four-byte values. Cases outside their supported arithmetic or addressing range remain unresolved.

## Evidence

**Established static evidence:** The supplied representation supports the input roles, access widths, offsets, constants, capture boundaries, repeated operations, plain-character conversion, unsigned character-index arithmetic, and terminal three-argument invocation representation described here.

**Inferred intent:** No application-level interpretation is required. “Record,” “field,” “container,” and “child” are descriptive roles for access relationships, not recovered types or ownership claims.

**Unknowns:** Target execution semantics and dependency behavior remain explicitly bounded above; they are not replaced with assumed safe defaults.

No runtime tests have been performed. All acceptance examples are conditional deductions from static evidence.
