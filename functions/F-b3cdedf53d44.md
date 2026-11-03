<!-- Trusted provenance: F-b3cdedf53d44; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b3cdedf53d44

## Purpose

F-b3cdedf53d44 performs a sparse update of caller-provided storage using two integer inputs and one captured global value. It distributes selected input bits across five byte fields, writes a global-derived 16-bit field, and writes a terminal zero byte.

When both integer inputs have zero low-order 16-bit portions, the routine substitutes decimal 33 for the first input before deriving its output fields. The second input is not replaced.

These are established transformation and memory-access behaviors. The supplied evidence does not establish an external format, application domain, or semantic meaning for the encoded fields.

## Interface

### Explicit inputs

- **First input:** A 32-bit integer. Its low 16 bits determine whether substitution applies and supply the first-input output fields.
- **Second input:** A 32-bit integer. Its low 16 bits determine whether substitution applies and supply the second-input output fields.
- **Destination:** A caller-supplied byte-addressed memory location. The routine writes selected locations within the nine-byte span beginning there.

The upper 16 bits of either integer do not affect the resulting destination values under the represented fixed-width operations.

### Implicit input

The routine reads a 32-bit global value before its destination writes. Only the low 16 bits of that captured value affect the stored result.

The global location’s initialization, access permissions beyond readability, and concurrent-access properties are not supplied.

### Output layout

The routine returns no value. Its output is the following memory update:

| Destination location | Access width | Result |
|---|---:|---|
| Offset 0 | 8 bits | Twice the value represented by second-input bits 0–4 |
| Offset 1 | 8 bits | Value represented by second-input bits 5–10 |
| Offset 2 | 8 bits | Value represented by second-input bits 11–15 |
| Offset 3 | 8 bits | Value represented by effective-first-input bits 0–4 |
| Offset 4 | 8 bits | Value represented by effective-first-input bits 5–8 |
| Offset 5 | No access | Not read or written by the supplied body |
| Offsets 6–7 | One 16-bit access | Captured global contribution plus effective-first-input bits 9–15, reduced to 16 bits |
| Offset 8 | 8 bits | Zero |

Bit numbering begins with bit 0 as the least significant bit. The **effective first input** is the original first input unless the substitution condition applies.

The 16-bit field is a native-width memory store, not two separately specified byte stores. Its physical byte ordering is not established.

### Preconditions and ownership

For normal completion, the global location must support the represented read, and the destination must support every listed write. A writable nine-byte destination span satisfies the bounds requirement, but does not by itself establish suitable alignment or other target-specific access permissions.

The address at offset 6 must permit a 16-bit access under the target’s rules. Null, inaccessible, undersized, or improperly aligned destinations have no established successful behavior.

No allocation, release, ownership transfer, or retention of the destination pointer appears in the supplied body. The caller remains responsible for storage lifetime. No non-aliasing requirement is explicitly supplied.

## Behavior

### Capturing the global input

The global read occurs before the destination writes and before the input-processing operations represented in the supplied body. Its captured value supplies the later 16-bit result; there is no second global read.

Consequently, if the destination overlaps the global location and that overlap is valid in the target environment, earlier destination writes do not replace the value already captured for this invocation. They may still modify the underlying global storage.

This is an access-order property, not a synchronization guarantee.

### Substitution condition

Substitution occurs exactly when both of these conditions hold:

- The first input’s low 16 bits are zero.
- The second input’s low 16 bits are zero.

The effective first input then becomes decimal 33, whose low 16-bit representation is 0x0021. Otherwise, the first input remains unchanged.

Nonzero upper halves do not prevent substitution. Conversely, a zero low half in the first input alone is insufficient when the second input’s low half is nonzero.

In the substitution case, the second-input fields are all zero. The effective first input produces 1 at offset 3, 1 at offset 4, and a zero contribution to the 16-bit field.

### Second-input fields

The three second-input fields collectively carry all 16 low-order input bits:

- Offset 0 carries the lowest five bits, shifted one position toward higher significance. Its range is 0–62; it is always even.
- Offset 1 carries the next six bits. Its range is 0–63.
- Offset 2 carries the highest five bits of the low 16-bit portion. Its range is 0–31.

Offset 0’s least significant bit is always zero. Bits above the stated field widths are also zero in these destination bytes.

The field at offset 2 excludes every input bit above bit 15. The represented operation narrows the relevant intermediate to a byte before its final shift; treating it as an unrestricted shift of the entire input would give the wrong contract.

### First-input fields

The effective first input is partitioned into:

- Five bits for offset 3, with range 0–31.
- Four bits for offset 4, with range 0–15.
- Seven bits for the 16-bit field’s increment, with range 0–127.

Together these cover the effective first input’s low 16 bits. Its upper half contributes nothing to the output.

### Global-derived arithmetic

The increment is the unsigned value represented by effective-first-input bits 9–15. It is added to the captured global value’s low 16-bit contribution, and only the low 16 bits of the result are stored.

A useful description of the final result is unsigned addition modulo 65,536. This describes the stored bits, not necessarily the signed interpretation of the intermediate arithmetic.

The supplied representation interprets the captured global contribution as a signed 16-bit quantity before addition. Its range is therefore −32,768 through 32,767. Adding an increment of 0–127 gives an intermediate range of −32,768 through 32,894. This fits within a signed 32-bit integer.

Interpreting the captured low half as unsigned instead produces the same final 16-bit pattern. No saturation, separate carry output, or arithmetic-error indication is present. The captured global value’s upper 16 bits do not affect this result.

### Access sequence and preservation

