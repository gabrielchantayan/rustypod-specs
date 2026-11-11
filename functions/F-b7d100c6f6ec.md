<!-- Trusted provenance: F-b7d100c6f6ec; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b7d100c6f6ec

## Purpose

F-b7d100c6f6ec repeatedly invokes indirect dependencies and performs ordered byte, halfword, and word transfers through externally supplied and computed addresses.

The supplied static representation shows unconditional repetition, not a terminating operation. Each iteration contains a software breakpoint. Execution after that breakpoint depends on the execution environment.

No application-level purpose is established. Initialization, device control, scheduling, and record processing are possible interpretations, not contractual facts.

This contract separates visible operations from unknown dependency behavior. It does not assume that dependencies preserve global storage, validate addresses, allocate objects, release memory, or return normally.

## Interface

### Entry and completion

No explicit parameters, ordinary return value, or normal return path are shown.

Execution nevertheless depends on implicit inputs:

- An inherited linkage value selecting the first indirect call target.
- A frame-like address base used for word accesses and subsequently modified.
- An inherited register value used in a late byte-store address.
- Two external slots containing 32-bit pointer-like values.
- Memory reachable through captured and computed addresses.
- Primary and secondary register outputs from dependencies.

Consequently, the absence of declared parameters does not establish a self-contained, parameter-free interface.

An iteration that reaches its end immediately begins another iteration. A dependency, memory fault, breakpoint handler, or other environmental event may prevent that progression; no recovery behavior is specified.

### Widths and notation

| Quantity | Represented width |
|---|---:|
| Byte | 8 bits |
| Halfword | 16 bits |
| Word or pointer-like value | 32 bits |
| Paired result | 64 bits, split into low and high 32-bit words |

Offsets are byte offsets. Halfword reads identified below as unsigned are zero-extended before subsequent use. Endianness, alignment requirements, address permissions, and the physical meaning of the frame-like base are unspecified.

The following are contract labels, not implementation identifiers:

- **Frame:** the current frame-like base.
- **Object A:** a pointer captured from the first external pointer slot.
- **Fixed base:** `0x21C48960`.
- **Scaled fixed value:** Fixed base logically shifted right by 18 bits, then multiplied by four; the result is `0x21C4`.
- **Small base:** an unsigned halfword used directly as an address base.
- **Object B:** a word read at Small base offset `0x44`, subsequently used as an address.
- **Table base:** a pointer captured from the second external pointer slot.
- **Object C:** a word read at Scaled fixed value offset `0x54`.
- **Record index** and **Field value:** captured, zero-extended halfwords.
- **First pair**, **second pair**, and **third pair:** paired dependency results captured at different points.

F-D01 through F-D37 identify call occurrences in execution order. They do not imply 37 distinct target functions.

## Behavior

### Ordering and arithmetic rules

The phases below occur in order, with operations within each phase ordered as listed. Computations are included where their placement affects the contract.

Reads capture values at the stated time. A later dependency changing the source storage does not cause a fresh read. This rule describes the represented data flow; it is not a general claim that dependencies preserve arbitrary registers or memory.

No explicit bounds checks, null checks, status checks, or error branches are shown.

Unsigned 32-bit shifts and arithmetic use their represented 32-bit results. Signed additions, signed multiplications, signed left shifts, and address calculations require the qualifications in **Uncertainties**. In particular, a later unsigned conversion does not establish defined behavior for an earlier overflowing signed operation.

### 1. Initialize and capture Object A

1. Establish the zero byte that will later be written at Object A offset `0x15`.
2. Invoke F-D01 with two explicit zero arguments.
3. After F-D01 returns, establish a separate unsigned zero byte for F-D06’s arithmetic.
4. Invoke F-D02.
5. Invoke F-D03.
6. Read Object A from the first external pointer slot.
7. Invoke F-D04.

Only the future Object A clearing byte is initialized before F-D01. The arithmetic zero byte is initialized after F-D01 and before F-D02.

On the first iteration, F-D01’s target is selected using the inherited linkage value plus `0xFFE`. Near the end of the iteration, the linkage value is explicitly replaced with a fixed continuation value. Subsequent iterations therefore use that updated value rather than necessarily retaining the entry value.

### 2. Process the first pair and perform narrow stores

