<!-- Trusted provenance: F-3ef7e8fe4c0c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3ef7e8fe4c0c

## Purpose

The routine describes an ordered sequence of memory mutations using input-derived addresses, dependency addresses, fixed storage, and an incoming stack-supplied pointer. Its two represented termination paths end at invalid-instruction boundaries. No ordinary return is shown.

No application-level purpose is established. The supplied evidence does not justify describing the routine as an initializer, decoder, allocator, or exception handler.

This contract distinguishes:

- **Established static evidence:** represented access widths, expressions, ordering, dependency-address uses, and control flow.
- **Conditional consequences:** results that follow under explicitly stated arithmetic and memory assumptions.
- **documented_only unknowns:** behavior that requires instruction semantics, complete storage declarations, platform information, or runtime observations.

Warnings about invalid instruction data and overlapping globals constrain interpretation. The represented sequence does not establish that every operation is executable or reachable.

## Interface

### Explicit inputs and result

| Input | Represented type | Observable use |
|---|---|---|
| First | Unsigned 32-bit | Address formation, byte-store data, and masked 32-bit data. |
| Second | Signed 32-bit | Unsigned subtraction and a 32-bit store. |
| Third | Uninterpreted 32-bit | No represented use. |
| Fourth | Signed 32-bit | Address addition and a shift-derived destination. |

The declared result is void. There is no represented normal-return path or returned status.

### Implicit inputs and storage roles

The routine also uses:

- An incoming pointer to 32-bit storage, represented at stack displacement `0x250`.
- The address of that stack slot, separately from the pointer stored there.
- Addresses associated with **F-e47d7b2e9c88** and **F-8afdaab91693**.
- A late-read 32-bit global.
- Current global contents used in composite assignments.
- Memory reached by indirect reads.

The following descriptive roles identify storage without importing application meaning:

- **Address-cache global:** the represented 32-bit location at `0x0897EA80`, assigned four times the incoming pointer’s numerical value.
- **Late source global:** the 32-bit global read at the start of the final phase.
- **High-halfword field:** the 16-bit location assigned the late source value’s upper half and read again near termination.

Other globals remain identified by their effects below. Distinct storage roles do not establish disjointness.

### Arithmetic and access preconditions

The representation uses 8-, 16-, and 32-bit accesses and suggests a 32-bit address model. Actual address width, alignment rules, permissions, and valid ranges are not independently established.

The second input undergoes unsigned 32-bit subtraction by `0xA8`. From that difference the routine derives:

- A **doubled-difference address**, obtained by multiplying by two.
- A **masked difference**, obtained by clearing bit 31.
- A **half-index**, obtained by logically shifting the masked difference right by one.

The unsigned difference and its doubling wrap modulo \(2^{32}\). The half-index lies between zero and `0x3FFFFFFF`.

Other expressions involve signed arithmetic, signed shifts, or a shift count of 32. Their machine-level results must not be inferred from ordinary unsigned arithmetic. In particular, a represented 32-bit shift by 32 is not established to produce zero, preserve its operand, or use a masked shift count.

No bounds, null, ownership, or alignment checks are represented. Every later effect depends on preceding operations completing.

## Behavior

### Initial mutations and content-derived destination

The routine forms a destination from the fourth input shifted left by two and then right by 32. This destination remains unresolved because the shift semantics are not supplied.

The initial writes occur in the following order:

| Order | Destination | Width and data |
|---:|---|---|
| 1 | Fourth input plus `0x0897E8B8` | Eight bits: low byte of the half-index. |
| 2 | Fixed global storage | Value `0xB8`; destination width is not independently supplied. |
| 3 | First input plus six | Eight bits: low byte of the first input. |
| 4 | Unresolved fourth-input-derived destination | 32 bits: second input’s representation. |
| 5 | Doubled-difference address plus four | 16 bits: `0xFFFF`. |
| 6 | Doubled-difference address plus six | 16 bits: `0xFFFF`. |
| 7 | Unresolved fourth-input-derived destination plus four | Eight bits: `0xB8`. |
| 8 | Doubled-difference address plus two | 16 bits: `0xFFFF`. |
| 9 | Another global | Low 16 bits of the doubled-difference address. |

