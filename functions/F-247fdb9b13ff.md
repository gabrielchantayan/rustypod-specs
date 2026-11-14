<!-- Trusted provenance: F-247fdb9b13ff; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-247fdb9b13ff

## Purpose

F-247fdb9b13ff is a partially recovered routine that interleaves indirect dependency calls with byte-sized and 32-bit memory writes. Its visible behavior includes scaling dependency results, extracting bit fields, using bytes as destination selectors, and storing values at computed addresses.

No application-level purpose is established. In particular, the available evidence does not establish that the addressed storage represents objects, hardware registers, a buffer, or a serialized format.

The supplied representation explicitly reports invalid instruction data and ends at a truncated control-flow boundary. This contract therefore describes the **visible prefix**, conditional on execution reaching each operation. It does not establish a complete callable implementation, successful completion, or a normal return.

## Interface

### Explicit inputs

The recovered interface presents four input slots:

| Input | Width | Visible use |
|---|---:|---|
| First input | 32 bits | Passed to the first indirect dependency. |
| Second input | 32 bits | No explicit use in the visible prefix. |
| Third input | 32 bits | No explicit use in the visible prefix. |
| Fourth input | 32 bits | Shifted left by 10 positions; the resulting value is supplied twice to one dependency. |

“No explicit use” does not establish that an input is irrelevant to the original executable. The recovery is incomplete, and the dependency calling convention is not established.

### Additional entry state

The visible prefix also depends on entry state outside those four inputs:

- An inherited address-like value used for the first byte-selector read.
- An inherited 32-bit value used by the first word store.
- An inherited control-transfer value used to derive the first dependency target.
- A subsequently exposed 32-bit secondary result, whose producing call is not identified reliably by the representation.
- Memory contents at dynamically computed addresses.
- The availability and behavior of numerous indirect dependencies.

These are required environmental inputs to the visible behavior, not established public parameters.

### Output and termination

No explicit return value is established. One dependency supplies a recovered 64-bit result that is split into lower and upper 32-bit halves for later operations.

The visible prefix ends after a final indirect call at a bad-instruction boundary. Whether the original executable returns, faults, continues through unrecovered instructions, or transfers elsewhere is **documented_only**.

### Preconditions

For the entire visible prefix to execute:

- Every reached indirect target must be executable under the actual instruction-set and calling-convention rules.
- Every reached memory read and write must be permitted at its computed address.
- Dependencies must return when subsequent behavior is expected.
- The actual execution model must support the arithmetic and access patterns represented here.

There are no visible checks enforcing these conditions.

## Behavior

### Interpretation and arithmetic

For clarity, this contract uses mathematical names for observed values, not implementation identifiers:

- **Entry base**: the inherited address-like value.
- **Entry word**: the inherited 32-bit word.
- **Primary value**: an early 32-bit dependency result.
- **Secondary value**: the incompletely attributed 32-bit secondary result.
- **Lower half** and **upper half**: the two halves of the later 64-bit dependency result.

Unless otherwise qualified, arithmetic expressions below describe 32-bit bit-vector behavior: multiplication and left shifts retain the low 32 bits. Right shifts of explicitly unsigned values are logical. A byte write retains only the low eight bits of its value.

This bit-vector interpretation is well supported for unsigned expressions. Several recovered expressions instead use signed 32-bit operands for multiplication, address formation, and left shifting. Their exact machine behavior is not established merely by the recovered signed types. Examples relying on overflow or negative signed operands remain conditional on confirmation of the executable semantics.

Each byte selector is an unsigned eight-bit load. A destination formed by adding 17 to such a selector lies between addresses 17 and 272 inclusive. It is not shown as an offset from an object base.

Each word access has a recovered width of 32 bits. Byte order, alignment requirements, address-space mapping, and fault behavior are not established.

### Dependency notation and ordering

Dependency IDs below are local opaque identifiers. An ID range such as **F-D02.1–F-D02.5** identifies five separate call occurrences; it does not assert that their targets are distinct functions.

Most runs contain:

1. A call with the stated explicit arguments.
2. Four calls with no explicit arguments.

For those runs, the fifth call supplies any result named in the description. Intermediate results have no explicit consumers in the visible representation. “No explicit arguments” does not prove that a dependency cannot inspect registers, globals, or other machine state.

Dependency calls are observable ordering boundaries. They must not be removed, merged, or reordered merely because a recovered result appears unused.

### Initial word and byte writes

**F-D01.1–F-D01.5** begins with a call receiving the first input, followed by four calls with no explicit arguments. The final result is an index.

The routine writes the entry word to the 32-bit destination at:

**4 × index + 20**

It then executes **F-D02.1–F-D02.5**, passing **4 × index** to the first call. The final call supplies the primary value.

Next, the routine reads one byte from **entry base + 1** and writes the low byte of **4 × primary value** to **selector + 17**.

**F-D03.1–F-D03.5** follows, beginning with the argument **4 × primary value**. Its final result supplies another index.

The routine then:

- Reads a byte at **64 × primary value + 1**.
- Retains that byte as a selector.
- Writes the low byte of **(primary value AND 0x3fffffff) shifted right by 20** to **selector + 17**.

