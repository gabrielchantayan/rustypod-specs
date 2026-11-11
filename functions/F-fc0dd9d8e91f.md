<!-- Trusted provenance: F-fc0dd9d8e91f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fc0dd9d8e91f

## Purpose

F-fc0dd9d8e91f specifies a sequence of memory accesses, global assignments, address calculations, and abnormal termination points. The supplied static evidence does not establish an application-level purpose, a successful result, or a normal return path.

**Status: documented_only.** The representation reports invalid instruction data and overlapping global symbols. It also contains shifts by 32 on represented 32-bit signed operands. Those operations remain unresolved; this contract does not replace them with zero, sign extension, or masked shift counts.

The following is a contract of the displayed static effects, conditional on execution reaching them. It does not establish that all effects are executable or reachable on a real target. No runtime tests have been performed.

## Interface

### Explicit inputs

The displayed interface has four arguments and a void return declaration.

| Input | Width | Established use |
|---|---:|---|
| First argument | 32 bits | Supplies a low byte and a signed-right-shift-derived address seed |
| Second argument | 32 bits | No direct use shown |
| Third argument | 32 bits | No direct use shown |
| Fourth argument | 32 bits, address-bearing | Supplies the base for a byte store at offset `0x0B` |

No normal completion is displayed, so the void declaration is not evidence of a reachable ordinary return.

### Additional entry state

The operation also depends on:

- A **primary base**, used in memory accesses and signed arithmetic.
- A **secondary base**, used for a word read at offset `0x5C`.
- An **incoming byte**.
- A **stack-derived pointer**, represented as obtained from an entry-stack slot.
- The numeric address of that stack slot, separately from its pointer contents.
- Global storage and numeric associations with two opaque dependencies.

The four explicit arguments are therefore insufficient to establish a complete calling convention. The evidence does not establish when or how the additional entry values become available.

### Widths and memory requirements

Bytes, halfwords, and words are respectively 8, 16, and 32 bits. Represented pointers are 32 bits. Explicit unsigned byte and halfword loads zero-extend when used as wider values. Explicit narrowing retains the low-order bits.

Global destination declarations are unavailable. A conversion to an 8- or 16-bit expression does not, by itself, establish the storage width of the global receiving that expression. Taking a symbol’s address likewise does not establish the referenced symbol’s size.

No validation precedes the displayed accesses. Continuing without interruption requires the actual addresses to satisfy the target’s mapping, permission, and alignment requirements. Endianness, alignment rules, and pointer-conversion behavior are unresolved.

No allocation, release, ownership transfer, or cleanup is shown. Address use does not establish ownership or extend the lifetime of the addressed storage.

## Behavior

The ordering below follows the displayed operations, not a proven hardware memory-ordering model. It does not impose an operand-evaluation order within a single expression where none is established. Aliasing can affect later reads; a previously captured value remains distinct from a later read of its original storage.

### 1. Initial reads and stores

First, capture a word from primary base plus `0x38`, then a word from secondary base plus `0x5C`. These are the **first entry word** and **second entry word**.

Perform these byte stores in order:

1. Low byte of the second entry word, at primary base plus `1`.
2. Incoming byte, at fourth argument plus `0x0B`.
3. Low byte of the first argument, at primary base plus `0x15`.
4. Low byte of the first entry word, at second entry word plus `0x1F`.

Next, capture two global word values, in order, as the **first global base** and **second global base**. Then read an unsigned byte at first entry word plus `0x0A`; its zero-extended value is the **entry byte**.

The global reads occur after the four byte stores. They are not entry-time snapshots.

### 2. Seed-dependent accesses

The **seed** is the first argument shifted right by 23 as signed 32-bit data, then retained as unsigned 32 bits. Under arithmetic signed right shift, the signed intermediate ranges from −256 through 255. Negative intermediates become their unsigned 32-bit representations. Target signed-shift semantics are not independently established.

Continue in this order:

