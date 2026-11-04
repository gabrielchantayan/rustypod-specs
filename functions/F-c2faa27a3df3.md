<!-- Trusted provenance: F-c2faa27a3df3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c2faa27a3df3

## Purpose

F-c2faa27a3df3 modifies memory selected by an explicit address argument, an incoming register, and several shared storage locations. Its observable behavior is an ordered memory-update prefix. Earlier writes can change values that determine later destinations or payloads.

The supplied static representation establishes five indirect writes with specified widths and six assignments to shared storage whose complete widths are not supplied. It does not establish a normal return. Analysis stops at a boundary identified as invalid instruction data; behavior at or beyond that boundary remains unresolved.

This contract describes those memory effects without assigning them an application-level purpose. In particular, storing an address does not establish that the address is subsequently invoked, that a processor configuration is installed, or that the stored value identifies an owned object.

Evidence classifications used below:

- **Established:** directly supported by the supplied static representation.
- **Conditional deduction:** follows only under the stated representation, layout, or environmental assumptions.
- **documented_only:** relevant behavior not resolved by the available evidence.

No runtime tests have been performed.

## Interface

### Explicit inputs

The represented interface contains three arguments:

| Role | Width or representation | Contract |
|---|---|---|
| Payload | Two bytes | An opaque bit pattern written to the initial destination. No field interpretation or validation is visible. |
| Unused input | Four bytes | Does not participate in the represented execution prefix. |
| Initial address base | Recovered integer representation | Determines the initial destination using a displacement of 42 bytes. It is not accompanied by an object extent, access permission, or ownership guarantee. |

The initial displacement is byte-based in the recovered expression: arithmetic precedes conversion to the two-byte destination type. The representation does not independently establish an ABI, integer overflow behavior, or the conversion from an integer result to an accessible address.

### Ambient inputs and shared storage

An unsigned register value is consumed without a visible initialization. Its high-order bits select a one-byte clearing destination. Its origin and calling-convention status are **documented_only**; it is an input to the represented prefix, not an established fourth argument.

The following neutral labels describe shared storage roles. They are not claims that the underlying regions are disjoint.

| Label | Relevant property |
|---|---|
| Low-base cell | Located at represented address 0x00000040. Its contents contribute to a clearing destination. |
| Mutable-base cell | Located at represented address 0x0004B000. Read at two distinct stages. |
| Captured-base slot | Receives an earlier value from the mutable-base cell and is used again later. |
| First zero slot; second zero slot | Assigned zero in that order. |
| Constant slot | Assigned the operand 0xFFFFFFA0. |
| First address slot; second address slot | Assigned the address value 0x0897FAC4 at different stages. |

The evidence warns that some shared symbols overlap smaller symbols at the same address. It does not supply a complete layout, destination widths, or a usable alias map.

The address of the low-base cell and its stored value are separate facts. A cell located at 0x00000040 does not imply that the later clearing destination is 0x0000004C.

### Results and preconditions

No result value or normal-return contract is established. The observable results are memory accesses, completed changes, and any environment-specific consequences of those accesses.

Reaching the endpoint requires all preceding reads and writes to be permitted and meaningful in the execution environment. The routine does not visibly check mapping, alignment, object extent, lifetime, or permissions. These are access requirements, not validated guarantees.

## Behavior

### Initial memory effects

The first observable update writes the two-byte payload beginning 42 bytes beyond the initial address base. It is followed by a one-byte write of zero to the register-selected address.

Only after these two writes is the low-base cell consulted. Its then-current contents, combined with a displacement of 12, determine a destination receiving a four-byte zero.

This ordering matters even without concurrency. If either initial write overlaps the low-base cell, the later destination depends on the modified cell, not its entry value. Similarly, the four-byte clearing operation can change storage used by subsequent stages.

No destination read for merging is represented for these writes. Their logical widths do not establish instruction count, bus transaction count, or atomicity.

### Register-selected destination

The selector calculation first shifts the unsigned register value right by 26 positions, then subtracts 208.

Under the recovered 32-bit unsigned interpretation, the extracted quantity ranges from zero through 63. Unsigned subtraction therefore selects an address in the inclusive range **0xFFFFFF30–0xFFFFFF6F**, not the range zero through 63. Only the highest six register bits influence that choice; the lower 26 bits are discarded.

This is a conditional arithmetic result under the 32-bit unsigned model. Independent machine-width and address-conversion evidence is absent. The selected range is not established as mapped, writable, or ordinary memory.

### Shared-state updates and repeated base access