After capturing the global value, the routine performs destination writes in this order:

1. Offset 0.
2. Offset 1.
3. Offset 2.
4. Offset 3.
5. Offset 4.
6. One 16-bit store beginning at offset 6.
7. Offset 8.

The first five and last writes are byte accesses. There are seven destination write operations covering eight byte positions.

No destination read is represented. Offset 5 is not initialized or otherwise modified by an explicit destination access. In ordinary storage without external interference, its previous value remains unchanged.

The sequence does not establish hardware atomicity, inter-thread visibility, or indivisibility of the 16-bit access.

## Side effects

The supplied body establishes:

- One 32-bit global read.
- Five input-derived byte writes.
- One global-derived 16-bit write.
- One final zero-byte write.

It contains no function calls, explicit global assignment, memory-management operation, or status return.

Destination aliasing can nevertheless make the writes affect globally accessible storage. In particular, overlapping the captured global location may change that location after its value has been read.

There is no pointer validation, bounds check, alignment check, explicit error report, or recovery branch. The exact response to an invalid memory access is not established. If execution fails after some writes, no rollback mechanism is shown; the evidence does not establish precisely which effects would remain visible.

## Dependencies

There are no called-function dependencies in the supplied body. No additional opaque function IDs are required.

The non-function dependencies are:

- The readable 32-bit global location.
- Caller-provided destination storage.
- Target support for the represented byte and 16-bit accesses.
- Target byte ordering, alignment, and memory-access rules.

No unknown callee is involved, so the contract does not depend on assumptions about a callee preserving globals or managing memory.

## Acceptance examples

These are conditional arithmetic expectations, not execution results. Unless otherwise stated, assume valid ordinary memory, no concurrent modification, and no overlap between the destination and global storage. A stated 16-bit result describes its numeric bit pattern without choosing byte order.

### Both low halves zero

For first input 0, second input 0, and captured global low half 0x1234:

| Location | Expected result |
|---|---|
| Offsets 0–2 | 0, 0, 0 |
| Offsets 3–4 | 1, 1 |
| Offset 5 | Previous value |
| Offsets 6–7 | 0x1234 |
| Offset 8 | 0 |

First input 0x00010000 and second input 0x00020000 produce the same result. Their upper halves do not prevent substitution.

### Only the first low half zero

For first input 0, second input 1, and captured global low half 0x1234:

- Offsets 0–4 become 2, 0, 0, 0, and 0.
- The 16-bit result is 0x1234.
- Offset 8 becomes zero.
- Offset 5 remains unchanged.

The nonzero second low half prevents substitution.

### Representative extraction

For first input 0x1234, second input 0xABCD, and captured global low half 0x1234:

- Offsets 0–4 become 26, 30, 21, 20, and 1.
- The first-input increment is 9.
- The 16-bit result is 0x123D.
- Offset 8 becomes zero; offset 5 remains unchanged.

If the target is little-endian, offsets 6 and 7 contain 0x3D and 0x12. If it is big-endian, they contain 0x12 and 0x3D.

### Boundary between first-input fields

With second input 1 and captured global low half zero:

- First input 0x0100 produces 0 at offset 3, 8 at offset 4, and a zero 16-bit result.
- First input 0x0200 produces 0 at offsets 3 and 4, and a 16-bit result of 1.

This distinguishes the highest bit carried by offset 4 from the lowest bit contributing to the arithmetic field.

### Maximum fields and modular wrap

For both input low halves equal to 0xFFFF and captured global low half 0xFFF0:

- Offsets 0–4 become 62, 63, 31, 31, and 15.
- The increment is 127.
- The 16-bit result is 0x006F.

The result does not clamp to 0xFFFF. Changing only the upper halves of either integer or of the captured global value leaves these destination values unchanged.

### Captured-value behavior under overlap

If a valid destination overlaps the global location, the arithmetic field must use the global value captured before the destination writes, rather than a value reconstructed from storage after offsets 0–4 have been written.

The final global storage contents depend on the precise overlap and target byte order. No single final global value follows without those details.

## Uncertainties

The following remain **documented_only**:

- **Domain meaning:** No application-specific interpretation of the inputs or fields is established.
- **Target representation:** The extraction contract follows the supplied 32-bit fixed-width representation. It is not a guarantee about unrestricted signed-shift expressions in another implementation language.
- **Byte ordering:** The physical order of the two bytes in the 16-bit field is unspecified by the evidence.
- **Alignment:** Exact alignment requirements and support for unaligned stores are unknown.
- **Global properties:** Initialization, volatility, device-memory behavior, mutability, and concurrent access rules are unknown.
- **Aliasing validity:** Capture-before-write is established, but the validity of particular overlapping regions depends on the memory environment.
- **Fault behavior:** Invalid-access outcomes, interrupted-write effects, and recovery mechanisms outside this body are unknown.
- **Concurrency:** No synchronization, atomicity, or intermediate-state visibility guarantee is established.
- **Platform and calling convention:** No complete ABI or broader caller contract is supplied.

Calling this routine a sparse field-packing operation is a behavioral description. Any more specific format interpretation would be inferred intent.

## Evidence

The supplied static body establishes the captured global read, low-16-bit substitution condition, field boundaries, access widths, destination offsets, represented write order, 16-bit result conversion, terminal zero byte, and absence of function calls or a return value.

The signed-intermediate range and modular-result equivalence follow from the represented widths and conversions. The acceptance examples are arithmetic deductions from those operations, not observed outputs.

No runtime tests have been performed. No independent caller evidence, target memory specification, or external format documentation was supplied.