1. Store the low halfword of the primary base at primary base plus its signed right shift by 11.
2. Capture a word at second global base plus the seed with bits `0x88` set.
3. Capture an unsigned byte at the signed-right-shifted primary base plus that same bit-set seed.
4. Store the captured word at entry byte plus `8`.
5. Store the entry byte as a word at seed plus `0x40`.
6. Store the captured word at first global base plus `0x74`.
7. Capture a word at entry byte plus `0x28`.
8. Capture another word at first global base plus `0x5C`.
9. Store the low byte of the first global base at the address given by the step-2 word plus `3`.
10. Store the step-3 byte at first global base plus `0x10`.
11. Store the low byte of the step-8 word at the zero-extended step-3 byte plus `0x1C`.

The destinations derived from unsigned bytes are low numeric addresses, not implicitly pointers into an established object.

### 3. Masking, halfword stores, and unresolved pointer formation

Define:

- **Reduced seed:** seed minus `0x38`.
- **Doubled base:** twice the result of seed minus `0xE0`.
- **Masked difference:** seed minus `0xE0`, bitwise AND `0x7FFFFFFF`.
- **Half-offset:** masked difference shifted logically right by one.

These seed-based operations use unsigned 32-bit arithmetic. The half-offset ranges from zero through `0x3FFFFFFF`.

The word captured at entry byte plus `0x28` also supplies an **unresolved pointer**: the representation shifts that signed 32-bit word left by two, then right by 32. Neither this pointer nor the validity of its formation is established.

The next effects are:

1. Store the low byte of the half-offset at that captured word plus `0x0897E8B8`.
2. Assign `0xB8` to a global.
3. Store the low byte of the reduced seed at seed minus `0x32`.
4. Store the reduced seed as a word through the unresolved pointer.
5. Store halfword `0xFFFF` at doubled base plus `4`.
6. Store halfword `0xFFFF` at doubled base plus `6`.
7. Store byte `0xB8` at unresolved pointer plus `4`.
8. Store halfword `0xFFFF` at doubled base plus `2`.
9. Assign the low 16 bits of the doubled base to another global.
10. Read an unsigned halfword at the doubled base, zero-extend it to an address, and store there a word equal to reduced seed AND `0x897E8B80`.

The sentinel stores occur at offsets `4`, `6`, then `2`. The halfword read occurs afterward, including after the intervening stores and global assignment.

### 4. Dependency-associated values and sign branch

The representation separately evaluates the numeric association with **F-e47d7b2e9c88** for three purposes:

1. Capture an unsigned 32-bit value for the sign test and self-addressed stores.
2. Interpret another evaluation as signed 32 bits and shift it right by 32, forming an unresolved **branch base**.
3. Interpret another evaluation as signed 32 bits, shift it right by 31, and narrow to a **sign halfword**.

These are separate displayed evaluations, not an established single cached value. Their stability is unknown. In particular, the shift by 32 must remain unresolved.

Store the sign halfword at branch base plus `4`. Then test the captured value as signed 32 bits.

If it is negative, execution reaches the first invalid-instruction endpoint. No later operation is displayed on that branch.

Otherwise, continue:

1. Store the sign halfword at branch base plus `0x12`.
2. Store the captured dependency-associated word through its own numeric value as an address.
3. Read a signed halfword at branch base plus the half-offset.
4. Repeat the sign-halfword store at branch base plus `0x12`.
5. Repeat the self-addressed word store.
6. Sign-extend the captured signed halfword to 32 bits, shift it right by 16, and store the word at branch base plus `8`.

Under arithmetic right shift, the final word is zero for a nonnegative halfword and `0xFFFFFFFF` for a negative halfword. The repeated stores are distinct effects and must not be collapsed.

### 5. Global assignments and selector-based stores

On the continuing branch:

1. Assign zero to a global.
2. Form a **selector** by adding `0x58` to the captured dependency-associated value AND `4`. The selector is `0x58` or `0x5C`.
3. Store the branch base as a word at selector plus `4`, giving numeric address `0x5C` or `0x60`.
4. Logically shift the masked difference right by 17, convert to unsigned 16 bits, and assign the result to another global.
5. Convert the half-offset to unsigned byte, then unsigned halfword, and assign it to a different global.
6. Store the complete half-offset as a word at four times the selector plus `0x10`, giving numeric address `0x170` or `0x180`.
7. Assign the complete half-offset to another global.

The step-4 expression is at most `0x3FFF`; step 5 retains only the half-offset’s low byte. Neither conversion establishes the destination global’s storage width. Step 7 preserves the full half-offset and follows the selector-based word store.

