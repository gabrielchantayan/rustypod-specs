<!-- Trusted provenance: F-b9cde51cedf7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b9cde51cedf7

## Purpose

F-b9cde51cedf7 populates seven consecutive integer slots in caller-supplied storage. The recorded values combine explicit inputs, the destination’s own address, two shifted values, and two values taken from entry machine state outside the declared argument list.

The available static representation shows seven stores followed by a terminal bad-instruction indication. It does not establish a successful return, a recoverable error result, or usable execution beyond that point.

The observable operation resembles construction of a compact state record. That interpretation is **inferred intent**, not an established description of the record’s purpose. No evidence identifies the record as a particular application object, processor frame, allocator structure, or protocol message.

This contract distinguishes:

- **Established static evidence:** operations and ordering directly represented in the supplied material.
- **Conditional interpretation:** behavior that depends on the target’s integer widths, pointer representation, or machine execution semantics.
- **documented_only unknowns:** details that cannot be resolved from the supplied material.

No runtime tests have been performed.

## Interface

### Explicit inputs

The declared interface has four arguments, in this order:

| Position | Role | Represented type | Use |
|---|---|---|---|
| 1 | Primary value | Signed integer | Stored directly in slot 0 and used to derive slot 5 |
| 2 | Destination | Pointer to signed integers | Supplies the storage base and the address value recorded in slot 1 |
| 3 | Secondary value | Signed integer | Stored directly in slot 2 |
| 4 | Unsigned control value | Unsigned integer | Shifted right by two positions and recorded in slot 3 |

The supplied representation uses integer types without an explicit target data model. A 32-bit integer model is a plausible interpretation, but the text alone does not independently establish it.

Under the **conditional 32-bit model**, each destination slot occupies four bytes, the seven-slot region occupies 28 bytes, and the store offsets are 0, 4, 8, 12, 16, 20, and 24 bytes from the destination.

Without that model, the established layout is seven consecutive integer-sized slots, not an unconditionally proven 28-byte structure.

### Additional entry-state inputs

Two other values affect the output:

1. An unsigned entry-state value, shifted right by 22 bit positions before storage.
2. A signed entry-state value, stored without a visible transformation.

Neither value is supplied through the four declared arguments. Their provenance is **documented_only**: they might reflect an actual machine-level calling convention, context established by earlier execution, or incompleteness in the recovered interface. They must not be assumed to be zero, initialized, caller-independent, or stable across invocations.

Accordingly, the seven recorded values cannot be determined from the four explicit arguments alone.

### Output and completion

There is no declared return value.

The concrete output visible in the supplied representation is the sequence of writes to destination storage. No success code, returned pointer, byte count, or error flag is shown.

Normal completion is not established. After the seventh store, the representation ends in a terminal bad-instruction operation. Whether actual execution traps, stops, transfers to an exception handler, or behaves differently is **documented_only**.

### Storage preconditions and ownership

For all seven stores to complete, the destination must identify storage that permits integer-sized writes at each of the seven slot locations. Any required alignment follows the actual target’s memory-access rules; those rules are not supplied.

There is no visible null check, bounds check, allocation, or writability check.

The function uses caller-supplied storage. No visible operation allocates, releases, retains, or transfers ownership of that storage. These absences do not establish what an unknown fault handler or terminal operation might do.

## Behavior

### Ordered record construction

The visible store sequence is:

| Order | Slot | Recorded content |
|---|---|---|
| 1 | 0 | Primary value |
| 2 | 1 | Destination address converted to a signed integer |
| 3 | 2 | Secondary value |
| 4 | 3 | Unsigned control value shifted right by 2 |
| 5 | 4 | Unsigned entry-state value shifted right by 22 |
| 6 | 5 | Primary value shifted left by 11 |
| 7 | 6 | Signed entry-state value |

Each slot is written once in the supplied sequence. There is no visible read of a previous destination-slot value, no merge with existing contents, and no initialization of storage outside these seven slots.

The destination address is therefore both an addressing input and recorded data. Relocating the destination can change slot 1 even when all other inputs are unchanged.

The listed order is the order represented by the supplied static material. It is not evidence of cross-thread publication order, hardware barriers, or indivisibility of the whole record.

### Integer widths and transformations

The right shift producing slot 3 operates on an unsigned input. At the represented expression level, it discards the two least significant bits and introduces zeros at the high end. For an unsigned value with sufficient width, its numerical result is the input divided by four with any fractional remainder discarded.

The right shift producing slot 4 also operates on an unsigned value. It discards 22 low-order bits. Its numerical result is the entry-state value divided by 4,194,304 with any fractional remainder discarded.

Under the conditional 32-bit model:

- Slot 3’s result ranges from 0 through 1,073,741,823.
- Slot 4’s result ranges from 0 through 1,023.
- Both results fit in a 32-bit signed integer.

These ranges depend on the 32-bit assumption. A wider unsigned input could produce a shifted result that does not fit in the signed destination type, introducing an additional conversion question.

Slot 5 is derived from a signed input shifted left by 11. For nonnegative primary values whose product by 2,048 remains representable, the numerical interpretation is multiplication by 2,048.

Under a 32-bit signed model, that straightforward interpretation covers primary values from 0 through 1,048,575 inclusive.

The representation does not justify treating every other primary value as a defined wraparound computation. In particular, negative inputs and positive inputs whose shifted result exceeds the signed range require actual target instruction semantics to resolve. A machine instruction might retain only a fixed-width bit pattern, but that is not established merely by the signed expression shown.

### Address recording

Slot 1 receives an integer conversion of the destination pointer, not a value loaded from destination memory.

