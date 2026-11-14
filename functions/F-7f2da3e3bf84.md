<!-- Trusted provenance: F-7f2da3e3bf84; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7f2da3e3bf84

## Purpose

F-7f2da3e3bf84 is an externally stateful routine represented as a straight-line sequence of indirect calls, calculated memory accesses, and global-state updates.

The supplied representation establishes bit extraction, index scaling, byte-table indirection, word-to-byte transfers, and several word stores. It does not establish a reliable application-level purpose. In particular, the evidence does not justify describing the routine as a constructor, serializer, dispatcher, memory manager, or hardware initialization operation.

This contract distinguishes:

- **Established static evidence:** operations and ordering explicitly present in the supplied representation.
- **Inferred interpretation:** a likely 32-bit execution model used to explain address arithmetic and value widths.
- **documented_only unknowns:** dependency behavior, register conventions, address validity, and terminal behavior that cannot be established from the supplied material.

No runtime tests have been performed.

## Interface

### Explicit inputs and result

The represented interface has three 32-bit inputs and no declared return value.

- **First input:** no explicit use appears in the represented body.
- **Second input:** no explicit use appears in the represented body.
- **Third input:** interpreted as an unsigned 32-bit value for a logical right shift by 23. The resulting 9-bit value is written to a calculated word location.

The absence of explicit reads of the first two inputs does not establish that the overall execution is independent of them. Indirect calls have unresolved calling conventions and may observe incoming machine state.

### Additional incoming state

The routine also depends on state not represented as explicit parameters:

- An incoming return-address-related value used to calculate the first indirect call target.
- An incoming 32-bit register value used in shifts, address calculations, and a sign-based selector.
- Several secondary-register values exposed after indirect calls, including one eventually used as a byte pointer.
- A writable global word, designated **G1** in this contract.
- A global word, designated **G2**, whose contents are later captured and used as a memory base.
- A three-byte global quantity, designated **G3**, used in the final reconstruction of G1.
- Memory reachable through all calculated addresses and dependency-provided values.

G1, G2, and G3 are contract labels, not source symbol names. They do not imply disjoint storage.

### Width and address conventions

The following notation describes memory accesses without implying a data structure:

- A **byte read** retrieves an unsigned 8-bit value.
- A **word read or write** accesses 32 bits.
- A **halfword write** accesses 16 bits.
- A **low-byte transfer** keeps only bits 0–7 of a value.
- Address offsets are measured in bytes.

The representation strongly suggests 32-bit registers and addresses. Logical shifts of unsigned words and low-byte truncation are explicit. Exact machine behavior for overflowing signed multiplication, signed left shifts, invalid pointers, and unaligned accesses remains documented_only.

There is no established object layout, allocation size, ownership transfer, or buffer-length argument.

## Behavior

### Execution model and dependency boundaries

The visible body contains no conditional branch or loop. Its operations form one ordered path, provided every preceding call returns and every preceding memory access succeeds.

Numerous indirect calls occur between the operations below. These calls are observable parts of the routine, not removable padding. They may change memory or global state, fail to return, or affect subsequent secondary-register values.

Dependency identifiers below name unresolved call sites or explicitly identified sets of call sites. They do not assert that different sites invoke different functions, or that repeated-looking sites invoke the same function.

### 1. Initial call sequence and first word update

The routine first invokes three indirect dependencies in sequence. The first target is calculated from incoming return-address-related state; subsequent targets use fixed represented anchors plus offsets.

After these calls, it writes a word at:

**four times a secondary-register value, plus 20 bytes.**

The stored value is the incoming ambient word logically shifted right by 22. This produces a value from 0 through 1,023.

A following indirect call receives the secondary-register value used by the represented expression. Further calls produce and forward an opaque 32-bit result, then obtain a word index for the next update.

The relationship between secondary-register values and conventional function return values is not established.

### 2. Third-input extraction and table-word read

Using the next dependency-produced index, the routine writes a word at:

**four times that index, plus 20 bytes.**

The stored value is the third explicit input logically shifted right by 23, yielding a value from 0 through 511.

It then reads a word from the address obtained by:

1. Keeping the low 29 bits of a secondary-register value.
2. Shifting that masked value right by 19.
3. Adding 20 bytes.

The resulting read address lies between 20 and 1,043 inclusive. The expression does not scale the extracted index by four and therefore does not guarantee word alignment.

The read result is retained across subsequent calls. A dependency receives four times the earlier word index, and several more calls execute.

### 3. First two byte-table updates

The retained word is logically shifted right by 22. Call this extracted value **E**; its range is 0 through 1,023.

The routine reads a byte at the incoming ambient-word address plus one. Treating that byte as an unsigned value, it adds 17 and writes the low byte of E to the resulting address.

Consequently:

- The byte-table-selected destination lies between 17 and 272 inclusive.
- Only eight bits of the ten-bit E are stored.
- The destination need not be related to the source byte’s address.

After additional calls, including one receiving an explicitly zero shifted value, the routine reads a byte at **four times E plus one**. It adds 17 to that byte and writes zero to the selected destination.

The source address for this second selector lies between 1 and 4,093 inclusive.

### 4. Sign-selected zero writes

Following additional calls, including a call receiving two copies of a dependency-produced value shifted left by two, the routine captures another unsigned 32-bit dependency result.

After forwarding that result and making another call, it performs a zero-byte write selected through a byte read at address 1 or address 2:

- Address 1 is selected when the incoming ambient word has its sign bit clear.
- Address 2 is selected when that sign bit is set.

This interpretation depends on the represented signed right shift being arithmetic.

The selected byte is interpreted as unsigned, 17 is added, and zero is written there.

A later dependency receives three copies of a secondary-register value. That value is also retained for subsequent left-shifted arguments. Another call receives an opaque result and the previously captured unsigned word shifted right by 23.

After further calls, the routine repeats the sign-selected byte read and zero write. This is a fresh byte read, not reuse of the earlier selection. Its unsigned byte result is retained as **B** for later address calculations.

The two sign-selected writes can therefore target different locations if intervening calls or aliasing change the selector byte.

### 5. First global update and byte refreshes

After more calls, including another duplicate-argument call, a dependency returns a 32-bit word. The routine writes that word to G1 before forwarding it to another dependency.

Following the next call, it reads a word at **B plus 116**, truncates the word to its low byte, and writes that byte at **B plus one**.

A dependency then receives the retained secondary-register value shifted left by 11. A subsequent dependency produces a represented 64-bit result. The next call receives, in order:

1. The low 32-bit half.
2. The high 32-bit half.
3. The word previously written to G1, logically shifted right by 23.

After further calls, a new dependency result is multiplied by four to form a base **P**. The routine reads a word at P plus 116 and writes its low byte at P plus one. It then passes P to another dependency.

Several calls later, a different result is shifted left by two and retained as **T**. Before using T in the next call, the routine repeats the word read at P plus 116 and the byte write at P plus one.

These two refreshes are distinct accesses separated by calls. The second read is not established to equal the first.

The following dependency receives four arguments:

1. The retained secondary-register value shifted left by 12.
2. T.
3. T again.
4. The result from which T was formed, shifted left by 18.

### 6. Payload capture and propagation

After a chain of calls and opaque-result forwarding, a dependency provides an index. Four times this index forms a new base **Q**.

The routine reads a word at Q plus 116. Call the captured word **V**. It writes the low byte of V at Q plus one, then passes Q to a dependency.

After more calls, it writes the complete V to a word address formed as **16 times a new dependency result, plus 20**.

A following dependency receives a transformed version of T: logical right shift by two, followed by left shift by 16. Under 32-bit bit-vector arithmetic, this retains only a subset of T’s bits; it is not generally equivalent to recovering and preserving the original producer’s full value.

After further calls, another word write stores V at **four times a new dependency result, plus 20**.

### 7. Scaled-value and zero stores

A later dependency result is shifted left by two. The shifted value is written as a word at **T with its low two bits cleared, plus 20**.

The following dependency receives two copies of that shifted value.

After additional calls, the routine writes a zero word at **four times another dependency result, plus 20**. That scaled base is then passed to a dependency.

A subsequent dependency returns a represented 64-bit result. The routine:

- Shifts the low half left by two.
- Writes that shifted word at **four times the high half, plus 20**.
- Reads G2 and retains the captured value.
- Calls a dependency with the shifted low half, the unscaled high half, and two more copies of the shifted low half.

After further calls, it writes the same shifted low-half value at **four times a new dependency result, plus 20**.

Only then does it read a word at **the captured G2 value plus 20**. Call this word **W**.

The base used for this read is the earlier G2 snapshot. Intervening changes to G2 do not select a new base in the represented sequence.

### 8. Final byte-table updates

After another call sequence, a dependency returns a represented 64-bit result. Retain its low half as **L** and its high half as **H**.

The routine reads a byte at **V logically shifted right by 23, plus one**. It adds 17 to the unsigned byte and writes the low byte of W to the selected destination.

A dependency then receives H three times, followed by L shifted left by two.

After additional calls, the routine performs a second byte-table update:

- Keep the low 30 bits of L.
- Shift that value right by 21.
- Add one and read a byte there.
- Add 17 to that unsigned byte.
- Store the low byte of W logically shifted right by 22 at the resulting destination.

The first selector-read address lies between 1 and 512. The second also lies between 1 and 512. The stored portion of W in the second update is bits 22–29; bits 30–31 are discarded by byte truncation.