A subsequent unsigned 16-bit read occurs at the doubled-difference address itself. The zero-extended result becomes the destination address of a 32-bit store whose data is the first input masked by `0x897E8B80`.

That destination is determined by memory contents, not merely by the doubled-difference address. In the represented 32-bit model, it lies between `0x00000000` and `0x0000FFFF`. Earlier writes may affect the read through aliasing.

### First dependency address and early termination

The address associated with **F-e47d7b2e9c88** is used without invoking that dependency.

Two quantities are derived from its signed 32-bit interpretation:

- A **dependency-derived base**, using a right shift by 32; its value is unresolved.
- A **sign halfword**, using a right shift by 31 followed by narrowing to 16 bits.

Under conventional arithmetic-right-shift semantics, the sign halfword is zero for a nonnegative address interpretation and `0xFFFF` for a negative one.

Before the sign test, the routine writes that halfword at the dependency-derived base plus four. If this write completes and the dependency address is negative as signed 32-bit, the routine reaches the first invalid-instruction boundary. Later phases are not represented on that branch.

On the nonnegative branch, the ordered effects are:

1. Write the sign halfword at the dependency-derived base plus `0x12`.
2. Write the dependency address as 32-bit data at that same dependency address.
3. Read a signed 16-bit sample at the dependency-derived base plus the half-index.
4. Repeat the halfword write at base plus `0x12`.
5. Repeat the self-addressing 32-bit write at the dependency address.
6. Sign-extend the retained sample, shift it right by 16, and write the 32-bit result at base plus eight.

Under conventional arithmetic-right-shift semantics, the last value is zero for a nonnegative sample and `0xFFFFFFFF` for a negative sample.

The repeated writes remain distinct effects. The intervening sample read and possible externally observable memory behavior prevent treating them as a single contractual write.

### Selector and incoming-pointer effects

A global is assigned zero. Bit 2 of the first dependency address then selects either `0x58` or `0x5C`.

The selector controls two 32-bit destinations:

| Dependency-address bit 2 | Selector | Destination receiving the dependency-derived base | Destination receiving the half-index |
|---|---:|---:|---:|
| Clear | `0x58` | `0x5C` | `0x170` |
| Set | `0x5C` | `0x60` | `0x180` |

Between those two stores, separate globals receive:

- The masked difference shifted right by 17, narrowed to 16 bits.
- The half-index’s low byte, zero-extended to 16 bits.

After the selector-derived half-index store, another global receives the full half-index.

The incoming pointer then supplies a sequence of overlapping writes and an indirect read:

1. Its starting address receives the selector as 16 bits.
2. The same starting address receives 32 bits formed by shifting the dependency-derived base left by eight.
3. The same shifted value is written 20 bytes beyond the pointer.
4. The starting address receives the pointer’s own numerical value as 32 bits.
5. A 32-bit value is read from twice the pointer’s numerical value.
6. The low 16 bits of that loaded value are written at the pointer’s starting address.
7. The retained full loaded value is used as both destination address and 32-bit data for a self-addressing store.

The narrowing in the sixth effect does not narrow the retained value used by the seventh.

Next, the routine writes a 32-bit sign-derived value at 32 times the **incoming stack slot’s address**, plus `0x18`. The data comes from the incoming pointer’s signed 32-bit interpretation shifted right by 31. Under conventional arithmetic-right-shift semantics, it is zero or `0xFFFFFFFF`.

This stack-slot-derived destination is distinct from expressions using the pointer’s numerical value.

A subsequent global assignment constructs a 24-bit numerical concatenation: a 16-bit value of five above a byte read from current global storage. The destination’s complete storage extent is not supplied. The byte is read at this point, rather than necessarily retained from an earlier assignment.

The incoming pointer’s numerical value is then written as 32-bit data at four times that value plus `0x18`.