After the low-base-selected clearing operation, the mutable-base cell is read and its value assigned to the captured-base slot. This transfers a value; it does not copy an object from an address represented by that value.

The first zero slot is then assigned zero, followed by the second zero slot, followed by the constant slot. Only after those assignments is the mutable-base cell consulted again. This second consultation determines the destination receiving a four-byte value of **1200**, using a displacement of 16.

The two mutable-base consultations are semantically distinct. Intervening overlapping assignments, external changes, or read-sensitive memory behavior can produce different values. Substituting the first value for the second would discard an established dependency on execution-time state.

The global assignments have known operands but incompletely specified destinations:

- Zero is the assigned operand for each zero slot.
- The constant operand has the represented value 0xFFFFFFA0.
- Destination widths, conversions, and affected bytes depend on declarations not supplied.

If preserved as a 32-bit pattern, 0xFFFFFFA0 is exact. Interpreted as a signed 32-bit two’s-complement value, that pattern denotes −96. Neither interpretation establishes the constant’s purpose or the slot’s actual type.

### Final captured-state-dependent effects

The write of 1200 precedes assignment of 0x0897FAC4 to the first address slot. The captured-base slot is used only afterward for a final indirect update: its contents contribute both to a destination displaced by 20 and to a payload obtained by right-shifting by 29 positions. The indirect destination receives four bytes under the recovered integer-width model.

The captured-base slot need not still contain its originally assigned value. Intervening writes may overlap it, including the write of 1200 or the first address-slot assignment.

The representation contains separate uses of this slot for destination selection and payload calculation. It does not resolve their physical read count or their relative evaluation order. A contract requiring both uses to observe one identical sample would therefore be too strong when the storage can change or has read side effects.

For a 32-bit payload operand:

- A logical right shift by 29 yields a value from zero through seven.
- A sign-extending arithmetic right shift by 29 yields a value from −4 through three.

The available shared-storage declarations do not establish which interpretation applies. Conversion of the result to the four-byte destination also depends on the recovered type model.

After this indirect update, the second address slot is assigned 0x0897FAC4. That is the last represented memory update.

Neither address-slot assignment reads through 0x0897FAC4 or establishes an invocation. Storing the same address operand twice also does not guarantee equal final slot contents: intervening overlap can alter the first slot, and the last assignment can alter other results.

### Arithmetic, footprints, and retention

The five indirect writes have logical widths of two bytes, one byte, four bytes, four bytes, and four bytes, respectively. The six shared-storage assignments must not automatically be treated as six four-byte stores.

The displacements associated with shared-base values are 12, 16, and 20. Treating them as byte offsets follows an integer-base interpretation. Missing global declarations prevent independent confirmation that no pointer-based scaling is involved.

The selector’s unsigned wraparound deduction must not be generalized to other additions. Overflow and address conversion for the initial address and shared-base calculations remain **documented_only**.

A completed write establishes an effect at that stage, not its survival to the endpoint. Subsequent overlapping writes may replace all or part of it. Byte order determines the placement of multibyte patterns and is not supplied.

### Errors and endpoint

No explicit validation failure, error result, recovery branch, or rollback is represented.

If an access faults, later operations need not occur. No visible mechanism restores earlier completed effects. Whether a failing access can itself produce partial changes depends on the target and remains unresolved.

The final boundary is marked as invalid instruction data. This limits the established control-flow description. It does not independently prove a particular hardware exception, handler invocation, process termination, or nonreturning instruction. Normal return is likewise not established.

## Side effects

### Aliasing and mutable observations

The contract requires preservation of the represented ordering where writes can influence later reads. Relevant conditional cases include:

- An initial write alters the low-base cell before its consultation.
- The low-base-selected clear alters the mutable-base cell before its first consultation.
- An intervening shared assignment alters the mutable-base cell before its second consultation.
- An intermediate write alters the captured-base slot before its final uses.
- The final indirect update changes a previously assigned shared slot.
- The last address-slot assignment replaces part of an earlier result.

These are possible consequences of overlap, not an established overlap map.

For ordinary memory without concurrent changes, the later observations follow the changes actually made to overlapping bytes. For device-backed or concurrently modified storage, additional behavior depends on access semantics not provided here.

### External visibility and interruption

There is no visible transaction boundary, lock, or memory barrier. The described order is the represented execution dependency order, not a guarantee of cross-processor visibility or device transaction ordering.

Whole-routine atomicity and individual-access atomicity are not established. Interruption or failure can leave completed earlier updates without the later ones.

Reads may have effects if the addressed storage is read-sensitive. Writes may trigger effects beyond retained byte changes if the destination is device-backed. The evidence does not distinguish ordinary memory from such storage.

