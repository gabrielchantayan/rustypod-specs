<!-- Trusted provenance: F-a7c7afa4b061; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a7c7afa4b061

## Purpose

F-a7c7afa4b061 selects an object using a signed index and a globally supplied base address, then writes the 32-bit integer value **1** at byte offset **2052** within the selected object.

The established operation has three dependent memory-access stages: read the global base value, read an object address from the indexed entry, and write the destination field. There is no visible validation branch or conditional update.

The evidence does not establish the object’s type or the destination field’s meaning. Describing the field as an enable flag, readiness marker, or particular status would be inferred intent rather than established behavior.

## Interface

### Explicit input

The function accepts one **signed 16-bit integer**, with representable values from **−32768 through 32767**.

The input is used as an index with a **396-byte stride**. No visible check restricts it to nonnegative values or compares it with a collection bound.

### Implicit input state

The function reads a **global base value** and uses that value as the base address for indexed entry selection. This is a logical read of global state, not merely a reference to the address at which that global state resides.

At the selected entry address, the function reads a **32-bit address-bearing value**. That value identifies the selected object for the subsequent destination-address calculation.

The operation therefore depends on:

- Global storage supplying the base value.
- Readable entry storage at the base plus the signed index multiplied by 396 bytes.
- Writable destination storage 2052 bytes beyond the object address read from that entry.

The global base value’s declaration and storage width are not independently provided. Its role as the base value read by the expression is established; its volatility, mutability, initialization rules, and synchronization requirements are not.

### Output

The function returns **no explicit value**.

On normal completion, it has written the 32-bit integer value 1 to the selected destination. It does not return the previous field value, the selected object address, or a success indicator.

### Operational preconditions

For the specified write to complete normally:

- The global base value must be accessible.
- The indexed entry must support the required 32-bit read.
- The object address read from that entry, plus 2052 bytes, must identify storage supporting the required 32-bit write.
- Address formation, alignment, permissions, and storage lifetime must permit those operations.

These are access requirements, not validations performed by the function. The evidence supplies no application-level valid index range.

## Behavior

### Global base acquisition

The function first obtains the global base value used in the entry-address calculation.

This global read is part of the established logical behavior. Uncertainty about whether the global is volatile, externally mutable, or implemented through a particular machine-level memory transaction does not remove that dependency.

The value obtained supplies the base for this invocation’s indexed entry selection. No second logical read of the global base is visible in the expression.

### Indexed selection and arithmetic

The selected entry address is the global base value displaced by the signed input multiplied by **396 bytes**.

Consequences include:

- Input 0 selects the entry at the acquired base.
- Input 1 selects the entry 396 bytes after that base.
- Input 2 selects the entry 792 bytes after that base.
- Input −1 selects the entry 396 bytes before that base.

There is no visible conversion of the index to an unsigned 16-bit quantity before multiplication. Negative inputs therefore produce negative displacements rather than large positive indices.

Across the full signed 16-bit input domain, the displacement ranges from:

- **−12,976,128 bytes**, for input −32768.
- **12,975,732 bytes**, for input 32767.

Both products fit within a signed 32-bit integer. The index multiplication itself therefore does not overflow a signed 32-bit result for any representable input.

That arithmetic bound does not establish the validity of adding the displacement to the base. It also does not establish the validity of adding 2052 bytes to the subsequently read object address. Address-space boundaries, pointer representation, and access validity remain relevant.

The 396-byte stride establishes the spacing between selected entry addresses. It does not establish a complete record layout, a record count, or the meaning of the intervening bytes.

### Logical read/write order

The established dependent access sequence is:

1. Read the global base value.
2. Read the 32-bit object address at the indexed location relative to that acquired base.
3. Write the 32-bit integer value 1 at byte offset 2052 from the object address just read.
4. Return without an explicit result.

Each later memory-access address depends on the value obtained by the preceding read.

The destination is relative to the **selected object address**, not relative to the global base or the selected entry address. The field offset is measured in bytes.

No destination-field read is visible before the write. No independent reload of the global base or indexed object address is visible after the destination has been determined.

This sequence describes logical accesses and their dependencies. It does not specify the number of physical bus transactions, instruction encodings, or additional ordering guarantees for unrelated memory.

### Destination update

The function assigns the complete **32-bit integer value 1** to the destination. The write covers the four bytes beginning at object-relative offset 2052.

The prior field value does not control the operation:

- A prior value of 0 is replaced with 1.
- A prior value of 7 is replaced with 1.
- A prior value of 1 is written with 1 again.

The operation is not evidenced as a bitwise update, increment, toggle, or read-modify-write. No preservation of other bits within that 32-bit destination is specified.

There is no visible early return for an already-set field.

### Aliasing and completion

No non-aliasing restriction is visible. The destination may overlap an entry, the global base storage, or another externally meaningful location if the supplied addresses produce that overlap.

Such an overlap does not introduce a visible additional read. The destination write uses the values already obtained through the preceding dependent reads. If that write changes global or entry storage, it can affect later invocations.

There is no explicit error-handling path. An inaccessible global, unreadable entry, unwritable destination, invalid address calculation, or unsuitable alignment has no specified recovery behavior in the supplied evidence.