1. Invoke F-D05 and capture the first pair.
2. Invoke F-D06 with two explicit arguments:
   - The first pair’s low word interpreted as signed 32-bit.
   - A value calculated by first adding the high and low words as signed 32-bit values, then converting that intermediate result to unsigned 32-bit and adding the separate unsigned zero byte.
3. Write F-D06’s secondary byte output at Object A offset `0x11`.
4. Invoke F-D07.
5. Write the previously established zero byte at Object A offset `0x15`.
6. Invoke F-D08.
7. Write F-D08’s 16-bit primary result at offset `0x0C` from its secondary address output.

The initial addition in F-D06’s second argument can overflow signed 32-bit arithmetic. The subsequent unsigned conversion and unsigned addition do not remove that uncertainty. Where the signed addition is defined, adding the unsigned zero leaves the converted value unchanged.

The Object A clearing write occurs after F-D07, not before it.

### 3. Process the second pair and publish derived words

1. Invoke F-D09 and capture the second pair.
2. Write a zero halfword at offset `0x2C` from the second pair’s high word interpreted as an address.
3. Logically shift the second pair’s low word right by 26 bits and write the result at Frame offset `0x26C`.
4. Establish Fixed base.
5. Invoke F-D10.
6. Read a pointer from Frame offset `0x320`.
7. Invoke F-D11.
8. Invoke F-D12.
9. Write the second pair’s low word, shifted left by two bits as an unsigned 32-bit value, at offset `0x54` from the pointer captured in step 6.

The Frame write contains a value from 0 through 63. The later shifted word discards bits shifted beyond the 32-bit width.

The destination pointer is read after F-D10 and before F-D11. A dependency subsequently modifying Frame offset `0x320` does not itself redirect the write that uses the captured pointer.

### 4. Reach the breakpoint and capture Object B

1. Invoke F-D13, F-D14, F-D15, F-D16, and F-D17, in that order.
2. Read an unsigned halfword at Fixed base offset `0x34`; retain its zero-extended value as Small base.
3. Invoke F-D18.
4. Read a word at Frame offset `0x280` and pass it to F-D19.
5. Capture F-D19’s signed 32-bit primary result.
6. Execute a software breakpoint with immediate `0x92`.

The remainder executes only if control continues beyond the breakpoint:

7. Shift the captured F-D19 result left by two bits using the represented signed operation.
8. Derive Scaled fixed value by logically shifting Fixed base right by 18 bits and multiplying by four.
9. Read Object B from Small base offset `0x44`.

Small base is the halfword value itself, not Fixed base plus that value. Its possible unsigned range is 0–65,535.

The breakpoint precedes both the scaling of F-D19’s result and the Object B read. The signed shift is not an established checked index conversion.

### 5. Perform table and halfword updates

1. Invoke F-D20.
2. Invoke F-D21 and capture its word result.
3. Read a word at Small base offset `0x04`.
4. Invoke F-D22 with F-D21’s captured result and the word just read.
5. Read Table base from the second external pointer slot.
6. Invoke F-D23.
7. Write a halfword obtained by converting Table base to signed 16-bit at the address formed from Table base plus the scaled F-D19 result.
8. Invoke F-D24.
9. Read Object C from Scaled fixed value offset `0x54`.
10. Invoke F-D25 and capture its pointer-like primary result.
11. Write Scaled fixed value, converted to signed 16-bit, at offset `0x04` from that result.

The Object C read is at numeric address `0x2218`. The final halfword value is `0x21C4`, which is representable as signed 16-bit.

The Table base conversion may start outside the signed 16-bit range. Under conventional low-bit narrowing, its stored bit pattern is Table base’s low 16 bits; the supplied representation alone does not define every out-of-range signed conversion.

Table base is captured before F-D23. The table write does not reread the external slot afterward.

### 6. Capture fields and modify Frame