### Ownership

No allocation, deallocation, resource acquisition, or ownership transfer is visible.

Address arithmetic and address storage do not establish authority to access, retain, execute, or release the referenced region. No unresolved endpoint or environmental behavior may be assumed to preserve shared storage or manage memory on the routine’s behalf.

## Dependencies

No separately identified callable dependency with an opaque function ID is established by the supplied representation. The invalid-data endpoint is not sufficient evidence for an ordinary callee contract.

The behavior depends on the following environmental provisions:

- An incoming unsigned register value.
- Shared-storage types, widths, layout, and access semantics.
- Integer widths, shift rules, and address conversions.
- Address mapping, alignment, permissions, and byte order.
- Any read or write behavior associated with device-backed locations.
- Fault handling and execution behavior at the unresolved endpoint.

These dependencies carry no implied preservation, recovery, ownership, or release guarantees.

## Acceptance examples

These examples are conditional deductions from static evidence, not runtime observations.

### Two-byte payload and byte order

Assume the initial base is 0x00001000, address addition is valid, and the destination is accessible. For payload 0xABCD, the first write affects exactly the two-byte logical footprint beginning at 0x0000102A.

On a little-endian target, the bytes are 0xCD and 0xAB. On a big-endian target, they are 0xAB and 0xCD. Neither case permits treating the operation as a four-byte replacement. Final retention additionally requires no later overlapping write.

### Selector boundaries

Under 32-bit unsigned arithmetic:

| Incoming register value | Selected byte address |
|---|---|
| 0x00000000 | 0xFFFFFF30 |
| 0xFC000000 | 0xFFFFFF6F |
| 0xFFFFFFFF | 0xFFFFFF6F |

If the selected location accepts the access, its one-byte destination receives zero. The last two inputs select the same address because their high six bits are identical.

### Stable shared state

Assume all accesses succeed, relevant storage does not overlap, shared-base displacements are byte-based, and both mutable-base consultations return 0x00002000. Also assume a 32-bit captured-base slot preserves that value until its final uses.

The captured-base assignment receives 0x00002000. The four-byte value 1200 is written at 0x00002010. The later captured-state-dependent destination is 0x00002014 and receives zero under either logical or arithmetic 32-bit right-shift semantics.

This example does not establish that the routine returns.

### Changed second observation

Assume the mutable-base cell returns 0x00002000 on its first consultation and 0x00003000 on its second. With byte-based displacement, the captured assignment uses the former value, while the write of 1200 targets 0x00003010.

Using 0x00002010 for that write would not satisfy the represented behavior.

### Signedness remains observable

Assume the payload-side use of the captured-base slot observes the stable 32-bit pattern 0xE0000000.

A logical shift by 29 produces seven. A sign-extending arithmetic shift produces −1, represented as 0xFFFFFFFF in 32-bit two’s complement. The available evidence does not select between them. This example establishes neither a valid destination nor successful execution of the write.

### Failure after completed writes

Assume the initial payload write and selector-derived byte clear complete, but the subsequent low-base-cell read faults.

Those earlier effects precede the fault in the represented execution. No local rollback is established. Exception delivery, possible resumption, and any later execution remain **documented_only**.

## Uncertainties

The following remain **documented_only**:

- Shared-storage widths, signedness, conversions, volatility, and complete overlap layout.
- Independent ABI confirmation of integer, register, and pointer representations.
- Origin and preservation requirements of the incoming register.
- Overflow and address conversion outside the conditional unsigned selector calculation.
- Scaling of shared-base displacements without the missing declarations.
- Physical read count and relative evaluation order within the final captured-state-dependent update.
- Endianness, alignment, mapping, permissions, and access atomicity.
- Ordinary-memory versus device-memory behavior.
- Representation and lifetime of the stored address and whether it identifies executable storage.
- Fault delivery, partial failing accesses, interruption, and recovery.
- Runtime control flow at and beyond the invalid-data endpoint.

An inferred application purpose would not resolve any of these uncertainties.

## Evidence

The supplied static representation supports the explicit argument widths, the five indirect write footprints under its recovered type model, six shared-storage assignments, their operands, two distinct mutable-base consultations, and the ordering dependencies described above. It also identifies overlapping shared symbols and a control-flow endpoint associated with invalid instruction data.

The representation does not provide complete shared-storage declarations, an independent ABI, a memory map, or an execution trace. The earlier draft and review feedback are not independent behavioral evidence.

No runtime tests have been performed. All acceptance examples are conditional static deductions; none asserts observed execution or successful completion.