### Final mutation phase

The late source global is read into a retained 32-bit value. The address associated with **F-8afdaab91693** is obtained without invoking that dependency.

The address-cache global is assigned four times the incoming pointer’s numerical value. Four times the second dependency address is also formed for subsequent use.

The next effects, in order, are:

1. Use the address-cache global’s contents as both the destination and data of a 32-bit self-addressing store.
2. Read the address-cache global again and write the half-index as 32 bits at its current value plus four.
3. Write four times the retained late source value as 32-bit data at that same source-derived address.
4. Write four times the second dependency address at the retained late source value plus its signed shift-derived expression: left shift by six, then right shift by 17.
5. Assign the retained late source value’s upper 16 bits to the high-halfword field.
6. Assign zero to another global.

The second effect represents a fresh global use, not an explicitly retained copy. The relative evaluation order of destination and data uses within the first effect is not established.

However, a fresh read does **not** establish that the preceding self-addressing store changes the address-cache global. Under ordinary byte-addressed memory and 32-bit wrapping address formation:

- The address-cache global occupies four bytes starting at the four-byte-aligned address `0x0897EA80`.
- Its assigned value, four times the incoming pointer, is also four-byte aligned.
- A four-byte store at that value either does not overlap the global or starts exactly at the global.
- In the exact-overlap case, the store writes the same 32-bit value already held there.

Thus that store alone does not change the address-cache global under this model, regardless of endianness. Device-specific write effects, concurrent mutation, or a different storage model would be additional assumptions, not consequences of ordinary overlap.

The third effect is independently source-derived. Its address must not be replaced with the pointer-derived address-cache value.

Next, a base equal to eight times the second dependency address receives a 32-bit zero, followed by another 32-bit zero at base plus four. A byte store at base plus `0x10` contains the low eight bits of base plus eight.

The final global assignments occur in this order:

1. Assign `0x897EA700` to fixed storage.
2. Assign `0x66` to another location.
3. Assign `0xEAEC` to another location.
4. Assign `0x1A` to another location.
5. Assign a fixed datum’s address to another global.
6. Read the high-halfword field at this point and combine it above the low halfword `0xEA70` to form a 32-bit composite assigned to overlapping global storage.
7. Assign `0x5C` to a final location.

The sixth effect uses the field’s current contents, not necessarily the retained late source value’s upper half. Intervening writes can matter where actual overlap exists.

Widths of all other symbol-based assignments are not supplied. The final represented action is an unconditional invalid-instruction boundary.

## Side effects

### Memory and ordering

Potentially affected storage includes caller-addressed memory, low-address memory, stack-related addresses, fixed globals, and locations derived from dependency addresses.

Ordering is established at the supplied representation’s level. No atomicity, synchronization, or hardware ordering guarantee is established.

The overlap warning prevents assuming that differently named globals are independent. It does not prove arbitrary overlap outcomes: actual locations, access widths, stored values, and alignment still constrain which changes are possible.

Endianness is unspecified. This contract gives numerical widths and address offsets rather than inventing a byte layout for incompletely described overlapping storage.

### Ownership and lifetime

No allocation, release, reference-count operation, ownership transfer, or lifetime extension is represented.

No evidence establishes exclusive ownership or permission to modify any destination. Dependency-address references are not calls and establish no allocation, release, or global-preservation contract.

### Errors and termination

No error result, rollback, cleanup, or recovery path is represented.

An invalid address, protection violation, alignment fault, or unresolved arithmetic operation may prevent reaching either explicit invalid-instruction boundary. The resulting platform signal or exception is unknown.

Completed writes are not represented as being undone.

## Dependencies

| Opaque ID | Established relationship | Unknowns |
|---|---|---|
| **F-e47d7b2e9c88** | Its address supplies sign tests, shifts, selector bits, stored data, and self-addressing destinations. | Relocation, executable validity, writability, ownership, and any implementation behavior. |
| **F-8afdaab91693** | Its address supplies scaled data and destination addresses. | Relocation, executable validity, writability, ownership, and any implementation behavior. |

