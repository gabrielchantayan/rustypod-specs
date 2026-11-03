<!-- Trusted provenance: F-ce3abcc1007c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ce3abcc1007c

## Purpose

F-ce3abcc1007c updates three adjacent 32-bit fields in a destination object using a one-byte discriminator and a conditional 32-bit value from a source object.

A zero discriminator produces three zero destination fields. A nonzero discriminator permits the source value to be copied, but that value has a second gating role: if it is zero, the remaining destination fields are also cleared. Only a nonzero selected value triggers access to global state.

For that active case, the function obtains a context address through a global root, stores that address in the destination, and conditionally copies a further 32-bit value through the context.

This is a description of the supplied static representation. It does not establish the semantic meaning of these objects or fields, and no runtime tests have been performed.

## Interface

### Inputs

The function takes two arguments represented as 32-bit integers used as addresses:

- **Destination base address:** identifies the object whose fields are updated.
- **Source base address:** identifies the object from which the discriminator and, conditionally, a value are read.

The observed source layout is:

| Byte offset | Width | Interpretation established by use |
|---|---:|---|
| +4 | 1 byte | Discriminator; zero and nonzero are distinguished |
| +8 | 4 bytes | Selected value, read only when the discriminator is nonzero |

The observed destination layout is:

| Byte offset | Width | Result |
|---|---:|---|
| +4 | 4 bytes | Selected source value, or zero |
| +8 | 4 bytes | Context address, or zero |
| +12 | 4 bytes | Indirectly obtained value, or zero |

The function does not explicitly access the four-byte source region beginning at offset +0, the source bytes at offsets +5 through +7, or the destination region beginning at offset +0. No total object size or additional field layout is established.

### Global input

The active branch uses one global root slot. The observable chain is:

| Location | Width | Use |
|---|---:|---|
| Global root slot | 4 bytes in the represented address model | Supplies a root-object address |
| Root object at +48 | 4 bytes | Supplies the context address |
| Context at +3936 | 4 bytes | Supplies an optional record address |
| Optional record at +24 | 4 bytes | Supplies the final destination value |

The context address and optional record address are treated as addresses by later reads. The last value is merely copied as a 32-bit quantity; its meaning is not established.

### Output and return

There is no explicit return value. Successful completion is observable through the three destination writes.

The first field preserves the selected 32-bit source value rather than normalizing it to one. The third field is also copied without interpretation. Signedness does not change the established branch behavior: the selected value and optional record address are tested only for equality with zero.

### Preconditions

For normal completion:

- Each executed destination write must target writable storage for four bytes.
- The source discriminator must be readable.
- The four-byte source value must be readable if the discriminator is nonzero.
- The global pointer chain must be readable to the depth actually traversed.

The function performs no explicit validation of the destination address, source address, global root address, or context address. The optional record address is the only address in the active chain explicitly tested for zero before its associated dereference.

The static representation does not establish alignment requirements, endianness, address-overflow behavior, or a portable language-level object model. Callers must provide storage and address relationships valid for the target environment.

## Behavior

### Selecting and storing the first value

The first observed access reads one byte at source offset +4.

If that byte is zero, the selected value is zero. The source field at offset +8 is not read on this branch.

If that byte is nonzero, the function reads four bytes at source offset +8 and uses that value as the selected value. Every nonzero discriminator has the same gating effect; there is no special handling for a particular nonzero byte.

The selected value is then written as four bytes at destination offset +4. This write occurs before any access to the global root.

### Clearing the remaining fields

If the selected value is zero, the function writes:

1. Four-byte zero at destination offset +8.
2. Four-byte zero at destination offset +12.

It then returns normally.

This path is taken both when the source discriminator is zero and when the discriminator is nonzero but the source value is zero. Neither case reads the global root or any object reachable through it.

### Populating the remaining fields

If the selected value is nonzero, the function:

1. Reads the root-object address from the global root slot.
2. Reads a context address from root-object offset +48.
3. Writes that context address to destination offset +8.
4. Reads an optional record address from context offset +3936.
5. If the optional record address is zero, selects zero for the final destination field.
6. Otherwise, reads four bytes at optional-record offset +24 and selects that value.
7. Writes the selected final value to destination offset +12.

There is no explicit zero check on the context address. A zero context therefore does not select a clean fallback. It is stored into the destination and then used as the base for the read at offset +3936. Whether that access succeeds or faults depends on the execution environment.

A zero optional record address does select a clean fallback: the final destination field becomes zero, without a read at optional-record offset +24.

### Observable access order

The operation is sequential, not an atomic replacement of the destination fields.

On the inactive path, the destination write order is +4, then +8, then +12.

On the active path:

- The source reads precede the destination +4 write.
- The destination +4 write precedes the global-root read.
- The context is read before being stored at destination +8.
- The destination +8 write precedes the read at context offset +3936.
- Any read at optional-record offset +24 precedes the final destination +12 write.

No destination field is explicitly read, but destination storage may overlap storage later read through another address.

### Overlapping storage