The mask and shift produce a ten-bit intermediate value. The byte store discards its upper two bits.

**F-D04.1–F-D04.5** begins with **4 × the preceding index** and yields another index.

### Fourth-input contribution and zero byte

**F-D05.1–F-D05.4** has a different argument pattern:

| Occurrence | Explicit arguments |
|---|---|
| F-D05.1 | Four times the latest index; the retained selector |
| F-D05.2 | None |
| F-D05.3 | Fourth input shifted left by 10, supplied twice |
| F-D05.4 | None |

After these calls, the routine reads a byte at address **1** and writes zero to **that byte + 17**.

The fixed read address follows from a locally established zero multiplied by a large scale factor. It is not an input-relative access. The read cannot be omitted: its value selects the destination of the zero write.

### Secondary-value contribution

**F-D06.1–F-D06.5** begins with three explicit arguments:

- Secondary value.
- Secondary value again.
- Secondary value multiplied by 16.

Its final result supplies an index.

The routine reads a byte at **16 × secondary value + 1**, retains it as a selector, and writes the low byte of **secondary value shifted right by 22** to **selector + 17**. The shifted value can contain ten significant bits; only eight are stored.

**F-D07.1–F-D07.5** begins with four times the preceding index and supplies another index.

**F-D08.1–F-D08.4** then performs:

| Occurrence | Explicit arguments or result |
|---|---|
| F-D08.1 | Retained selector first; four times the latest index second |
| F-D08.2 | No explicit arguments |
| F-D08.3 | Two zero arguments |
| F-D08.4 | No explicit arguments; recovered 64-bit result |

The selector/index order here differs from F-D05.1 and must be preserved.

The recovered 64-bit result is split numerically into lower and upper 32-bit halves. The routine writes zero to **4 × upper half + 20**.

### Word-store chain

The following operations occur in order, with the listed dependency runs between stores.

| Dependency activity | Following visible memory effect |
|---|---|
| F-D09.1–F-D09.5; first argument is upper half; final result is an index | Store upper half at **4 × index + 20**. |
| F-D10.1–F-D10.5; first argument is four times the preceding index | Store **lower half × 0x04000000** at **lower half × 0x10000000 + 20**. |
| F-D11.1–F-D11.5; first argument is **lower half × 0x04000000**; final result is an index | Store **(lower half AND 0x1fffffff) shifted right by 9** at **4 × index + 20**. |
| F-D12.1–F-D12.5; first argument is four times the preceding index; final result is a new index | Store four times the new index at **16 × new index + 20**. |

The mask-and-shift store retains original lower-half bits 9 through 28, producing at most 20 significant bits.

After the final row, the routine supplies three arguments to **F-D13.1**:

- A 32-bit value read from **4 × latest index + 20**, shifted right by 12.
- Four times that index.
- A 32-bit value read from the same address, shifted left by two.

These are two represented reads from the same location. Their relative evaluation order is not established. If the location is volatile, concurrently modified, or side-effecting, the two values need not agree.

F-D13.2–F-D13.5 have no explicit arguments. The final result supplies an index. The routine then stores the previous scaled index at **4 × new index + 20**.

### Repeated scaled-index transitions

The next transitions preserve distinct destination formulas:

| Dependency run | First call’s explicit arguments | Effect after the run |
|---|---|---|
| F-D14.1–F-D14.5 | Four times the previous result index | With the new index, store **4 × index** at **16 × index + 20**. |
| F-D15.1–F-D15.5 | Current scaled index twice; lower half shifted left by 26 | Store the current scaled index at **4 × new result index + 20**. |
| F-D16.1–F-D16.5 | Four times the previous result index | With the new index, store **4 × index** at **16 × index + 20**. |
| F-D17.1–F-D17.5 | Current scaled index | Store the current scaled index at **4 × new result index + 20**. |
| F-D18.1–F-D18.5 | Four times the previous result index | With the new index, store **4 × index** at **16 × index + 20**; then read a word at **4 × index + 20**. |
| F-D19.1–F-D19.5 | Current scaled index | Store the previously read word shifted right by 12 at **4 × new result index + 20**. |
| F-D20.1–F-D20.5 | Four times the previous result index | With the new index, store **4 × index** at **16 × index + 20**. |

The write at **16 × index + 20** does not establish the contents of **4 × index + 20**. They are different address calculations and must not be treated as one field.

The word read following F-D18 is retained across the F-D19 dependency calls. It is not reread immediately before the shifted store.

### Final field extraction and byte writes

After F-D20, **F-D21.1** receives:

- The low six bits of the scaled index produced by F-D19, shifted left by 14.
- The scaled index produced by F-D20.
- The unscaled result index from F-D19, shifted left by 30.

F-D21.2–F-D21.5 have no explicit arguments. The resulting index selects a word destination at **4 × index + 20**, where the routine stores the scaled index from F-D20.

**F-D22.1–F-D22.5** begins with four times the latest result index. Its final result is a new 32-bit value.

The routine reads a selector byte at:

**F-D19 result index × 0x10000000 + 1**