1. Invoke F-D26.
2. Invoke F-D27.
3. Read an unsigned halfword at Object C offset `0x34`; retain its zero-extended value as Record index.
4. Invoke F-D28 and capture the third pair.
5. Write the third pair’s low word at Frame offset `0x258`.
6. Read an unsigned halfword at offset `0x14` from the third pair’s high word interpreted as an address; retain its zero-extended value as Field value.
7. Form a signed left-shift-by-six result from the third pair’s low word.
8. Multiply the third pair’s high word, interpreted as signed 32-bit, by four.
9. Decrease Frame by `0xF4`, or 244 bytes.
10. Multiply Record index by four using unsigned arithmetic.
11. Write the third pair’s low word at address `4 × Record index + 0x74`.

The Frame offset-`0x258` write uses Frame before the decrement. Subsequent iterations use the changed Frame in the represented flow; no restoration appears.

Because Record index is an unsigned halfword, its multiplication by four and addition of `0x74` fit within 32 bits. That arithmetic fact does not establish that the resulting address is accessible.

The third pair’s signed shift and signed multiplication have separate domain restrictions. Neither is established as an unconditional wrapping operation.

### 7. Replicate Field value and finish the iteration

1. Invoke F-D29.
2. Write Field value as a zero-extended 32-bit word at Object B offset `0x54`.
3. Invoke F-D30.
4. Write the same captured Field value as a 32-bit word at Object B offset `0x64`.
5. Invoke F-D31.
6. Write a halfword obtained by converting Object B to signed 16-bit at the address formed from the previously calculated shift-by-six result plus F-D31’s secondary output.
7. Invoke F-D32.
8. Invoke F-D33.
9. Read one byte at address `5 × Record index`.
10. Invoke F-D34.
11. Write the captured byte at the address formed from Field value, the inherited additional register value, and offset `0x19`.
12. Invoke F-D35.
13. Write the previously calculated four-times-high-word result at Object B offset `0x44`.
14. Invoke F-D36.
15. Replace the linkage value with the fixed continuation value used for the next iteration.
16. Read a word at offset `0x44` from the previously calculated four-times-high-word result.
17. Invoke F-D37 with that word.
18. Begin the next iteration.

The byte-source formula has no additional array base. Its arithmetic fits within 32 bits for every possible Record index.

Both Field value writes use the same capture, despite the intervening F-D30 call. Likewise, F-D34 occurs between the byte read and the corresponding byte write.

The Object B halfword conversion has the same out-of-range signed-narrowing qualification as the Table base conversion.

## Side effects

Direct memory effects include:

- Byte writes at Object A offsets `0x11` and `0x15`.
- Halfword writes through dependency outputs, paired-result addresses, a table-derived address, and a shifted-address expression.
- Word writes relative to Frame.
- A word write at the Record index-derived address.
- Two word writes of Field value into Object B.
- A byte transfer between computed addresses.
- A final word write at Object B offset `0x44`.

No disjointness is established. These locations may alias one another or storage read later. The ordering above therefore matters even when two labels appear to describe different objects.

Control-state effects include a breakpoint, a Frame decrement, linkage replacement, and unconditional repetition.

No direct allocation, deallocation, ownership transfer protocol, rollback, or cleanup path is shown. A pointer-derived store is not evidence of ownership transfer.

No explicit error result is reported. Invalid addresses, invalid call targets, dependency behavior, arithmetic outside a defined domain, or breakpoint handling may prevent further execution. Their outcomes remain unspecified.

## Dependencies

| Opaque occurrence IDs | Visible interface role |
|---|---|
| F-D01 | Two explicit zero arguments; linkage-relative dispatch |
| F-D02–F-D04 | No explicit arguments; primary results unused |
| F-D05 | Produces the first pair |
| F-D06 | Two explicit word arguments; secondary byte output consumed |
| F-D07 | No explicit arguments; primary result unused |
| F-D08 | Produces a halfword and secondary address |
| F-D09 | Produces the second pair |
| F-D10–F-D18 | No explicit arguments; primary results unused |
| F-D19 | Consumes a Frame-resident word; produces a signed word later shifted |
| F-D20 | No explicit arguments; primary result unused |
| F-D21 | Produces a word passed to F-D22 |
| F-D22 | Consumes two explicit words |
| F-D23–F-D24 | No explicit arguments; primary results unused |
| F-D25 | Produces a pointer-like destination |
| F-D26–F-D27 | No explicit arguments; primary results unused |
| F-D28 | Produces the third pair |
| F-D29–F-D30 | No explicit arguments; primary results unused |
| F-D31 | Supplies a secondary address component |
| F-D32–F-D36 | No explicit arguments; primary results unused |
| F-D37 | Consumes a word read through a scaled address |