The result is not established as a portable address encoding. Its interpretation depends on:

- Pointer width.
- Signed integer width.
- Rules for converting a pointer to an integer.
- Whether the pointer value is representable in the destination integer type.

For a compatible 32-bit address model, the slot may preserve the destination’s address bits. That is a conditional interpretation, not an unconditional guarantee. No evidence specifies behavior for a wider pointer or a conversion requiring loss of information.

### Errors and interrupted execution

There is no visible validation before the first store. An invalid destination is not converted into a documented status result.

If a memory access fails, later stores are not established to occur. The representation contains no rollback path. Earlier successful stores can therefore leave a partially updated record.

The same limitation applies if the destination refers to memory with special access semantics. The supplied material does not establish support for device memory, trapping memory, concurrent mutation, or observation between stores.

After the seventh store, no further ordinary operation is available for analysis. In particular, the terminal indication must not be interpreted as a demonstrated successful return.

## Side effects

### Destination memory

The established visible data side effect is seven consecutive integer-slot stores. Existing contents of those slots are replaced.

There is no visible destination read, append behavior, conditional omission of a slot, or write outside the seven-slot region.

The complete update is not shown as a transaction. A consumer must not infer that the record becomes visible all at once.

### Execution state

The function reaches a terminal bad-instruction indication after the writes in the supplied representation.

The resulting process, thread, exception, and control-transfer effects are **documented_only**. No evidence establishes whether a handler runs, whether execution can resume, or whether the environment terminates.

### Ownership, synchronization, and external effects

No allocation, deallocation, lock operation, synchronization barrier, file operation, or network operation is visible before the terminal indication.

This is a statement about the supplied material, not a guarantee about unseen execution or the implementation of the terminal operation. No preservation of unrelated globals, external state, or storage ownership is established for that operation.

## Dependencies

No ordinary callable dependency with a supplied opaque function ID appears in the available body.

The terminal bad-instruction operation has no supplied dependency mapping. Its identity and implementation remain **documented_only**; assigning it an invented opaque function ID would imply unsupported provenance.

Execution also depends on target facilities rather than identified callees:

- Integer and pointer widths.
- Integer conversion rules.
- Shift instruction semantics.
- Memory-access and alignment behavior.
- Delivery of the two additional entry-state values.
- Fault or terminal-operation handling.

No assumptions about global preservation or memory ownership follow from these unresolved dependencies.

## Acceptance examples

These are conditional acceptance criteria, not observed test results. Unless stated otherwise, they assume 32-bit integers, a compatible destination-address conversion, writable storage, no access faults, and entry-state values available as described.

### Fully determined record on a bounded input

Given:

- Primary value: 3.
- Secondary value: −7.
- Unsigned control value: 15.
- Unsigned entry-state value: 12,582,912.
- Signed entry-state value: −9.
- Destination address numerically representable as 4,096.

The expected slots, in store order, are:

| Slot | Expected value |
|---|---:|
| 0 | 3 |
| 1 | 4,096 |
| 2 | −7 |
| 3 | 3 |
| 4 | 3 |
| 5 | 6,144 |
| 6 | −9 |

Acceptance of the record contents does not establish normal return after the final store.

### Low-bit discard in the control value

With all other inputs held fixed, unsigned control values 8, 9, 10, and 11 each produce 2 in slot 3.

This criterion checks discarded low bits, not rounding to the nearest integer.

### Entry-state shift boundary

Unsigned entry-state values below 4,194,304 produce zero in slot 4. A value of exactly 4,194,304 produces one.

This criterion depends on the entry-state value actually being supplied; substituting an assumed default is not acceptable.

### Distinct destinations

Two invocations with identical explicit value inputs and identical entry-state values, but different representable destination addresses, produce different slot-1 address values under the compatible address model.

The remaining slot values are unchanged under those assumptions.

### Signed left-shift boundary

A primary value of 1,048,575 produces 2,147,481,600 in slot 5 under the 32-bit signed model.

For a primary value of 1,048,576, this contract does not prescribe a wrapped negative result. The behavior is **documented_only** until the actual instruction semantics are established.

### Partial update

If the first three stores succeed and the fourth store faults, slots 0 through 2 have already been assigned in the represented sequence. No rollback or later-slot update is promised.

The environment’s fault response is outside the established behavior.

## Uncertainties

The following remain **documented_only**:

- Actual integer and pointer widths, byte order, and alignment requirements.
- Provenance and validity of the two additional entry-state inputs.
- Whether the recovered argument list fully describes the machine-level interface.
- Pointer-to-integer conversion behavior outside directly representable addresses.
- Exact behavior of the signed left shift for negative or overflowing inputs.
- Fidelity of the recovered sequence near the reported bad instruction.
- Actual execution after the seventh store.
- Fault delivery, handler behavior, and possible resumption.
- Concurrent visibility and any permitted use with special memory.
- Application-level meaning of the seven-slot record.

None of these uncertainties is resolved by assuming conventional initialization, normal return, ownership transfer, or preservation of global state.

## Evidence

The supplied static material directly supports:

- Four declared arguments and no declared return value.
- Two additional entry-state values used as inputs.
- Seven destination stores in the order documented above.
- Recording of the destination address through an integer conversion.
- Unsigned right shifts by 2 and 22 positions.
- A signed left shift by 11 positions.
- A terminal bad-instruction indication after the final visible store.
- No visible validation, destination reads, rollback, allocation, release, or ordinary return path.

The 32-bit byte layout, the record-construction purpose, and possible fixed-width machine interpretations are conditional or inferred, not independent observations.

No implementation access, runtime execution, or runtime tests were used to establish this contract.
