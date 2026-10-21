<!-- Trusted provenance: F-efd0aa934fd7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-efd0aa934fd7

## Purpose

F-efd0aa934fd7 updates a caller-supplied address slot and a four-byte field at byte offset four within one or two addressed records.

Its established behavior is an ordered sequence of memory accesses:

- Read the slot once and retain its initial value.
- If that value is nonzero, write the supplied record address to the field at the retained address plus four.
- Otherwise, write the supplied record address to the slot immediately.
- Clear the supplied record’s offset-four field.
- If the retained initial value was nonzero, write the supplied record address to the slot last.

For nonoverlapping storage, this leaves the slot holding the supplied record address, the supplied record’s offset-four field holding zero, and any previous record’s offset-four field holding the supplied record address.

The operation resembles appending a record through a maintained tail slot. That is inferred intent, not an established container contract. Aliasing can produce results inconsistent with an abstract append operation, including a final self-link.

## Interface

### Inputs

The function accepts three arguments:

| Argument | Established role |
|---|---|
| First argument | An unused four-byte value |
| Slot address | Identifies storage read and written as one four-byte value |
| Supplied record address | A four-byte value stored in memory and used to form the address of a four-byte field at byte offset four |

The **previous address** means the four-byte value initially read from the slot. Zero selects the initially-zero branch. Every nonzero value selects the other branch; the function performs no separate validity test.

The first argument is not inspected and does not contribute to address formation, branch selection, or stored values.

The supplied representation establishes four-byte slot contents, record-address values, and field accesses. It does not establish the width of the slot-address argument itself, the calling convention, or how these values relate to pointers on a wider platform.

### Output

There is no explicit return value or status result. Normal completion is observable through the ordered memory writes.

### Accessed layout

| Region | Width | Use |
|---|---:|---|
| Slot | Four bytes | Read the previous address; store the supplied record address |
| Previous address plus four bytes | Four bytes | Store the supplied record address, only when the previous address is nonzero |
| Supplied record address plus four bytes | Four bytes | Store zero |

Each offset-four field occupies bytes four through seven relative to its record address. The offset is measured in bytes, not records or four-byte elements.

No offset-zero record field is directly accessed as such. Nevertheless, any accessed region may overlap other caller-visible storage.

The evidence does not establish byte order. Exact overlap of complete four-byte regions can be described using stored values; partial overlap requires additional representation information.

### Preconditions for normal completion

The selected accesses must be valid in the execution environment:

- The slot must support a four-byte read and a four-byte write.
- The supplied record’s offset-four field must support a four-byte write.
- When the initial slot value is nonzero, the previous record’s offset-four field must support a four-byte write.
- Address formation and any applicable alignment requirements must be satisfied.

These are requirements for normal completion, not checks performed by the function. A nonzero address is not validated. A zero supplied record address is not rejected.

## Behavior

### Initial read and retained branch condition

The first memory access reads the slot’s four-byte value. Both subsequent conditional decisions use that retained value.

There is no second slot read. A direct or aliased write that changes the slot does not change which later conditional action is selected.

In particular, an initially-zero slot receives an early write but no final slot write. An initially-nonzero slot receives a final slot write even if earlier writes have changed its contents to zero.

### Initially nonzero slot

When the retained previous address is nonzero, the ordered accesses after the initial read are:

1. Store the supplied record address in the previous record’s offset-four field.
2. Store four-byte zero in the supplied record’s offset-four field.
3. Store the supplied record address in the slot.

The previous field value and supplied field value are not read before replacement.

With three distinct, nonoverlapping accessed regions, the final state is:

- Previous record’s offset-four field: supplied record address.
- Supplied record’s offset-four field: zero.
- Slot: supplied record address.

Those final field values require the stated nonoverlap condition. The last slot write can otherwise overwrite a field written earlier.

### Initially zero slot

When the retained previous address is zero, the ordered accesses after the initial read are:

1. Store the supplied record address in the slot.
2. Store four-byte zero in the supplied record’s offset-four field.

There is no write through the previous address and no final slot write.

With nonoverlapping storage, the slot ends with the supplied record address and the supplied record’s offset-four field ends with zero. If the field overlaps the slot, the clearing write occurs last and can alter the value just stored in the slot.

### Aliasing and last-write effects

No alias check is present. For valid accesses, the ordered writes—not an assumed list invariant—determine the result.

**Previous and supplied record addresses are equal and nonzero.** Their shared offset-four field first receives that address and then receives zero. The field is therefore temporarily cleared.

Its final value depends on the last slot write:

- If the slot does not overlap the shared field, the field remains zero. No final self-link remains.
- If the slot exactly occupies the shared field, the last slot write restores the record address. The final field is a self-link.
- If the slot partially overlaps the shared field, the last write can alter part of the cleared field. Its resulting value is not established without byte-order and execution-environment details.

Equality of the two record addresses alone therefore does not establish whether a self-link survives.

**The slot exactly occupies the supplied record’s offset-four field.** If the slot initially contains zero, the function writes the supplied record address there and then clears the same location. Its final value is zero. If the initial value is nonzero, the clearing write is followed by the final slot write, so the shared location ends with the supplied record address.

**The slot exactly occupies the previous record’s offset-four field.** In the nonzero branch, the first write changes the slot. The retained initial value still controls the final conditional write. If the supplied record’s field is a separate region, that field is cleared and the slot ends with the supplied record address. If the supplied field also aliases this location, the three-write interaction described above applies.

These cases may coincide; they are not mutually exclusive. In particular, equal record addresses and a slot located at their shared offset-four field produce three successive writes to one location.

### Address arithmetic and boundary values

Both field addresses are formed by adding four to an address-bearing value. No range check precedes either operation.

