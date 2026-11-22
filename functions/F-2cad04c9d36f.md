<!-- Trusted provenance: F-2cad04c9d36f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2cad04c9d36f

## Purpose

F-2cad04c9d36f performs a long sequence of indirect invocations, derives scalar values and addresses from invocation results and ambient state, mutates calculated and fixed-location memory, and ends with an unresolved indirect transfer represented as an invocation.

No domain-level purpose is established. Constants resembling character values do not establish text output, formatting, serialization, or a string representation.

This contract distinguishes:

- **Established static evidence:** relationships, explicit access widths, conversions, and ordering visible in the supplied representation.
- **[INFERENCE]:** conclusions requiring assumptions such as wrapping arithmetic or stable ordinary memory.
- **documented_only:** unresolved behavior that cannot support an unconditional executable acceptance requirement.

The contract describes represented access events, not independently verified machine transactions. Repeated stores, transient values, and fresh reads remain significant even when final memory contents would be unchanged.

## Interface

### Explicit arguments

| Argument | Represented width | Established use |
|---|---:|---|
| First | 32 bits | Supplied unchanged as the first explicit operand of the initial invocation. |
| Second | 32 bits | No explicit use is shown. |
| Third | 32 bits | No explicit use is shown. |
| Fourth | Signed 32 bits | Shifted left by seven and supplied as the second explicit operand of the initial invocation. |

The absence of explicit uses for the second and third arguments does not establish their irrelevance under the actual calling convention. Indirect dependencies may observe incoming register or stack state not exposed as explicit operands.

### Additional inputs

Execution also depends on:

- Incoming link-related state used to construct the initial target.
- An ambient unsigned 32-bit value.
- Secondary register outputs associated with intermediate invocations.
- Results represented as 32-bit values and 64-bit containers.
- Fixed-location globals whose declarations are not supplied.
- Explicitly typed accesses through calculated addresses.
- An unresolved final transfer target.

The four explicit arguments do not define a complete standalone invocation protocol.

### Output and completion

No explicit return value is represented. A return follows the final indirect invocation.

Completion requires every preceding invocation and access to complete and the final transfer to resume at that return. No success flag, error code, returned pointer, or output length is established.

### Preconditions and ownership

Successful traversal requires compatible calling conventions, executable indirect targets, and accessible memory for every reached access. Values subsequently used as addresses must be valid in that role. Alignment, address mapping, and permitted low-address accesses depend on the unknown environment.

These are environmental preconditions, not validations performed by this function.

No allocation, release, reference counting, ownership transfer, or lifetime policy is established. Dependencies must not be assumed to preserve globals, retain arguments, or own or release referenced storage.

## Behavior

### Preparation and indirect invocations

Preparation is straight-line in the supplied representation: no explicit loop, conditional branch, or validation phase appears.

The initial indirect target is constructed from incoming link-related state plus 4094 bytes. Later targets are predominantly constructed from embedded continuation-related values plus the same displacement. This does not establish that the targets are equivalent, distinct, or implementations of a shared operation.

The opening invocation order is:

1. An invocation receiving the first argument and the fourth argument shifted left by seven.
2. Two invocations with no explicit operands shown.
3. An invocation receiving two explicit zeros.
4. Two further invocations with no explicit operands shown.
5. An invocation receiving the ambient unsigned value shifted left by thirteen, a secondary invocation-output value, and the ambient value shifted logically right by eleven.

“No explicit operands shown” does not establish a zero-argument signature.

Subsequent invocations receive combinations of retained results, secondary outputs, explicit zeros, masks, and shifts. Represented 64-bit containers supply low and high 32-bit halves to later operands. This establishes scalar extraction, not an in-memory layout or an independently recovered 64-bit return convention.

Several retained values are explicitly reset to zero. Later operands use those reset values, not previously extracted fields. Retention of scalar results across calls does not establish preservation of global memory by those calls.

### Widths and arithmetic

Explicit word accesses and word-sized scalar values are represented as 32 bits. Explicit byte and halfword accesses are 8 and 16 bits respectively.

Unsigned extractions have these mathematical bounds:

| Extraction | Range |
|---|---:|
| Low 24 bits, logically shifted right by four | 0–1,048,575 |
| Low 24 bits, logically shifted right by eight | 0–65,535 |
| Low 24 bits, logically shifted right by twelve | 0–4095 |
| Low 24 bits, logically shifted right by nineteen | 0–31 |
| Low 18 bits, logically shifted right by eleven | 0–127 |
| Low 17 bits, logically shifted right by four | 0–8191 |
| Low 16 bits, logically shifted right by five | 0–2047 |

