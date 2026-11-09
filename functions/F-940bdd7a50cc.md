<!-- Trusted provenance: F-940bdd7a50cc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-940bdd7a50cc

## Purpose

F-940bdd7a50cc initializes selected fields of a caller-supplied destination object, classifies a 32-bit input code, copies a four-word descriptor, and initializes embedded state through one or two dependencies.

The function has no explicit return value. Its observable result consists of destination writes, a byte-table read, and dependency calls. It is not a whole-object clearing operation: destination bytes not explicitly written remain unchanged unless a dependency modifies them.

The interpretation as an object initializer is inferred from the write pattern. The field meanings, object type, dispatch-boundary values, table contents, and dependency contracts are not established.

## Interface

The function accepts seven arguments, in order:

| Argument | Contract established by the supplied representation |
|---|---|
| Destination | Address of writable storage containing 32-bit fields, embedded regions, and a stored address. |
| Descriptor | Address of four readable 32-bit words, occupying 16 bytes. |
| First value | A 32-bit value copied to destination byte offset 0. |
| Second value | A 32-bit value copied to destination byte offset 4. |
| Input code | An unsigned 32-bit value used for classification and a separate 16-bit store. |
| Associated value | A 32-bit value copied to destination byte offset 144. |
| Optional-state selector | An integer tested only for equality with zero. Zero skips optional initialization; any nonzero value requests it. |

No explicit success indicator, error code, or returned address is provided.

### Representation and storage requirements

The layout below uses the four-byte word spacing shown by the supplied representation. Stored addresses are represented as four-byte fields. This supports a 32-bit address model; applicability to another address model is not established.

The destination must permit every direct access described below. On an ordinary execution in which the first dependency leaves the stored address unchanged, direct writes extend through byte offset 147 inclusive. This is a lower bound on required storage, not a complete object-size guarantee: either dependency may access additional storage under its own unknown contract.

The descriptor must support four 32-bit reads. The table must support a byte read at the selected index. Relevant fields must satisfy the target’s alignment requirements; unaligned-access behavior is not established.

The function contains no explicit null, bounds, alignment, or allocation checks. Valid storage and valid dependency inputs are caller preconditions, not conditions enforced by this function.

## Behavior

### Input-code classification

Classification first clears bit 13 of the 32-bit input code, using the mask `0xFFFFDFFF`. Every other bit remains unchanged.

The resulting 32-bit value is classified using two externally stored dispatch boundaries. Their contents are not supplied. Comparisons described as signed use the signed 32-bit interpretation of the masked value and the relevant boundary. Equality checks compare the represented values.

The decision has the following priority:

1. **Equality with the first dispatch boundary:** the normalized class is 16.
2. **Signed value below the first dispatch boundary:** the lower dispatch region applies:
   - Values 1, 2, and 4 retain those respective values as their class.
   - Values 8 and `0x64` produce class 8.
   - Value `0x65` produces class 16.
   - Every other value produces class 0.
3. **Signed value above the first dispatch boundary:** the upper dispatch region applies:
   - Equality with the second dispatch boundary produces class 16.
   - Below the second boundary in signed order:
     - Values `0x565` and `0x1444` produce class 16.
     - Value `0x1888` produces class 32.
     - Every other value produces class 0.
   - Above the second boundary in signed order:
     - Value `0xC422` produces class 16.
     - Value `0xD444` produces class 32.
     - Every other value produces class 0.

The `0x1444` case follows from the displayed subtraction-and-comparison condition. It is not a range of accepted values.

The complete set of possible normalized classes is therefore:

**0, 1, 2, 4, 8, 16, and 32.**

The boundary values are necessary to determine which numeric cases are reachable. For example, the visible `0x64` rule does not establish an unconditional mapping from `0x64` to 8: equality with a boundary or membership in another dispatch region can take precedence.

The high bit is preserved by masking. Consequently, inputs whose masked value has bit 31 set participate in dispatch as negative signed 32-bit values. Classification is not an unsigned range lookup.

### Destination layout

The function directly initializes the following fields:

| Byte offset | Width | Written value |
|---:|---:|---|
| 0 | 32 bits | First value argument |
| 4 | 32 bits | Second value argument |
| 8 | 32 bits | Normalized class |
| 12 | 32 bits | Zero-extended byte obtained from the class table |
| 16 | 16 bits | Low 16 bits of the original input code |
| 20 | 32 bits | First descriptor word |
| 24 | 32 bits | Second descriptor word |
| 28 | 32 bits | Third descriptor word |
| 32 | 32 bits | Fourth descriptor word |
| 36 | 32-bit address | Address of destination byte offset 40 |
| 40 | 32 bits | Zero |
| 144 | 32 bits | Associated value argument |