No explicit disjointness check or overlap prohibition appears in the supplied representation. Consequently, an overlapping-storage contract must preserve the actual read/write order rather than assume that every input is captured at entry.

If source and destination have the same base address, both source reads required by the selected branch occur before the first destination write. The function therefore uses the source discriminator and, where applicable, source value as read before those destination fields are overwritten.

Other overlap can affect later reads. For example, if destination offset +8 designates the same storage as context offset +3936, writing the context address changes the optional-record field before it is read. Similarly, a destination +4 write that overlaps the global root slot can change the root address subsequently obtained.

These are consequences of the represented access order, not evidence that such layouts are intended or valid in every language-level reconstruction.

## Side effects

On normal completion, the function performs exactly three explicit four-byte destination writes. It does not directly write the global root slot or any traversed object except through possible aliasing with the destination.

No allocation, deallocation, reference-count adjustment, callback, lock operation, or external function call is visible.

Copying an address into the destination does not establish an ownership transfer. The function neither acquires nor releases ownership of the context or optional record. Their lifetime requirements beyond the executed reads are unknown.

There is no explicit error status, rollback, or recovery path. An access failure can leave a partially updated destination:

- Failure while reading the source value occurs before any destination write.
- Failure during the global-root or root-object reads occurs after destination +4 has been written.
- Failure while reading context offset +3936 occurs after destination +4 and +8 have been written.
- Failure while reading optional-record offset +24 likewise occurs before the final destination write.

The environment’s fault handling and the atomicity of individual memory accesses are not established.

## Dependencies

There are no visible callee dependencies and therefore no additional opaque function IDs to enumerate.

The function depends on:

- Source storage with the required readable fields.
- Destination storage with three writable four-byte fields.
- A global root slot and its conditionally traversed object chain.
- Target memory semantics for address calculation, access validity, and faults.

No assumption about an unknown callee preserving globals or managing memory is needed: the supplied body contains no calls. Concurrent modification of the global chain remains unspecified.

## Acceptance examples

These examples are conditional static expectations, not executed tests. Unless an example states otherwise, all accessed storage is valid, disjoint, and unchanged by other actors.

### Zero discriminator bypasses the source value and global chain

Given a zero byte at source +4:

- Destination +4 becomes zero.
- Destination +8 becomes zero.
- Destination +12 becomes zero.
- Source +8 is not read.
- The global root is not read.

Readable source storage at +8 and a valid global chain are unnecessary for this branch.

### Nonzero discriminator with zero value remains inactive

Given a nonzero byte at source +4 and a zero 32-bit value at source +8:

- The source value is read.
- All three destination fields become zero.
- The global chain is not read.

A nonzero discriminator alone does not activate context traversal.

### Nonzero value with no optional record

Given a nonzero discriminator, selected value 7, a readable context address, and zero at context +3936:

- Destination +4 becomes 7, not 1.
- Destination +8 receives the context address.
- Destination +12 becomes zero.
- No optional-record payload read occurs.

### Nonzero value with an optional record

Given a nonzero discriminator, selected value with bit pattern `0x80000000`, a readable context, and a nonzero optional record whose +24 field contains `0xA1B2C3D4`:

- Destination +4 receives `0x80000000`.
- Destination +8 receives the context address.
- Destination +12 receives `0xA1B2C3D4`.

Neither high-bit-set value is rejected or normalized.

### Zero context does not terminate traversal

Given an active selected value and a zero context address:

- Destination +4 is written.
- Destination +8 is written with zero.
- The function attempts the context-relative read at byte offset +3936.
- Successful zeroing of destination +12 is not guaranteed.

### Same-base source and destination

Given identical source and destination base addresses, an initially nonzero discriminator, and an initial source +8 value of 9:

- The initial value 9 is read before destination +4 is overwritten.
- The active branch is selected.
- The later destination +8 write replaces the original source-value field.

The function does not reread that source-value field.

## Uncertainties

**documented_only — Field meanings.** The discriminator may represent validity or enablement, and the destination may be a compact derived-state record. These are inferred interpretations, not established semantics.

**documented_only — Address model.** The supplied representation uses 32-bit integer quantities as addresses. The original declared pointer types, ABI details, and valid address range are not independently established.

**documented_only — Ownership and lifetime.** The copied context address may refer to borrowed or otherwise managed storage. No lifetime extension or release obligation is shown.

**documented_only — Concurrency.** No synchronization or stable-snapshot guarantee is visible. Behavior under concurrent mutation is not established.

**documented_only — Exceptional behavior.** Exact fault delivery, partial-access effects, and recovery by the surrounding program are unknown.

**documented_only — Intended aliasing.** Sequential overlap effects are visible in the representation, but supported application-level overlap arrangements are not specified.

## Evidence

The established behavior comes solely from the supplied static function representation: its access widths, offsets, zero tests, branches, and read/write ordering.

The representation supports a one-byte discriminator read, conditional four-byte source read, three four-byte destination writes, and conditional global pointer traversal. It contains no visible calls or explicit memory-management operations.

Semantic names such as “context” and “optional record” describe roles in that traversal, not recovered type identities. No runtime execution or testing has been performed.