Other arithmetic includes left shifts, multiplication by powers of two, byte-offset addition, and subtraction of 0x96.

**[INFERENCE]:** Under 32-bit wrapping machine arithmetic, overflowing high bits are discarded and address calculations are modulo \(2^{32}\). Under that model, successive left shifts by thirty-one and thirty produce zero, as do successive left shifts by fourteen and twenty-three. Such simplifications do not remove intervening invocation effects.

Wrapping behavior is not established as portable semantics for every signed expression.

### Explicit accesses versus undeclared globals

An explicit unsigned-byte access establishes an 8-bit read and subsequent zero-extension when converted to an unsigned 32-bit value.

A bare fixed-location global does not establish the same facts. Its declaration is absent. A receiving temporary’s width, an explicit conversion on the assigned expression, and the address encoded in a symbol do not establish the width or signedness of the underlying global access.

In particular:

- The global associated with 0x05 is read and converted to an unsigned 32-bit value. Neither an unsigned 8-bit load nor zero-extension is established. The converted value is subsequently used as an address base.
- The global associated with 0xa9 supplies a retained 32-bit value. This does not establish a 32-bit memory read.
- The global associated with 0x087fbbbc supplies a retained signed 32-bit address-like value. Its underlying access width and conversion behavior are also unresolved.
- Assigning an explicitly narrowed 8-bit expression to a global does not establish an 8-bit destination store.

These distinctions apply to every fixed-location global below.

### Opening calculated-memory mutations

The following conceptual quantities describe relationships, not object types:

- **High-position base:** a dependency result multiplied by \(2^{25}\).
- **Shifted-result base:** the low half of an earlier represented result shifted left by twenty-five.
- **Small index:** a dependency result shifted left by eight, masked to fifteen bits, then logically shifted right by six.
- **Doubled-index base:** twice the small index.
- **Final preparation result:** the last represented 64-bit preparation result.

Under wrapping arithmetic, both high-position bases are multiples of \(2^{25}\). The small index ranges from 0 through 508 in multiples of four; the doubled-index base ranges from 0 through 1016 in multiples of eight. No zero check is represented.

The opening access sequence establishes:

| Order | Explicit calculated-memory effect |
|---:|---|
| 1 | Store the shifted-result base as a word at the final result’s high half plus 0x14. |
| 2 | Store the low byte of that high half at shifted-result base plus 9. |
| 3 | Store the high-position base as a word at the final result’s low half multiplied by 256, plus 0x54. |
| 4 | Replace the byte at shifted-result base plus 9 with the shifted-result base’s low byte. |
| 5 | Store a zero word at high-position base plus 0x14. |
| 6 | Read an unsigned byte at high-position base plus 5. |
| 7 | Store a zero word at doubled-index base plus 0x14. |
| 8 | Store the zero-extended byte as a word at high-position base plus 0x34. |
| 9 | Read a word at high-position base plus 0x14. |

The two stores at shifted-result base plus 9 remain separate events.

**[INFERENCE]:** With stable ordinary memory and completed accesses, the final read in this table yields zero. Device behavior or asynchronous mutation can invalidate that readback conclusion.

### Cross-linked mutations and fixed-location inputs

Another zero word store at doubled-index base plus 0x14 precedes two separate byte stores at that base plus 0x15.

The global associated with 0x05 is then read and converted to an unsigned 32-bit value. This read precedes assignment to the global associated with 0x01. The latter assignment receives an explicitly narrowed 8-bit value derived from the doubled-index base; its destination width remains **documented_only**.

The converted 0x05 value supplies addresses at offsets 0x15 and 0x54. It is not established to lie within 0–255. A word is read at that derived base plus 0x54 before the destination is replaced. Call the retained word **the linkage value**.

Other established effects in this region include:

- A byte store at high-position base plus 0x11, a word store through the previously retained word from offset 0x14, and then a zero byte store at high-position base plus 0x11.
- Word loads at offset 0x74 followed by stores through loaded address-like values.
- Repeated self-valued word stores at offset 0x24 and repeated low-byte stores at offset 9.
- Replacements at doubled-index base plus 0x14, including a shifted value and the linkage value.
- Assignment of twice the linkage value to the global associated with 0x54.
- Assignment of an explicitly narrowed doubled-index-base value to the global associated with 0x15.

The two global assignments do not establish their memory-store widths.