Only two bytes are directly written at offset 16. The function does not explicitly initialize bytes 18 and 19.

The 16-bit input-code store and the normalized-class store have distinct meanings. The former preserves bit 13 if it was present in the original input. The latter is computed after clearing that bit.

### Observable read and write order

The supplied representation establishes the following order. This describes explicit operations in the function; it does not assert atomicity or a concurrency memory-order contract.

1. The masked input is classified using the dispatch-boundary state.
2. The descriptor’s second, third, and fourth words are read.
3. The descriptor’s first word is read and written to destination offset 20.
4. The previously read descriptor words are written to offsets 24, 28, and 32, respectively.
5. The normalized class is written to destination offset 8.
6. The externally stored table base is read.
7. The first and second value arguments are written to destination offsets 0 and 4.
8. The low 16 bits of the original input code are written at destination offset 16.
9. One unsigned byte is read at the table base plus the normalized class. Its value is zero-extended and written as a 32-bit value at destination offset 12.
10. The associated value is written to destination offset 144.
11. The address of destination offset 40 is written to destination offset 36.
12. Zero is written as a 32-bit value at destination offset 40.
13. F-f991ac076e9f is called with the address of destination offset 44.
14. After that dependency returns, the address currently stored at destination offset 36 is read. A 32-bit zero is written 84 bytes beyond that address.
15. The optional-state selector determines the final operations, as described below.

The table-base read precedes several destination writes, but the selected table byte is read after those writes. This distinction matters if caller-supplied storage overlaps the table.

There is no general non-overlap check. Descriptor reads are completed before the descriptor-copy writes begin, so those writes do not themselves corrupt unread descriptor words. However, the evidence does not establish a general alias-safe contract for the descriptor, destination, dispatch-boundary storage, or table. Earlier writes can affect later reads when their storage overlaps.

### Embedded-state initialization

Before calling F-f991ac076e9f, the function establishes an embedded-state address at destination offset 36 and clears the first 32-bit word at the referenced region.

F-f991ac076e9f receives the address of destination offset 44, not the address of offset 40.

After that call, the function does **not** simply reuse the address originally placed at offset 36. It reads that field again. Therefore, the first post-call zero store is relative to the address present after the dependency returns.

If that stored address remains the address of destination offset 40, the post-call store at relative offset 84 lands at destination offset 124. If the dependency changes the stored address, the write follows the changed address instead.

### Optional-state handling

If the selector is zero:

- The function reads the address currently stored at destination offset 36.
- It writes a 32-bit zero 88 bytes beyond that address.
- It returns without calling F-8e156b91569c.

If the selector is nonzero:

- F-8e156b91569c is called with the address of destination offset 132.
- After that dependency returns, the function reads the address currently stored at destination offset 36.
- It writes the address of destination offset 132 into the four-byte field 88 bytes beyond the currently stored address.
- It returns.

If the address at offset 36 remains unchanged, the relative-offset-88 field is destination offset 128. Preservation of that address by either dependency is not established.

The stored address of destination offset 132 is an interior address into caller-supplied storage. Its presence does not establish allocation, ownership transfer, reference counting, or a release obligation.

## Side effects

Established direct effects are:

- Writes to the destination fields listed above.
- A two-byte write that preserves the rest of its containing four-byte slot.
- Reads of dispatch-boundary state and the table-base state.
- One byte read from the class table.
- A mandatory call to F-f991ac076e9f.
- A conditional call to F-8e156b91569c.
- Two post-initialization writes through the address stored at destination offset 36.

No direct allocation, deallocation, I/O, logging, or synchronization appears in the supplied function. Such effects remain possible inside dependencies.

The function performs no rollback. If a memory access fails or a dependency does not return normally, earlier writes may already be visible. No explicit recovery or failure result is shown.

Final field values are not guaranteed solely by the pre-call stores. Dependencies may modify destination storage or other state, and the function does not restore most earlier fields afterward.

## Dependencies

### F-f991ac076e9f

