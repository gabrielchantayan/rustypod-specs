<!-- Trusted provenance: F-be7a1ec3891e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-be7a1ec3891e

## Purpose

F-be7a1ec3891e specifies an operation whose visible behavior is an ordered sequence of memory accesses, address calculations, fixed-storage assignments, and a signed branch. Both represented paths end at invalid-instruction boundaries. No ordinary return is shown.

**Established static evidence:** The supplied representation includes explicit byte, 16-bit, and 32-bit indirect accesses; assignments to fixed-storage objects whose declarations are absent; repeated references to an incoming stack-slot pointer; and references to two function-associated identifiers without explicit invocation.

**Inferred intent:** No application-level purpose is established. The values written resemble address fragments and instruction-related constants, but initialization, patching, or instruction construction are not contractual purposes.

This specification describes the supplied static representation. It does not establish that the entire sequence is reachable, correctly decoded, or executable under a particular machine model.

## Interface

### Explicit arguments

| Position | Represented type | Observable use |
|---|---|---|
| First | Signed 32-bit integer | Converted through unsigned arithmetic into an adjusted value, a doubled address, and masked fragments. |
| Second | Uninterpreted 32-bit value | Written through an initially computed destination. |
| Third | Uninterpreted 32-bit value | No visible use. |
| Fourth | Signed 32-bit integer | Supplies the initial byte-write address and another destination expression. |

No argument is established as a validated buffer, length, handle, or ownership-bearing reference.

### Additional entry state

The operation also depends on:

- An incoming unsigned 32-bit register value, called **register value** below.
- A pointer represented in an incoming stack slot.
- The address of that stack slot, called **slot address**.
- Separate evaluations of identifiers associated with F-e47d7b2e9c88 and F-8afdaab91693.
- Values read from fixed storage and computed addresses.

**Pointer stability is unresolved.** The representation repeatedly refers to the incoming stack-slot pointer rather than explicitly capturing it once in a separate value. Computed writes may overlap the slot. Consequently, this contract uses **pointer-use value** to mean the pointer represented at a particular use, without asserting that every use equals the entry value or that each use causes a physical reload.

If the pointer remains stable, its repeated uses have the familiar single-pointer interpretation described below. Under possible slot overlap, subsequent pointer values and derived addresses remain **documented_only**. Uses within one expression also have no independently established capture or reload policy.

### Results, failures, and ownership

No result value, success status, error return, allocation, release, or ownership transfer is visible.

The signed-negative branch reaches the earlier invalid-instruction boundary. The continuing path reaches a second such boundary. Neither marker specifies a processor exception, signal, diagnostic, or recovery mechanism.

Memory accessibility, address validity, and applicable alignment requirements are unchecked execution preconditions. A failing access may prevent later effects; no rollback is shown. Properties of the dependency-associated identifiers do not establish memory ownership or preservation of global state.

## Behavior

### Arithmetic and address interpretation

Define:

- **Adjusted input:** first argument minus `0xA8`, using unsigned 32-bit subtraction.
- **Masked input:** adjusted input with bit 31 cleared.
- **Half-index:** masked input logically shifted right by one.
- **Doubled base:** adjusted input multiplied by two using unsigned 32-bit arithmetic, then interpreted as a pointer to 16-bit entries.
- **Primary destination:** fourth argument shifted left by two, then signed-right-shifted by 32, then converted to a pointer.

The unsigned subtraction and multiplication wrap modulo \(2^{32}\). Half-index lies between zero and `0x3FFFFFFF`.

Primary destination is unresolved: the represented signed operand is 32 bits wide, yet the right-shift count is 32. The preceding signed left shift also has interpretation constraints. Neither a zero result nor a machine-specific masked shift count is established.

Other signed arithmetic, integer-to-pointer conversions, and pointer arithmetic retain their unresolved target semantics unless specified otherwise below. In particular, signed multiplication of a pointer’s converted 32-bit representation must not silently become unsigned modular arithmetic.

### Fixed-storage identities

These labels identify storage roles, not recovered declarations. Except where an indirect access explicitly supplies a width, a converted value does not establish the destination’s write extent.