### 6. Stack-derived pointer effects

Through the stack-derived pointer, perform these stores in order:

1. Selector as a halfword at the pointer’s base.
2. Branch base shifted left by eight as a word at the base.
3. The same shifted word at offset `0x14`.
4. The pointer’s own 32-bit numeric value as a word at the base.

Then read a word at twice the pointer’s numeric value. Store its low halfword at the pointer’s base, followed by a word store of the complete captured value through that value interpreted as an address.

Next:

1. Store a word at 32 times the **entry-stack slot’s address**, plus `0x18`. The stored expression is the stack-derived pointer interpreted as signed 32 bits and shifted right by 31.
2. Read the currently visible byte-sized overlapping global associated with the narrowed assignment in the preceding section. Concatenate the 16-bit constant `5` above that byte and assign the resulting 24-bit expression to the wider overlapping global.
3. Store the stack-derived pointer, at pointer width, at four times the **pointer’s numeric value**, plus `0x18`.

Under arithmetic right shift, the first stored expression is zero or `0xFFFFFFFF`.

The stack slot’s address and its pointer contents are different address sources. The overlapping-byte reconstruction performs a read at this point; it is not specified to reuse the earlier narrowed expression.

### 7. Final capture, publication, and fresh global reads

Capture a final global word, then evaluate the numeric association with **F-8afdaab91693**.

Continue:

1. Publish four times the stack-derived pointer’s numeric value to a pointer-bearing global.
2. Evaluate F-8afdaab91693 again, after publication, and shift this second evaluation left by two for a later store.
3. Read the pointer-bearing global for a self-addressed word store: its value supplies the destination and its value supplies the stored word. The ordering of the operand reads within this store is not established.
4. Read that global again and store the full half-offset at its current value plus `4`.
5. Store four times the captured final global word through that calculated value as a word address.
6. Store the shifted second dependency-associated evaluation at the captured final global word plus that word shifted left by six and then signed-right-shifted by 17.

Publication does not make later global reads equivalent to the published value. The self-addressed store may alias storage read by the subsequent operation. The two F-8afdaab91693 evaluations are also distinct and occur on opposite sides of publication.

### 8. Final global and fixed-value stores

Next:

1. Interpret the captured final global word as unsigned, shift it logically right by 16, convert to 16 bits, and assign it to an **upper-halfword global**.
2. Assign zero to another global.
3. Multiply the first final-region evaluation of F-8afdaab91693, represented as signed, by eight to form an address.
4. Store zero words at that address and at offset `4`.
5. Store a byte at offset `0x10`, with value equal to the multiplied address’s low byte plus eight, retained to eight bits.

Separate global destinations then receive, in order:

1. `0x897EA700`.
2. `0x66`.
3. `0xEAEC`.
4. `0x1A`.
5. The address of the referenced symbol associated with the self-addressed global destination.
6. A reconstructed word formed from the **current** upper-halfword global above low halfword `0xEA70`.
7. `0x5C`.

For assignment 5, the referenced symbol’s size is unresolved: the representation supplies its address but neither its declaration nor an access establishing its storage width. The receiving global’s declaration is also unavailable.

The reconstruction in assignment 6 performs a current read after the preceding stores. It does not necessarily reuse the upper half of the previously captured final global word. Its constructed value is 32 bits.

The final displayed operation is an invalid-instruction endpoint. No normal return follows.

## Side effects

Established static effects include byte, halfword, word, and pointer-width stores; global assignments; repeated writes; overlapping-width overwrites; and writes through explicit inputs, hidden entry state, loaded values, and dependency-associated numeric values.

Some targets are derived from zero-extended bytes or halfwords. Others are fixed low numeric addresses, scaled pointer values, or unresolved shifts. None is established as valid merely because it appears as a destination.

There is no visible transaction, rollback, synchronization, error result, cleanup, or recovery path. An actual memory fault could interrupt the sequence before an invalid-instruction endpoint. The evidence does not establish hardware-level visibility or commitment of preceding stores after a fault.

Unknown aliasing requires preserving displayed reads and their order. It does not justify replacing a later global read with an earlier captured value.

## Dependencies