Neither dependency is represented as invoked. The invalid-instruction boundaries do not establish a callable error-handling dependency.

## Acceptance examples

These are conditional static consequences, not runtime test results. Each assumes that preceding operations complete.

1. **Second input equals `0xA8`.**  
   The unsigned difference, doubled-difference address, masked difference, and half-index are zero. The initial halfword stores target `0x4`, `0x6`, and `0x2`, in that order. The later unsigned halfword read targets zero. Accessibility is not implied.

2. **Second input has bit pattern `0x000000A7`.**  
   Unsigned subtraction produces `0xFFFFFFFF`; doubling produces `0xFFFFFFFE`. The masked difference is `0x7FFFFFFF`, the half-index is `0x3FFFFFFF`, its low byte is `0xFF`, and the masked difference shifted right by 17 is `0x3FFF`.

3. **Initial content-derived address is `0x1234`.**  
   If the unsigned halfword read yields `0x1234`, the following 32-bit store targets `0x00001234` and contains the first input masked by `0x897E8B80`.

4. **First dependency address has a negative signed interpretation.**  
   The halfword write at the unresolved dependency-derived base plus four precedes termination. If it completes, the negative branch reaches the first invalid-instruction boundary without entering the subsequent phases.

5. **Address-cache self-store exactly overlaps the global.**  
   Assume ordinary byte-addressed memory without external mutation or device-specific write effects, and an incoming pointer value of `0x0225FAA0`. Multiplication by four gives `0x0897EA80`, the address-cache global’s location. The self-addressing store writes `0x0897EA80` at `0x0897EA80`, leaving the global unchanged. The following reread therefore directs the half-index store to `0x0897EA84`. This result does not depend on endianness.

6. **Pointer-derived and late-source-derived destinations differ.**  
   If the incoming pointer is `0x400`, its four-times value is `0x1000`. If the retained late source value is `0x600`, its independent self-addressing store writes `0x1800` at `0x1800`, assuming the relevant accesses and arithmetic complete. It does not target `0x1000`.

7. **The high-halfword field changes before final composition.**  
   If it initially receives `0x1234` but actually contains `0xABCD` when the final composite reads it, the resulting composite is `0xABCDEA70`. The contract does not assert that any particular unspecified overlap causes that change.

8. **Only the third input changes.**  
   With identical remaining inputs, memory, implicit state, and execution semantics, changing only the third input produces no represented behavioral difference.

## Uncertainties

The following remain **documented_only**:

- Actual instruction decoding and control-flow validity.
- Results of shifts by 32.
- Signed overflow, signed left shifts, and platform-specific signed right shifts.
- Actual address width and signed address-arithmetic semantics.
- Complete widths and overlap relationships of incompletely typed globals.
- Endianness, alignment requirements, permissions, and device-memory behavior.
- Incoming stack pointer provenance and calling convention.
- Machine-level reloads and evaluation order not fixed by the representation.
- Dependency-address relocation and writability.
- Whether execution can reach the final boundary without earlier interruption.
- Whether invalid-instruction boundaries represent intentional traps, embedded data, or decoding failure.

No domain-level intent is inferred from storage labels. External mutation and device-specific write transformations are not established facts.

## Evidence

The contract relies on the supplied static representation’s explicit access widths, arithmetic expressions, ordering, conditional branch, dependency-address references, global reads, and termination boundaries.

The address-cache analysis accounts for its represented location, four-byte extent, pointer-times-four derivation, and self-stored value. A reread is preserved as a represented operation without inventing an impossible ordinary-memory transition from `0x00001000` to `0x00002000`.

The independent late-source-derived destination and the late current-storage read of the high-halfword field remain distinct from earlier retained values.

Warnings concerning invalid instruction data and overlapping globals materially limit interpretation. Complete storage declarations, instruction bytes, dependency implementations, and a platform specification were not supplied. **No runtime tests have been performed.**