| Identity | Address or role | Established information |
|---|---|---|
| S01 | `0x0897E8C0` | Assigned `0xB8`; destination width unknown. |
| S02 | `0x0897EA28` | Assigned doubled base’s low-16-bit fragment; destination width unknown. |
| S03 | First cleared region on the continuing path | Assigned zero; width unknown. |
| S04 | Masked-input-fragment region | Assigned a value converted to unsigned 16 bits; width unknown. |
| S05 | Half-index-fragment and later concatenation region | Receives a 16-bit-converted value, then a 24-bit composition; destination width unknown. |
| S05-byte | Smaller byte view at S05’s address | Read for the 24-bit composition. |
| S06 | Full-half-index region | Assigned half-index; width unknown. |
| S07 | `0x0897EA50` | Read as signed 32 bits. |
| S08 | `0x0897EA80` | Receives a signed 32-bit publication and is subsequently reread. |
| S09 | Upper-halfword-fragment region | Assigned a value converted to 16 bits; destination width unknown. Later supplies the upper 16-bit concatenation operand. |
| S10 | Second late-cleared region | Assigned zero; width unknown. |
| S11 | `0x00000086` | Assigned `0x897EA700`; width unknown. |
| S12 | `0xEA700000` | Assigned `0x66`; width unknown. |
| S13 | `0x0897EB26` | Assigned `0xEAEC`; width unknown. |
| S14 | `0x12FD4C34` | Assigned `0x1A`; width unknown. |
| S15 | `0x897EA65C` | Assigned the address of the smaller object at that same address; width unknown. |
| S16 | Final concatenation destination | Assigned a 32-bit composition; destination width unknown. |
| S17 | `0x12FD4CB8` | Assigned `0x5C`; width unknown. |

S05 and S05-byte have an established overlapping relationship. A complete overlap map for the remaining objects is unavailable. In particular, the relationship between S09 and S16 is not established.

### Initial access sequence

After calculating the initial quantities, the represented operations occur in this order:

1. Write half-index’s low byte at fourth argument plus `0x0897E8B8`.
2. Assign `0xB8` to S01.
3. Write the register value’s low byte at register value plus six. This addition is unsigned 32-bit arithmetic.
4. Write the second argument as 32 bits at primary destination.
5. Write `0xFFFF` as 16 bits at doubled base plus four bytes.
6. Write `0xFFFF` as 16 bits at doubled base plus six bytes.
7. Write `0xB8` as one byte at primary destination plus four bytes.
8. Write `0xFFFF` as 16 bits at doubled base plus two bytes.
9. Assign doubled base’s low-16-bit fragment to S02.
10. Read unsigned 16 bits at doubled base. Zero-extend that read to a numeric address and write the register value masked with `0x897E8B80` there as 32 bits.

The final read accesses the base entry, not one of the three neighboring entries explicitly initialized above. Earlier writes can nevertheless affect it through aliasing.

### First dependency evaluations and branch

The identifier associated with F-e47d7b2e9c88 is evaluated separately three times:

1. Save an unsigned 32-bit value as **branch value**.
2. Convert another evaluation to signed 32 bits and right-shift by 32 to obtain **companion base**.
3. Convert another evaluation to signed 32 bits, right-shift by 31, and truncate to signed 16 bits to obtain **sign fragment**.

The three evaluations are not established as interchangeable. Companion base retains the unresolved shift-by-32 semantics.

The operation then writes sign fragment as 16 bits at companion base plus four bytes and tests branch value interpreted as signed 32 bits.

If negative, it reaches the earlier invalid-instruction boundary. No subsequent operation is represented on that path.

Otherwise:

1. Write sign fragment as 16 bits at companion base plus `0x12`.
2. Write branch value as 32 bits at the address numerically equal to branch value.
3. Read signed 16 bits at the address obtained by unsigned 32-bit addition of companion base’s converted representation and half-index.
4. Repeat the 16-bit write at companion base plus `0x12`.
5. Repeat the 32-bit self-value write through branch value.
6. Sign-extend the saved 16-bit read to 32 bits, right-shift it by 16, and write the result as 32 bits at companion base plus eight.

For step 3, half-index’s unsigned type converts companion base to unsigned 32 bits. The addition wraps modulo \(2^{32}\) **before** pointer conversion. Half-index is a byte displacement here, not a count of 16-bit entries.

Under arithmetic signed right shift, step 6 writes zero for a nonnegative saved read and `0xFFFFFFFF` for a negative saved read. The repeated writes remain distinct; the intervening read and possible overlap matter.

### Low-address and stack-pointer-dependent operations

The continuing path next:

1. Assigns zero to S03.
2. Forms **selected offset** as `0x58` plus branch value’s bit two, retaining that bit’s numeric value of zero or four.
3. Writes companion base as 32 bits at selected offset plus four: address `0x5C` or `0x60`.
4. Assigns the unsigned-16-bit conversion of masked input shifted right by 17 to S04.
5. Assigns half-index’s low byte, zero-extended to 16 bits, to S05.
6. Writes half-index as 32 bits at four times selected offset plus `0x10`: address `0x170` or `0x180`.
7. Assigns half-index to S06.

The following operations use the incoming stack-slot pointer repeatedly. Each pointer reference is subject to the stability limitation in the Interface section:

1. Write selected offset’s low 16 bits through the pointer-use value.
2. Write companion base shifted left by eight as 32 bits through the pointer-use value.
3. Write the same shift expression as 32 bits at pointer-use value plus 20 bytes.
4. Write a pointer-use value’s signed 32-bit representation through a pointer-use destination.
5. Read 32 bits at twice the signed 32-bit representation of a pointer-use value; save this as **loaded value**.
6. Write loaded value’s low 16 bits through the pointer-use value.
7. Write loaded value as 32 bits at the address numerically equal to loaded value.
8. At 32 times the signed 32-bit representation of slot address plus `0x18`, write a pointer-use value’s signed 32-bit representation right-shifted by 31.
9. Read S05-byte, place the 16-bit value five above that byte, and assign the resulting 24-bit composition to S05.
10. At four times a pointer-use value’s signed 32-bit representation plus `0x18`, write a pointer-use value using a represented 32-bit pointer store.

Step 8 derives its destination from the **slot address**, not the pointer held there. Step 10 derives its destination from a pointer-use value. Neither distinction resolves whether earlier writes alter later pointer uses.

Step 9’s composition is numerically `0x000500` plus the byte read. That byte need not equal the originally assigned half-index fragment: intervening writes may overlap S05, and byte order affects how a multibyte assignment relates to its byte view.

### Late reads and publication

The next operations are:

1. Read S07 as signed 32 bits and save **late value**.
2. Evaluate the identifier associated with F-8afdaab91693 and save its represented signed 32-bit value as **saved dependency base**.
3. Assign four times a pointer-use value’s signed 32-bit representation to S08.
4. Evaluate the F-8afdaab91693-associated identifier again and shift that evaluation left by two, saving **shifted dependency value**.
5. Read S08 for an indirect store’s destination and read S08 for its 32-bit value, then perform that store.
6. Read S08 again, add four, and write half-index there as 32 bits.

The two operand evaluations in step 5 have no established relative order; both precede the store. Step 6 uses a fresh represented S08 access, not a substituted copy of the originally published expression.

For example, a partially overlapping step-5 store can change S08 and thereby change step 6’s destination. No alignment or disjointness guarantee excludes that case. The intervening identifier evaluation likewise has no established global-preservation guarantee, although no explicit call is shown.

### Remaining writes and termination

Using the saved late and dependency-derived values:

1. Write four times late value as 32 bits at the address equal to that product.
2. Shift late value left by six, signed-right-shift that result by 17, and add late value. At the resulting address, write shifted dependency value as 32 bits.
3. Convert late value to unsigned 32 bits, logically shift right by 16, convert the result to a 16-bit value, and assign it to S09. **The destination write width is unknown.**
4. Assign zero to S10.
5. Form **final base** as saved dependency base multiplied by eight.
6. Clear 32 bits at final base.
7. Clear 32 bits at final base plus four bytes.
8. At final base plus 16 bytes, write one byte formed from the byte-converted base plus eight, retaining the resulting byte representation.
9. Assign `0x897EA700` to S11.
10. Assign `0x66` to S12.
11. Assign `0xEAEC` to S13.
12. Assign `0x1A` to S14.
13. Assign the smaller colocated object’s address to S15.
14. Read S09 as the upper 16-bit operand of a concatenation with lower halfword `0xEA70`; assign the 32-bit composition to S16.
15. Assign `0x5C` to S17.
16. Reach the final invalid-instruction boundary.

The S09 concatenation operand is obtained at step 14. It is not necessarily the halfword derived at step 3: intervening writes may overlap its storage.

## Side effects

The represented writes are nontransactional. No bounds checking, null checking, rollback, synchronization, or error-reporting protocol appears.

Observable distinctions that must be retained include:

- Separate repeated stores around the signed 16-bit read.
- Partial-width writes through pointer-use values.
- Possible modification of the incoming pointer’s stack slot.
- Reads following potentially overlapping writes.
- S08 publication followed by separate represented accesses.
- Fresh S05-byte and S09 reads for concatenations.
- Separate dependency-associated identifier evaluations.