“No explicit arguments” does not exclude implicit inputs. An unused primary result does not exclude memory, register, ownership, or control-flow effects. Canonical dependency identities and complete contracts are unavailable.

## Acceptance examples

These are conditional criteria, not executed tests. Each assumes valid relevant addresses, returning dependencies, defined relevant arithmetic, and breakpoint continuation where required.

1. **Initialization order.** The clearing byte is established before F-D01. The separate arithmetic zero is established only after F-D01 returns and before F-D02. A trace that places both initializations before F-D01 does not match the represented order.

2. **Signed addition followed by unsigned conversion.** If the first pair’s high word is `0xFFFFFFFF` and its low word is `0x00000001`, the signed intermediate sum is zero. F-D06 receives zero as its second argument after unsigned conversion and addition of the unsigned zero.

3. **Overflow remains unresolved.** If those words are `0x7FFFFFFF` and `0x00000001`, their initial signed addition exceeds the signed 32-bit range. This contract does not require `0x80000000` as the argument merely because the subsequent addition is unsigned.

4. **Top-six-bit extraction.** For second-pair low word `0xFC000003`, the Frame offset-`0x26C` write is 63. The later offset-`0x54` write through the captured pointer is `0xF000000C`.

5. **Post-call clearing.** If F-D07 writes a nonzero byte at Object A offset `0x15`, the following direct write replaces it with zero before F-D08.

6. **Captured destination.** If F-D10 establishes pointer P at Frame offset `0x320` and F-D11 replaces that slot with Q, the later offset-`0x54` write still uses P.

7. **Captured field replication.** For Field value `0xFEDC`, both Object B word writes contain `0x0000FEDC`, even if F-D30 changes the original halfword source.

8. **Record index boundaries.** Index zero produces word destination `0x74` and byte source zero. Index 65,535 produces word destination `0x40070` and byte source `0x4FFFB`. No visible guard skips these accesses.

9. **Breakpoint stop.** If execution stops permanently at breakpoint `0x92`, the later Object B capture, table update, Frame decrement, and remaining transfers do not occur.

10. **Repeated Frame adjustment.** Two iterations reaching the decrement reduce Frame by 488 bytes, provided the address calculations are defined and no external mechanism changes Frame.

11. **Conditional narrowing.** If Table base ends in `0xABCD` and the execution model uses low-bit signed narrowing, its table halfword store has bit pattern `0xABCD`. This does not establish a complete pointer store or ownership transfer.

## Uncertainties

The following remain **documented_only**:

- Application-level purpose and semantic meanings of slots, objects, and offsets.
- Dependency identities, calling conventions, implicit inputs, register preservation, global mutation, ownership behavior, and nonreturning behavior.
- Whether secondary register outputs are intentional interfaces or limitations of the supplied representation.
- The physical meaning of Frame and the executable significance of its repeated decrement.
- Endianness, alignment, address mapping, and accessibility of low numeric addresses.
- Signed overflow, signed left-shift behavior outside a defined domain, and out-of-range signed narrowing.
- Whether address calculations wrap, fault, or depend on external invariants.
- Breakpoint continuation, failure handling, and the fidelity of the represented unconditional loop.

These unknowns must not be resolved by assuming conventional library behavior, adding unstated checks, or treating every arithmetic operation as unsigned wrapping arithmetic.

## Evidence

**Established static evidence:** the supplied representation shows the two distinct initialization positions; the signed intermediate addition followed by unsigned conversion and addition; the access widths and offsets; dependency ordering; captured-value uses; breakpoint placement; Frame adjustment; linkage replacement; and unconditional repetition described above.

**Derived facts:** Scaled fixed value is `0x21C4`; its offset-`0x54` read is at `0x2218`; the Record index formulas and acceptance-example arithmetic follow from the represented widths and constants.

**Inferred intent:** “Object,” “Table,” “record,” and “Frame” are organizational labels. They do not establish types, valid indices, ownership, or a stack discipline.

**Verification status:** static analysis only. No runtime tests have been performed. No dependency implementations, independent execution traces, or external interface documentation were supplied.