Subsequent calculated accesses use fourfold and eightfold versions of the linkage value. Explicit unsigned-byte reads at offset 5 supply zero-extended address bases for word reads at offset 0x74. These explicit byte reads are distinct from the undeclared global read at 0x05.

Repeated reads are not automatically snapshots. Retained values, conversely, remain the earlier results even if later stores alter their source locations. Some expressions contain multiple reads without a separately established internal order; that internal order remains **documented_only**.

### Constants and repeated publication

A later word obtained through an explicit byte-derived address is used for:

- An 8-bit store of 0x3a through four times that word.
- A 16-bit store of 0x3a through eight times that word.
- A 32-bit store of 0x3a through four times that word plus 0x20.

Define the **adjusted quantity** as four times the linkage value minus 0x96, and the **adjusted base** as four times that adjusted quantity.

At adjusted base plus 0x20, four word stores occur in order: 0x20, 0x2f, a zero-extended byte read from adjusted base plus 4, and 0x20 again. The byte read occurs after the 0x2f store and before the byte-derived store.

Next, the global associated with 0x087fbbbc is read into the retained **publication base**. Assignment of 0x20 to the global associated with 0x60 follows. The first scaled publication store follows both events. Their underlying global-access widths are unresolved; their represented order is established.

Each publication group contains these explicit accesses, in order:

1. A word store at a scaled destination plus 0x54.
2. A byte store of the adjusted base’s low byte at publication base plus 0x19.
3. A byte store of the publication base’s low byte at adjusted base plus 0x11.
4. A word store of the scaled destination at publication base plus 0x34.

The first group uses eight times the linkage value as its scaled destination. Subsequent groups use eight times the adjusted quantity. The word value stored at offset 0x54 is retained across the groups.

No replacement read of the publication-base global is represented within this region. Numerous groups are explicitly expanded rather than controlled by a data-dependent loop. They must not be collapsed into one group. This contract does not certify their numeric cardinality.

### Tail processing and final transfer

After publication, the global associated with 0xa9 is read into a retained 32-bit value. A computed word read at offset 0x74 follows, before assignment to the global associated with 0x3a.

The 0x3a assignment uses an expression formed from eight times a character-converted adjusted quantity plus 0x30. Its underlying store width and destination conversion remain **documented_only**. Under wrapping low-byte interpretation, the expression’s low eight bits equal those of eight times the adjusted quantity plus 0x30.

Further tail effects include:

- A byte store of 0x6c at adjusted base plus 8.
- Byte stores derived from retained values.
- A self-valued word store at offset 0x54.
- Assignment to the global associated with 0x67 from an explicitly narrowed 8-bit version of the retained 0xa9-derived value.
- A subsequent read of the 0x67 global used for assignment to the 0x21 global.
- Additional, separate reads of the 0x67 global used for calculated byte stores.
- Word replacements at offsets 0x10, 0x44, 0x54, and 0x74.
- A 16-bit store of 0x65 through twice a loaded address-like word.
- A 32-bit store of decimal 100 at a computed base plus 0x54.
- Further word loads at offsets 0x64 and 0x74, followed by a final word store through twice a recently loaded value.

The widths and signedness of the 0x67 and 0x21 global accesses are unresolved. The calculated byte-store destinations remain explicitly 8-bit; conversion from the undeclared global values must not be assumed to be zero-extension.

One calculated byte store derives from a signed right-shift expression. The shift count is the low eight bits of the adjusted base and can exceed 31. Signed-shift behavior and handling of out-of-range counts remain **documented_only**.

The final target receives four explicit operands:

1. 0x72.
2. A retained word loaded at offset 0x64.
3. An earlier retained word loaded at offset 0x14.
4. The last retained word loaded at offset 0x74.

A later, separate offset-0x64 load contributes to the final stores, not to replacement of the second invocation operand. No subsequent validation or cleanup is represented.

## Side effects

### Memory footprints

Explicit calculated accesses establish access footprints, not complete structure layouts.

| Width | Representative offsets or destinations |
|---|---|
| 8 bits | 1, 5, 8, 9, 0x0d, 0x10, 0x11, 0x15, 0x19 |
| 16 bits | Scaled destinations receiving 0x3a or 0x65 |
| 32 bits | 4, 0x10, 0x14, 0x20, 0x24, 0x34, 0x40, 0x44, 0x54, 0x64, 0x74 |

Offset 0x10 participates in both byte and word writes. Calculated bases may overlap; disjoint typed fields are not established.

Fixed-location global references are separately characterized:

| Location | Represented role; underlying access width unresolved |
|---|---|
| 0x05 | Read, then convert to unsigned 32 bits for address calculations. |
| 0x01, 0x15 | Assign explicitly narrowed 8-bit expressions. |
| 0x54 | Assign a doubled retained value. |
| 0x60 | Assign 0x20 after reading the publication-base global. |
| 0x087fbbbc | Read into the retained signed 32-bit publication base. |
| 0xa9 | Read into a retained 32-bit value. |
| 0x3a | Assign the character-derived arithmetic expression. |
| 0x67 | Assign a narrowed expression; later read separately several times. |
| 0x21 | Assign a value read from the 0x67 global. |

This table does not certify byte or word footprints for those globals. Missing declarations also prevent certification of overlapping-global extents.

### Failure and partial state

No explicit bounds check, null check, rollback, error handler, or recovery path appears. A failing access or nonreturning dependency can leave earlier effects intact.

Atomicity, reentrancy, thread safety, endianness, alignment behavior, and determinism from the explicit arguments are not established.

## Dependencies

- **F-unresolved-001:** the initial indirect target.
- **F-unresolved-intermediate:** an opaque family label for intermediate indirect targets, not an assertion that they share an identity.
- **F-unresolved-final:** the unresolved final target.

Only F-2cad04c9d36f has a resolved identity in the supplied mapping.

No dependency contract establishes normal return, global preservation, memory ownership, allocation, release, or a failure encoding. Zero is not established as failure, nor nonzero as success.

## Acceptance examples

These are conditional static criteria. No runtime tests have been performed.

1. **Initial shift:** Under 32-bit wrapping shifts, fourth argument 1 supplies 128 as the initial invocation’s second explicit operand. Bit pattern 0x02000000 supplies zero.

2. **Extraction:** Retaining the low 24 bits of 0x12345678 and shifting logically right by eight yields 0x3456. Changing only the highest byte leaves this extraction unchanged, not necessarily the overall execution.

3. **Small-index boundary:** A contributing result with its low seven bits set produces small index 508 and doubled-index base 1016. Clearing those bits produces zero without a represented rejection.

4. **Explicit byte widening:** If the explicit unsigned-byte read at high-position base plus 5 yields 0xe7, the corresponding word store at offset 0x34 receives 231.

5. **Undeclared global conversion:** The same conclusion is not unconditional for the 0x05 global. If its missing declaration specifies an unsigned 8-bit value, 0xe7 converts to 231. If it specifies a signed 8-bit value containing −25, conversion to unsigned 32 bits yields 0xffffffe7. Neither declaration is established.

6. **Transient writes:** An access model must retain the four ordered word stores at adjusted base plus 0x20 and the intervening explicit byte read.

7. **Publication ordering:** The publication-base global read precedes assignment of 0x20 to the 0x60 global; the first scaled publication store follows. This criterion does not prescribe the widths of the two global accesses.

8. **Retained publication base:** Later publication addresses use the captured 32-bit publication-base value, not a new global read.

9. **Tail global ordering:** The 0xa9 global read precedes the computed offset-0x74 word read and the 0x3a global assignment. Receiving a 32-bit value does not require a 32-bit read at 0xa9.

10. **Partial failure and completion:** Earlier writes need not be restored after a fault. If execution reaches the final invocation, its first explicit operand is 0x72; no success return value is required.

## Uncertainties

The following remain **documented_only**:

- Complete calling convention and ambient-state requirements.
- Dependency identities, contracts, and secondary-output conventions.
- Actual meaning of represented 64-bit containers.
- Declarations, access widths, signedness, and conversion behavior of fixed-location globals.
- Signed overflow, signed right shifts, and excessive shift counts.
- Address mapping, alignment, endianness, device behavior, and concurrent mutation.
- Object boundaries and alias relationships.
- Expression-internal ordering of otherwise unsequenced reads.
- Certified numeric cardinality of publication groups.
- Domain meaning, ownership policies, and whether the final transfer returns.

These unknowns do not justify replacing represented accesses with safer conventional behavior.

## Evidence

The supplied representation supports the explicit argument uses, invocation ordering, masks, shifts, explicitly typed calculated-memory accesses, retained-value relationships, publication ordering, and final operands.

It does not supply declarations for the bare fixed-location globals. Consequently, scalar receiving widths and narrowed assignment expressions are distinguished from underlying memory-access widths throughout this contract.

Wrapping arithmetic and ordinary-memory readback are conditional inferences. No implementation access, dependency resolution, runtime execution, memory tracing, or behavioral testing has been performed.