No volatility, atomicity, memory barriers, physical-load counts, or inter-thread ordering guarantees are established. The listed sequence describes represented evaluations, not a complete hardware memory model.

## Dependencies

| Opaque function ID | Established relationship | Unresolved properties |
|---|---|---|
| F-e47d7b2e9c88 | Three identifier evaluations supply branch value, companion base, and sign fragment. | Invocation semantics, stability, address validity, side effects, global preservation, and ownership. |
| F-8afdaab91693 | Separate evaluations supply saved dependency base and shifted dependency value, on opposite sides of S08 publication. | Invocation semantics, equivalence, side effects, global preservation, allocation, release, and ownership. |

No explicit invocation of either dependency appears. Function association alone does not justify describing these evaluations as calls or assuming callee behavior.

## Acceptance examples

These are conditional static examples, not executed tests. Each requires preceding accesses and unresolved arithmetic to admit the stated path.

1. **Zero adjusted input.** First argument `0xA8` produces adjusted input, masked input, half-index, and doubled-base numeric value zero. The first byte write is zero. Address zero is not thereby valid.

2. **Unsigned underflow.** First argument `0xA7` produces adjusted input `0xFFFFFFFF`, masked input `0x7FFFFFFF`, half-index `0x3FFFFFFF`, and doubled-base numeric value `0xFFFFFFFE`.

3. **Early termination.** A branch value negative under signed 32-bit interpretation selects the earlier boundary after the companion-base-plus-four store. The other dependency evaluations need not equal branch value.

4. **Wrapping read address.** If companion base has numeric representation `0xFFFFFFF0` and half-index is `0x20`, the signed 16-bit read’s address expression yields `0x00000010` before pointer conversion. It does not yield an unbounded sum.

5. **Offset selection.** A clear bit two selects low addresses `0x5C` and `0x170`; a set bit two selects `0x60` and `0x180`.

6. **Signed readback.** Under arithmetic right shift, a saved 16-bit value `0x8000` produces the later 32-bit write value `0xFFFFFFFF`; `0x7FFF` produces zero.

7. **Pointer-slot overlap.** If a computed write overlaps the incoming pointer’s stack slot, later pointer-derived destinations cannot be required to use the entry pointer. This representation alone also cannot require a particular physical reload or changed pointer value.

8. **S08 overlap.** If its first indirect store changes bytes of S08, the following address calculation uses the resulting S08 access, not an assumed preserved publication value.

9. **S09 conversion and concatenation.** Late value `0x12345678` supplies the converted 16-bit value `0x1234` for assignment to S09, without establishing a two-byte store. If S09 supplies `0xABCD` at the later concatenation, the composition is `0xABCDEA70`.

10. **Unused argument.** Changing only the third argument changes none of the visible explicit operations, with all other entry state held equal.

## Uncertainties

The following remain **documented_only**:

- Actual instruction decoding, reachability, and invalid-instruction outcomes.
- Semantics behind both represented shifts by 32.
- Other signed shifts, signed overflow, pointer conversion, and pointer arithmetic.
- Pointer-use stability under possible stack-slot overlap, including capture versus reload behavior.
- Target byte order, alignment, address range, and truncation rules.
- Missing fixed-storage declarations and destination widths, including S09’s.
- The complete storage-overlap map.
- Provenance and validity of the incoming register value and stack-slot contents.
- Stability and interpretation of dependency-associated identifier evaluations.
- Memory kind, fault handling, concurrency, and physical access counts.
- Higher-level application purpose.

These unknowns must not be resolved by convenient implementation defaults.

## Evidence

This contract relies only on the supplied static representation and opaque dependency mapping. The supplied representation carries warnings about invalid instruction data, truncated control flow, and overlapping storage symbols; those warnings identify limitations rather than independently proving machine behavior.

Established evidence includes explicit typed indirect accesses, unsigned input arithmetic, unsigned wrapping addition for the companion-base-plus-half-index read address, repeated stack-pointer references, the separate use of the stack-slot address, S08 rereferences, one signed branch, and no visible normal return.

For S09, the evidence establishes a **16-bit value conversion assigned to an object with an unavailable declaration**, followed later by use as a 16-bit concatenation operand. It does not establish an explicit 16-bit store to S09.

Storage labels and descriptive quantities are organizational devices, not recovered domain concepts. No runtime tests, execution traces, independent implementation inspection, or instruction-byte analysis have been performed.