The supplied representation uses four-byte integer-like values for the retained previous address and supplied record address. It does not establish a portable rule for overflow, high-bit-set values, or conversion from those values to usable addresses. Modular wrapping, trapping, and any specific high-address interpretation remain unresolved.

A supplied record address of zero still selects a clearing access at the address formed from zero plus four. It also causes zero to be stored wherever the supplied address is assigned. Whether that field access is valid is an environmental matter, not a special case handled by the function.

## Side effects

### Memory reads and writes

On an uninterrupted, normally completing invocation, the function performs:

- One four-byte read from the slot.
- Two four-byte writes when the retained value is zero.
- Three four-byte writes when the retained value is nonzero.

These are operation counts, not counts of distinct locations or changes in value. Several writes may target the same location, and a write may reproduce an existing value.

There are no direct calls, explicit allocation or release operations, or direct accesses to named global storage in the supplied body. Indirectly addressed storage could belong to a larger object or global region; the body does not identify its provenance.

### Ownership and lifetime

The function stores addresses and overwrites fields. It does not establish memory ownership, transfer rights, allocation origin, or release obligations.

The caller must arrange live, accessible storage for the selected accesses. Any surrounding convention under which insertion transfers logical ownership is outside the established behavior.

### Errors and partial state

There is no explicit validation failure, error return, recovery path, or rollback.

If an access fails, the body provides no mechanism to undo earlier completed writes:

- In the initially-zero branch, the slot assignment precedes the clearing access.
- In the nonzero branch, the previous field assignment precedes the clearing access.
- The nonzero branch’s final slot assignment is reached only after the preceding field accesses complete.

The environment determines fault delivery, continuation behavior, and whether a failing access can modify memory partially. The function does not establish those guarantees.

### Concurrency

No locking, atomic operation, or memory-ordering primitive is evidenced.

The listed order describes the supplied sequential behavior. It does not guarantee indivisible four-byte accesses, cross-thread visibility order, or safety against concurrent mutation. The function does not detect an intervening change to the slot.

## Dependencies

No called dependency is present in the supplied body. There are therefore no additional opaque function IDs to specify.

Execution still depends on the environment’s address interpretation, memory accessibility, alignment rules, byte order, and fault behavior. Those dependencies are environmental requirements, not evidenced function calls.

## Acceptance examples

These are conditional acceptance scenarios derived from static evidence, not executed tests. Unless an example states otherwise, all selected accesses are valid, address formation is valid, storage regions do not overlap, and execution is uninterrupted.

### Initially zero slot

**Given:** The slot contains zero.

**Required:** Store the supplied record address in the slot, then clear the supplied record’s offset-four field. Do not write through the zero previous address. Do not perform a later slot write.

### Distinct previous and supplied records

**Given:** The slot contains a nonzero previous address different from the supplied record address.

**Required:** Store the supplied record address in the previous record’s offset-four field, clear the supplied record’s offset-four field, and finally store the supplied record address in the slot.

The old contents of either field do not affect the sequence.

### Equal record addresses with a separate slot

**Given:** The slot contains the nonzero supplied record address, and the slot does not overlap that record’s offset-four field.

**Required:** Write the record address to its offset-four field, clear the same field, then write the record address to the separate slot.

**Final state:** The field is zero and the slot holds the record address. A final self-link in the field would contradict these conditions.

### Equal record addresses with an aliased slot

**Given:** The supplied record address is 0x1000, the slot begins at 0x1004, and the slot initially contains 0x1000. All selected accesses are valid.

**Required:** Perform three four-byte writes to the location beginning at 0x1004, in this order:

1. 0x1000.
2. Zero.
3. 0x1000.

**Final state:** The slot and record’s offset-four field are the same location and contain 0x1000. The field was temporarily cleared, but the final slot write restores a self-link.

### Initially zero slot aliases the supplied field

**Given:** The slot exactly occupies the supplied record’s offset-four field and initially contains zero.

**Required:** Store the supplied record address at the shared location, then clear it.

**Final state:** The shared location contains zero. The early slot assignment does not survive.

### Changing only the unused argument

**Given:** Two invocations differ only in the first argument and have otherwise identical relevant memory.

**Required:** Subject to the same successful-access conditions, both invocations select the same branch and perform the same ordered accesses with the same stored values.

## Uncertainties

The following remain **documented_only**:

- Complete record layout and the semantic meaning of the offset-four field.
- Whether the slot represents a tail, cursor, or another structure.
- Caller restrictions on aliasing, duplicate insertion, zero addresses, or alignment.
- Exact calling convention and slot-pointer representation.
- Overflow rules and interpretation of high-bit-set address values.
- Byte order and final values under partial overlap.
- Memory-access atomicity, fault behavior, and concurrent visibility.
- Ownership, allocation, and lifetime conventions outside this function.

The append-like interpretation is inferred intent only. It neither establishes a list invariant nor overrides the alias-sensitive write sequence.

## Evidence

**Established static evidence:** The supplied body reads the slot once, retains the resulting four-byte value, and uses its zero/nonzero classification for both conditional actions. It performs an early write either to the previous record’s offset-four field or to the slot, then clears the supplied record’s offset-four field, then conditionally writes the slot. There are no calls and no explicit return value.

The equal-record, aliased-slot self-link follows from that ordering: the shared field receives the record address, then zero, then the record address again. The conclusion that equal records leave a cleared field applies only when the final slot write does not overlap that field.

**Inferred intent:** With nonoverlapping storage, the operation resembles linking a new terminal record after a previous terminal record and updating a tail slot.

**Verification limit:** No runtime tests have been performed. All acceptance examples are conditional consequences of static access ordering, not observed execution results.