- Called once on every execution that reaches the initialization stage.
- Receives one address: destination byte offset 44.
- Runs after the embedded-state address is stored and the word at destination offset 40 is cleared.
- Runs before both relative post-initialization stores.
- Has no result explicitly consumed by this function.

Its storage extent, initialization semantics, global effects, ownership behavior, and failure behavior are **documented_only**. In particular, it must not be assumed to preserve destination offset 36.

### F-8e156b91569c

- Called once if and only if the selector is nonzero and execution reaches the optional stage.
- Receives one address: destination byte offset 132.
- Runs after the relative-offset-84 zero store.
- Runs before the relative-offset-88 address store.
- Has no result explicitly consumed by this function.

Its storage extent, initialization semantics, global effects, ownership behavior, and failure behavior are **documented_only**. The destination address field is reread after this call rather than assumed unchanged.

### External data dependencies

The function also depends on:

- A first dispatch boundary.
- A second dispatch boundary.
- A table-base value.
- Readable table bytes at the selected class indices.

The table is indexed in bytes, not 32-bit elements. A table byte of `0xFF`, for example, becomes the 32-bit value 255, not a negative value.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence. They are not reports of executed tests.

1. **Lower-region retained class.**  
   Given a masked input of 4 that lies below the first boundary in signed order, the class at destination offset 8 is 4. The table read uses byte index 4.

2. **Lower-region alias.**  
   Given a masked input of `0x64` in the lower dispatch region, the stored class is 8 and the table read uses byte index 8.

3. **Bit-13 separation.**  
   Inputs `0x0064` and `0x2064` have identical masked values. With identical boundary state, they select the same class. Their 16-bit stores at destination offset 16 differ: `0x0064` versus `0x2064`.

4. **Boundary precedence.**  
   If the masked input equals the first dispatch boundary, class 16 is selected even if that numeric input otherwise appears in a named lower-region case.

5. **Intermediate-region classification.**  
   If masked input `0x1444` is above the first boundary and below the second boundary in signed order, class 16 is selected. Under the same region conditions, `0x1888` selects class 32.

6. **Upper-region classification.**  
   If masked input `0xD444` is above both boundaries in signed order, class 32 is selected. This example does not claim the same result for different boundary values.

7. **Unsigned table-byte expansion.**  
   If the selected table byte is `0xFE`, the function writes the 32-bit value 254 at destination offset 12.

8. **Zero selector with preserved embedded address.**  
   If F-f991ac076e9f returns normally without changing destination offset 36, the function clears destination offsets 124 and 128. F-8e156b91569c is not called.

9. **Nonzero selector with preserved embedded address.**  
   If both dependencies return normally and preserve destination offset 36, the function clears destination offset 124, calls F-8e156b91569c with destination offset 132, and stores that interior address at destination offset 128. Negative selectors take this same branch.

10. **Changed embedded address.**  
    If F-f991ac076e9f changes the address at destination offset 36 to another valid writable region, the next zero store targets relative offset 84 of that region. The function does not force the original embedded address back into the field.

11. **Partial input-code field write.**  
    If bytes 18 and 19 initially contain sentinel data and neither dependency nor an overlapping access modifies them, those bytes remain unchanged. The function’s direct input-code store covers only bytes 16 and 17.

## Uncertainties

The following remain **documented_only**:

- Actual dispatch-boundary values and their signed ordering.
- Table contents, length, and semantic interpretation.
- Semantic meanings of the normalized classes and copied descriptor.
- Full destination size and dependency-accessed extents.
- Dependency effects on globals, destination fields, and memory ownership.
- Whether intended callers permit any overlapping storage.
- Target byte order and alignment-fault behavior.
- Fault propagation, exceptional exits, and dependency non-return behavior.
- Thread safety, reentrancy, and concurrent mutation of external state.

No successful-result postcondition should assume dependency preservation beyond what an independently established dependency contract provides.

## Evidence

**Established static evidence:** the supplied function representation shows the 32-bit mask, signed dispatch comparisons, finite normalized-class outputs, four-word descriptor copy, destination offsets, two-byte input-code store, unsigned byte-table lookup, dependency-call order, selector branch, and post-call rereads of the stored address.

**Inferred intent:** the routine appears to initialize an object containing embedded state and optionally link another embedded component. This interpretation does not establish field names, ownership, or dependency semantics.

**Verification limit:** no runtime tests have been performed. The acceptance examples are static, conditional obligations; they are not observed executions.