| Opaque dependency | Established relationship | Unresolved properties |
|---|---|---|
| **F-e47d7b2e9c88** | Three numeric-associated evaluations contribute to sign testing, shifts, addressing, and stores | Meaning, evaluation stability, address validity |
| **F-8afdaab91693** | Two numeric-associated evaluations occur around pointer publication; the first supplies multiplication by eight, the second a left shift by two | Meaning, stability across publication, address validity |

Neither dependency is visibly invoked. No callable argument contract, return contract, global-preservation guarantee, or memory ownership behavior follows from these references.

The invalid-instruction endpoints do not establish calls to an error handler.

## Acceptance examples

These are conditional static obligations, not executed tests. Each assumes its operations are reached without an earlier interruption and that unresolved arithmetic and addressing prerequisites are satisfied.

1. **Initial truncation:** First argument `0x12345678` supplies byte `0x78` for primary base plus `0x15`.

2. **Unsigned byte addressing:** Entry byte `0xA5` selects numeric address `0xAD` for the plus-eight word store. It is not sign-extended.

3. **Sentinel ordering:** Halfword `0xFFFF` is stored at doubled-base offsets `4`, `6`, and `2`, in that order.

4. **Negative branch:** A captured negative F-e47d7b2e9c88-associated value reaches the first invalid-instruction endpoint only after the sign-halfword store at branch base plus `4`.

5. **Distinct narrowing:** A masked difference of `0x7FFFFFFF` yields `0x3FFF` after logical right shift by 17. Its half-offset is `0x3FFFFFFF`; conversion through unsigned byte and unsigned halfword yields `0x00FF`. The later full-value assignment receives `0x3FFFFFFF`.

6. **Current-byte reconstruction:** If the overlapping byte reads as `0xA7` when reconstruction occurs, the constructed expression is `0x0005A7`, irrespective of an earlier assigned byte.

7. **Slot versus pointer:** With hypothetical stack-slot address `0x1000` and pointer contents `0x2000`, the slot-address-based store targets `0x20018`; the later pointer-scaled store targets `0x8018`.

8. **Fresh read after publication:** If the self-addressed store changes the pointer-bearing global, the following offset-4 destination uses its newly read value rather than automatically using the published value.

9. **Current upper halfword:** If the upper-halfword global reads as `0x1234` at final reconstruction, the constructed word is `0x1234EA70`, even if its earlier assignment differed.

10. **Signed propagation:** Under arithmetic right shift, captured signed halfword `0x8000` produces word `0xFFFFFFFF` after sign extension and right shift by 16; `0x7FFF` produces zero.

11. **Address without size inference:** The fifth final fixed-value assignment supplies the referenced symbol’s address. That operation alone imposes no byte-sized, halfword-sized, or word-sized storage requirement on the referenced symbol.

## Uncertainties

The following remain **documented_only**:

- Function-boundary validity and completeness of the calling convention.
- Results and validity of represented 32-bit operands shifted by 32.
- Target semantics for signed right shifts, signed left shifts, and signed overflow.
- Pointer conversions, byte order, alignment, permissions, and low-address mappings.
- Global declaration widths and exact overlap relationships.
- The size of the symbol whose address is stored in the final fixed-value sequence.
- Stability and meaning of dependency-associated evaluations.
- Effects of intervening stores on later global and overlapping-subfield reads.
- Physical behavior of the invalid-instruction endpoints.
- Application-level intent, successful completion, and normal return behavior.

Explicit unsigned 32-bit arithmetic wraps modulo \(2^{32}\). This rule does not resolve signed overflow, invalid shifts, or pointer arithmetic.

No application-level intent is inferred. Patterns resembling sign propagation, sentinel initialization, or pointer publication describe displayed dataflow, not a proven higher-level design.

## Evidence

Established evidence consists of the supplied static representation: explicit access widths and conversions, ordered operations, separate value captures and later reads, overlapping-global warnings, a sign-dependent branch, and two invalid-instruction endpoints.

The address-taking operation in the final fixed-value sequence establishes an address value, not the referenced symbol’s storage width. Widths cannot be recovered from symbol spelling or apparent naming conventions.

No independent executable inspection, dependency inspection, runtime execution, memory trace, or test result is available. All acceptance examples are conditional interpretations of static evidence, not observations of machine behavior.