The function does not visibly reject invalid indices. Nor does the evidence guarantee a particular exception, signal, or other outcome when an access cannot complete.

## Side effects

The explicit mutation is one logical **32-bit store of integer value 1** to the computed destination.

The function also performs the two logical reads described above: the global base-value read and the indexed object-address read.

Additional boundaries are:

- **Ownership:** No allocation, retention, ownership transfer, release, or deallocation is visible. The function uses existing storage; the evidence does not establish the ownership rules governing it.
- **Global state:** The base is explicitly read from global state. No separate write to that global is visible, but the destination can affect it if the addresses overlap.
- **Entry storage:** The selected entry is read rather than explicitly replaced. Destination aliasing can nevertheless alter it.
- **Other fields:** No additional field update is visible. Preservation of overlapping storage cannot be promised.
- **Synchronization:** No lock, barrier, atomic primitive, or synchronization protocol is visible. The function alone establishes no atomicity or inter-thread publication guarantee.
- **Repeated writes:** An existing destination value of 1 does not suppress the specified write.
- **Representation:** The integer value and four-byte write width are established. Byte order is not independently established by the supplied evidence.

Repeated invocations with the same index need not target the same object. Selection depends on the global base value and the indexed entry value obtained for each invocation.

## Dependencies

No called functions are visible. There are therefore no additional opaque function IDs to list.

The operation’s data dependencies are:

| Dependency | Established use |
|---|---|
| Global base storage | Supplies the base value through a logical read |
| Indexed entry storage | Supplies a 32-bit object address at a 396-byte-stride location |
| Selected destination storage | Receives the 32-bit integer value 1 at object-relative byte offset 2052 |
| Execution environment | Determines whether address formation and the required accesses are valid |

The indexed address-bearing read is 32 bits wide. The destination write is also 32 bits wide. The evidence does not define a corresponding layout or adaptation for an environment with wider pointers.

There are no unknown callees whose global-state preservation or memory ownership behavior must be assumed.

## Acceptance examples

These are conditional contract examples, not reports of executed tests. Successful cases require valid address formation and accessible storage throughout the relevant accesses.

### Zero index

Given an accessible global base value, a readable object address at that base, and a writable destination 2052 bytes beyond that object address:

- Input 0 obtains the global base value.
- The object address is read from the location identified by that base.
- The destination receives the 32-bit integer value 1.
- The function returns no explicit value.

### Positive index

For input 2:

- The entry address is 792 bytes after the acquired global base.
- The function reads the object address from that entry.
- It writes 1 at byte offset 2052 from the object address, not from the entry address.

### Negative index

If the location 396 bytes before the acquired global base contains a readable, usable object address, input −1 selects that location and writes 1 to its selected destination.

If that location is inaccessible, successful completion is not predicted. The function has no visible negative-index rejection before the attempted entry access.

### Full-domain arithmetic

Input −32768 produces a displacement of −12,976,128 bytes. Input 32767 produces a displacement of 12,975,732 bytes.

These products fit within signed 32-bit arithmetic. Acceptance of either complete invocation still depends on the resulting addresses and storage; arithmetic fit alone is insufficient.

### Existing destination value

For an accessible selected destination initially containing 0, 7, or 1, the resulting 32-bit field value is 1. None of those prior values controls a visible branch.

### Global base changed between invocations

Suppose the global base supplies one value for a first invocation and a different value for a later invocation.

With the same explicit index, the later invocation selects its entry relative to the newly obtained base. The contract does not treat the global base as a permanently fixed address constant.

### Entry changed between invocations

If the global base is unchanged but the selected entry supplies different object addresses on successive invocations, the same index can update different objects.

### Overlapping destination

If the computed destination overlaps the selected entry or global base storage, the four-byte destination write affects that overlapping storage.

The current write remains based on the values already read. Future invocations may select differently as a consequence of the changed storage.

## Uncertainties

The following remain **documented_only**:

- The selected object’s type and the semantic meaning of the destination field.
- The global base storage’s declared width, initialization, mutability, and volatility.
- The collection’s bounds, allocation layout, and application-level valid indices.
- Alignment requirements, byte order, and access-failure behavior.
- Address-overflow or wrapping behavior outside valid address formation.
- Caller-supplied lifetime and ownership rules.
- Whether aliases or concurrent modifications are permitted by the surrounding application.
- Any platform-provided atomicity, visibility, or synchronization guarantees.
- The physical memory transactions implementing the logical reads and write.
- Any application restriction narrowing the signed 16-bit input domain.

These unknowns do not make the logical global base read uncertain. They limit the surrounding storage and execution contract, not the established dependency sequence.

## Evidence

**Established static evidence:** one signed 16-bit input; a logical read of a global base value; indexed selection using a 396-byte stride; a dependent 32-bit read used as an object address; a 2052-byte object-relative field offset; one unconditional 32-bit store of integer value 1; no visible calls or validation branches; and return without an explicit value.

**Arithmetic derived from that evidence:** the signed index displacement ranges from −12,976,128 through 12,975,732 bytes, with both endpoints fitting within a signed 32-bit integer.

**Inferred intent:** writing 1 may mark a state, but the evidence does not establish which state or any wider protocol.

**Verification limit:** this contract is based solely on the supplied static source evidence. No runtime tests have been performed.