Further calls receive a scaled result and, later, two copies of another scaled result.

### 9. Terminal writes and null-target call

Near the end, a dependency supplies an opaque word that is forwarded to another call. A following dependency supplies a byte.

The routine then performs these operations in order:

1. Writes zero through a secondary-register-derived byte pointer.
2. Updates G1 with the supplied byte as its low byte and G3 as its upper three bytes.
3. Writes a zero halfword at **the low 30 bits of L, logically shifted right by seven**.
4. Invokes an indirect target whose represented address is zero, passing three zero arguments.

The halfword destination lies between zero and 8,388,607 inclusive and is not guaranteed to be aligned.

A return is represented after the zero-target call. Successful return is conditional on that call returning; ordinary null-call failure is not handled within the visible body.

## Side effects

The routine directly performs byte, halfword, and word writes, plus two updates to G1. It also performs multiple word and byte reads and a read of G2.

No rollback, validation, bounds check, alignment check, or explicit error-reporting path is visible.

The ordering above is significant because:

- Selector bytes can be modified by earlier stores or dependencies.
- Repeated reads may observe changed memory.
- Global symbols may overlap other storage.
- Calls occur between related accesses.
- A fault or nonreturning dependency can leave a partially updated state.

There is no established allocation, release, reference-count adjustment, or ownership transfer. Dependency behavior must not be treated as preserving globals, preserving pointed-to memory, or releasing memory.

## Dependencies

All dependency behavior is **documented_only** beyond the represented arguments, result widths, and call order.

For reference, the unresolved dependency sites are assigned these opaque identifiers:

| ID | Established role |
|---|---|
| F-D001 | First indirect call, target derived from incoming return-address-related state. |
| F-D002 | Remaining unresolved call sites not individually designated below; this denotes a set of sites, not one shared callee. |
| F-D003 | Produces the word stored in the first update to G1. |
| F-D004 | Produces the first represented 64-bit result, later forwarded as two halves with a shifted third argument. |
| F-D005 | Receives the four shifted arguments involving the retained secondary-register value and T. |
| F-D006 | Produces the represented 64-bit result whose halves determine a scaled store and its destination. |
| F-D007 | Produces the final represented 64-bit result whose low half is retained as L. |
| F-D008 | Produces the byte used in the final update to G1. |
| F-D009 | Terminal zero-address indirect call with three zero arguments. |

These identifiers are local contract references, not resolved implementation identities. The available private dependency map identifies only F-7f2da3e3bf84 itself.

## Acceptance examples

These examples are conditional observations, not executed tests. Each assumes execution reaches the relevant operation, dependencies return the stipulated values, and all accesses are supported.

1. **Third-input extraction:** A third input of 8,388,608 produces a stored word of 1 in the second direct word update. An input of 4,294,967,295 produces 511.

2. **Ten-bit extraction followed by byte truncation:** If the word retained for E is all ones, E is 1,023. The first byte-table update stores 255, while the next selector is read at address 4,093.

3. **Fresh sign selector:** For an ambient word with its sign bit clear, both sign-selected operations read address 1. If the first read returns 9 and the later read returns 12, the zero writes target addresses 26 and 29 respectively.

4. **Word-to-byte refresh:** If a refresh word read returns the numeric value 305,419,896, the corresponding byte write stores 120. This is numeric truncation, not an assertion about the memory byte order of the source word.

5. **Captured global base:** If G2 is captured as 4,096 and later changed by a dependency, the W read still uses address 4,116.

6. **Final extracted byte:** If W is all ones, both final byte-table stores write 255, even though one uses W directly and the other uses W shifted right by 22.

7. **Terminal failure:** If execution reaches F-D009 and address zero is not a valid callable target, completion is not established. Earlier writes are not undone by any visible recovery path.

## Uncertainties

The following remain **documented_only**:

- Whether the representation accurately separates instructions from embedded data.
- Exact calling conventions and secondary-register result lifetimes.
- Whether indirect targets are valid, returning, recursive, or state-mutating.
- Exact address width, overflow semantics, alignment requirements, and byte order.
- Validity of low-address table accesses and the zero-address call.
- Storage overlap among globals and calculated memory locations.
- Timing and aliasing of the G3 read during final G1 reconstruction.
- Any application-level invariants, ownership rules, or intended error behavior.

The repeated call pattern and unusual low-address accesses warrant caution; they do not independently establish corruption, obfuscation, or intended functionality.

## Evidence

This contract is based solely on the supplied static representation of F-7f2da3e3bf84.

Established evidence consists of the explicit interface, memory-access widths, arithmetic masks and shifts, represented call order, global accesses, and terminal zero-target call. Semantic names and dependency identifiers in this document are descriptive labels only.

No implementation lookup, dependency inspection, runtime execution, or tests were performed.