It writes the low byte of four times the new value to **selector + 17**.

The final dependency sequence is:

| Occurrence | Explicit arguments or result |
|---|---|
| F-D23.1 | Four times the new value, repeated twice; the new value multiplied by 16 |
| F-D23.2 | No explicit arguments; produces a 32-bit word |
| F-D23.3 | That word |
| F-D23.4–F-D23.7 | No explicit arguments; final result supplies an index |

F-D23.1 uses a target-address adjustment different from the prevalent adjustment elsewhere. Its target cannot safely be normalized to the other call sites.

The routine then rereads the selector byte at the same address used before F-D23. It writes the low byte of **(new value AND 0x3fffffff) shifted right by 20** to the newly selected destination **selector + 17**.

**F-D24.1–F-D24.5** begins with four times the F-D23 result index and supplies another index.

**F-D25.1** receives four times that latest index, followed by the selector from the second byte read. **F-D25.2** has no explicit arguments.

The visible prefix ends at the reported invalid-instruction boundary.

## Side effects

Established direct effects comprise:

- Byte reads used as unsigned destination selectors.
- Byte writes to selector-derived addresses.
- 32-bit reads and writes at computed addresses.
- Ordered indirect dependency calls.
- Consumption of entry state beyond the four explicit input slots.

No explicit allocation, release, ownership transfer, synchronization, error reporting, rollback, or cleanup is visible.

This absence does not constrain dependency behavior. Dependencies may alter memory, global state, address mappings, or external resources. They may also fault or fail to return.

Aliasing is unrestricted by the evidence. Selector sources, byte destinations, word destinations, and dependency-accessed storage may overlap. Later reads observe the memory state at their own execution points, not an assumed entry snapshot.

A fault after earlier writes does not have a visible rollback path. Atomicity and transactional behavior are not established.

## Dependencies

The opaque IDs F-D01.1 through F-D25.2, with the occurrence ranges specified above, identify unresolved indirect dependency call sites.

No dependency implementation or behavioral contract is available. Consequently:

- Explicit argument widths are recovered as 32 bits unless a 64-bit result is specifically identified.
- Lack of explicit arguments is not proof of a zero-argument ABI.
- Return-value ownership is unknown.
- Memory preservation, global-state preservation, allocation, release, exceptions, and nonlocal control transfer are unknown.
- Separate IDs may resolve to the same target.
- The first target depends on inherited control-transfer state.

No application-level dependency roles are assigned.

## Acceptance examples

These are conditional specification examples, not executed tests.

1. **Initial word placement.** If F-D01.5 supplies index 3, the entry word is written at address 32, provided that address permits a 32-bit write.

2. **Selector addressing.** If the byte at entry base plus one is 200 and the primary value is 65, the first byte write targets address 217 and stores 4: the scaled value is 260, truncated to eight bits.

3. **Masking before truncation.** If the primary value is `0x3ff00000`, the masked right shift produces `0x3ff`; the byte destination receives `0xff`, not a ten-bit value.

4. **Fourth-input argument duplication.** With fourth input 3, F-D05.3 receives 3072 in both explicit argument positions.

5. **Zero write still reads memory.** If address 1 contains 7 at the zero-write step, address 24 receives zero. The operation is not a direct write to a fixed destination.

6. **Argument-order distinction.** If the retained secondary selector is 9 and the relevant result index is 6, F-D08.1 receives 9 first and 24 second.

7. **Distinct word addresses.** For index 2, the scaled-index write targets address 52, while the separate scaled-base read targets address 28. A consumer must not assume the read returns the value just written.

8. **Selector reread after dependencies.** If the final selector source initially contains 5, the first final byte write targets address 22. If an intervening dependency changes that source to 8, the later byte write targets address 25.

9. **Incomplete execution.** If a dependency does not return, later direct writes are not required to occur. The visible prefix does not guarantee completion.

## Uncertainties

The following remain **documented_only**:

- Intended application behavior and data structures.
- Reliability of the recovered control-flow sequence beyond the invalid-instruction warning.
- Actual ABI, instruction mode, and indirect-target interpretation.
- Provenance of the secondary value and inherited entry state.
- Exact semantics of signed overflow and signed left shifts.
- Endianness, alignment requirements, memory permissions, and volatile behavior.
- Relative ordering of the two argument reads supplied to F-D13.1.
- Dependency ownership, preservation, and failure behavior.
- Normal termination, return values, and any unrecovered continuation.

An interpretation involving packed fields or an instruction-encoding process is plausible from the shifts and masks, but remains inferred intent and is not part of the acceptance contract.

## Evidence

Established static evidence consists solely of the supplied recovered representation: four input slots, indirect calls, explicit memory-access widths, arithmetic expressions, local ordering, a recovered 64-bit result, and warnings identifying invalid instruction data and truncated control flow.

The dependency identifiers and descriptive value names in this contract are editorial labels, not recovered symbols or resolved identities.

No implementation access, executable inspection, runtime tests, or dependency validation has been performed. The acceptance examples state consequences of the visible operations under their stated conditions; they are not observed execution results.
