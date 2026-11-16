<!-- Trusted provenance: F-5e3eb0fa0a5c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5e3eb0fa0a5c

## Purpose

F-5e3eb0fa0a5c performs an ordered sequence of indirect invocations, derives numeric and address-like quantities from their results, updates memory using several access widths, and reaches a final unresolved indirect transfer.

No domain-level purpose is established. Character-valued constants, repeated stores, and address-like words do not establish formatting, serialization, allocation, or record maintenance.

This contract distinguishes:

- **Established static evidence:** represented calls, arithmetic, memory accesses, retained observations, and ordering.
- **[INFERENCE]:** possible interpretations that are not acceptance requirements.
- **documented_only:** unresolved semantics or explicitly identified coverage limitations.

All effects are conditional on execution reaching them. A dependency may not return; an access may fail.

## Interface

### Explicit inputs

| Argument | Represented width | Established use |
|---|---:|---|
| First | 32 bits | No explicit use appears. |
| Second | 32 bits | No explicit use appears. |
| Third | Signed 32 bits | Shifted left five positions and supplied to two early invocations. |

None is established as a buffer, length, handle, or ownership-bearing reference. Incomplete invocation signatures and secondary register results prevent concluding that the first two arguments are irrelevant to the complete machine-level interface.

### Implicit inputs and preconditions

Execution depends on incoming control state, dependency results, secondary register values, fixed-location storage, and computed-address memory.

Each reached invocation requires a suitable executable target. Each reached memory access requires storage supporting its width and direction. No explicit checks establish non-null addresses, alignment, object bounds, permissions, or valid targets.

Direct accesses are 8, 16, or 32 bits wide. Some dependency results are represented as 64-bit quantities with low and high 32-bit components. Computed addresses derive from 32-bit quantities; their target-address-space interpretation is **documented_only**.

### Outputs and completion

No explicit return value is established. Outputs are dependency interactions, memory effects, and the final transfer.

The final transfer has four represented arguments:

1. Constant 0x72.
2. A late-loaded 32-bit word.
3. A retained word previously read from record base plus 0x14.
4. A retained word read from new base plus 0x74.

No success flag, error code, output length, or allocation result is established. A return follows the represented transfer, but whether that transfer returns normally is **documented_only**.

## Behavior

### Dependency interactions and arithmetic

The initial invocation phase is a fixed-order sequence without a visible loop or branch-selected alternative. Its first target derives from incoming control state. Later targets generally derive from embedded address-like values displaced by 0xFFE.

Calls without displayed arguments remain observable interactions. Unused results do not imply purity. Similar argument lists do not imply identical dependencies.

Early calls receive a primary result shifted left eight positions and the third input shifted left five. Another receives a paired result’s low component shifted left eight, its high component, and the scaled third input.

Subsequent calls combine primary results, retained results, secondary register values, masks, shifts, and explicit zeros. Some retained quantities survive several intervening invocations. No dependency is assumed to preserve globals, reachable memory, or incompletely represented machine state.

For a 64-bit result, the low component contains bits 0–31 and the high component bits 32–63. Logical full-result shifts by 34, 36, and 44 positions correspond to logical shifts of the high component by 2, 4, and 12. These are not oversized shifts of a 32-bit operand.

Explicit unsigned operations follow their represented widths. Numeric byte reads zero-extend; byte and halfword writes retain the low eight and sixteen bits. Signed overflow, signed left shifts, address arithmetic, and exceptional variable shift counts require target semantics where the representation does not resolve them.

### Retained quantities and arithmetic constraints

The following descriptive names denote numeric quantities, not validated objects or source identifiers.

| Quantity | Definition |
|---|---|
| Extracted index | A dependency result shifted left eight, masked to its low fifteen bits, then logically shifted right six. |
| Index base | Twice the extracted index. |
| Initial scaled base | A separate dependency result multiplied by \(2^{25}\). |
| Saved shifted result | Another dependency result shifted left eight. |
| Zero-read value | A word observed at initial scaled base plus 0x14 after an explicit zero store there. |
| Small base | The zero-extended byte observed at absolute 0x05. |
| Saved link | A word observed at small base plus 0x54. |
| Selected value | A later word selected through a fresh byte observation. |
| Adjusted quantity | Four times the saved link minus decimal 150. |
| Working base | Four times the adjusted quantity. |
| Global base | The word captured from absolute 0x087FBBBC. |
| Record base | Twice the selected value. |

Under 32-bit bit-vector evaluation, the extracted index is a multiple of four from 0 through 508. Index base is consequently a multiple of eight from 0 through 1016.

The adjusted and working quantities are not independent inputs. Under either defined signed arithmetic or 32-bit wrapping arithmetic:

- Adjusted quantity is congruent to 2 modulo 4.
- Working base is equivalently sixteen times the saved link minus 600.
- Working base’s low byte is congruent to 8 modulo 16.
- Therefore its low byte belongs to 0x08, 0x18, …, 0xF8.

These constraints apply to the examples below. Unresolved arithmetic semantics do not establish otherwise unreachable scenarios.

### Entry into direct memory updates

The final paired dependency result supplies the first destinations. A retained derived word is stored at its high component plus 0x14. The high component’s low byte is stored at another retained derived base plus 9. Initial scaled base is stored as a word at 256 times the final low component plus 0x54. The offset-9 byte is then overwritten with that retained base’s low byte.

Initial scaled base receives a zero word at offset 0x14, followed by a byte read at offset 5. Index base receives a zero word at offset 0x14. The observed byte is stored, zero-extended, as a word at initial scaled base plus 0x34.

A word read at initial scaled base plus 0x14 supplies the zero-read value. This read precedes another zero-word store at index base plus 0x14. Index base’s offset-0x15 byte then receives the zero-read value’s low byte twice.

The preceding zero store alone does not guarantee a zero observation: aliasing, coupled storage, concurrent changes, and device-backed memory are unresolved. No null check protects subsequent use of the observed word as a base.

### Early fixed-location and linked accesses

A byte read at absolute 0x05 supplies small base. A byte write at absolute 0x01 then carries index base’s low byte. The captured small base, restricted to 0–255, is reused without reloading 0x05.

Small base plus 0x15 receives the extracted index’s low byte. Initial scaled base plus 0x11 receives the saved shifted result’s low byte. The full saved shifted result is stored at zero-read value plus 0x54, after which initial scaled base plus 0x11 is cleared.

Only then is saved link read from small base plus 0x54. That location is overwritten with zero-read value, and index base plus 0x34 receives the extracted index.

A word from initial scaled base plus 0x74 selects another base. Its offset-0x54 word first receives a dependency-derived quantity scaled by \(2^{26}\), and later receives saved shifted result. Its offset-9 byte receives saved link’s low byte twice. Saved link plus 0x24 receives saved link itself twice.

Index base plus 0x74 is separately read to select a destination and obtain a payload for another offset-0x54 store. Their equality requires suitable stable-memory assumptions; their relative evaluation order within that individual assignment is not established.

Index base plus 0x14 receives the dependency-derived \(2^{26}\)-scaled quantity. A 32-bit store at absolute 0x54 then carries saved link shifted left one. Index base plus 0x14 subsequently receives saved link twice.

Index base plus 0x15 receives the low byte of four times saved link twice. Absolute 0x15 then receives index base’s low byte.

Thus the identifiable early absolute-access order is: byte read at 0x05, byte write at 0x01, word write at 0x54, and byte write at 0x15. Computed accesses intervene.

### Intermediate computed updates

This region uses four-times and eight-times forms of saved link, words loaded at offsets 4, 0x64, and 0x74, and zero-extended byte-derived selectors.

It includes word stores at offsets 0x14, 0x34, 0x44, and 0x54; byte stores at offsets 9, 0x0D, and 0x15; repeated stores; and stores of a base value through that same base. Intermediate occurrences remain observable even when payloads repeat.

A fresh word from index base plus 0x74 is doubled to form another base. A byte read at four times saved link plus 5 selects a word at the byte’s numeric value plus 0x74. That selected word participates in offset-0x44 and byte updates.

A later word read at four times saved link plus 4 supplies a different base. Its offset-5 byte is observed twice with intervening writes. The second observation selects the word designated selected value. The first observation cannot replace the second.

Four times selected value receives byte 0x3A; eight times selected value receives halfword 0x003A; four times selected value plus 0x20 receives word 0x0000003A.

Working base plus 0x20 receives four successive word payloads: 0x20, 0x2F, a zero-extended byte freshly read from working base plus 4, and 0x20. The byte read occurs after the 0x2F store and before its own word store.

### Captured global base and repeated region

After those four word stores, the function reads global base from absolute 0x087FBBBC. Absolute 0x60 then receives word 0x20. No global-base reload is represented, even if that write affects overlapping or coupled storage.

The repeated region’s word payload is four times selected value. Its initial group performs, in order:

- A payload-word store at eight times saved link plus 0x54.
- A byte store of working base’s low byte at global base plus 0x19.
- A byte store of global base’s low byte at working base plus 0x11.
- A word store of eight times saved link at global base plus 0x34.

Subsequent groups retain this ordering but substitute eight times adjusted quantity for eight times saved link in the first destination and final payload.

These are explicitly repeated groups, not a conditional loop. Their exact cardinality is not enumerated here: a **documented_only coverage limitation**, not evidence of runtime-variable repetition. Collapsing them into one final-state update would discard represented accesses.

### Late fixed locations and fresh observations

After the repeated region, a word read at absolute 0xA9 is retained. A word read at four times selected value plus 0x74 follows.

Absolute 0x3A then receives the low byte of eight times adjusted quantity plus 0x30. Under the arithmetic interpretations stated above, this payload is congruent to 0x40 modulo 0x20.

The function retains words from record base plus 0x14 and working base plus 4. It writes working base’s low byte at record base plus 0x0D; byte 0x6C at working base plus 8; global base’s low byte at working base plus 0x0D and plus 9; and record base itself as a word at record base plus 0x54.

Absolute 0x67 receives the retained 0xA9 word’s low byte. A word read at record base plus 0x44 occurs before a fresh byte read at 0x67 supplies the byte written to absolute 0x21.

The offset-0x44 word’s low byte is written at working base plus 0x10, and the full word at global base plus 0x54. A signed-right-shift-derived byte is written at record base plus 1. Its operand is the observed offset-0x44 word; its count is working base’s low byte.

Under the established congruence constraint, possible counts are 8, 24, 40, …, 248. Only 8 and 24 are below 32. The representation does not establish masking, saturation, or another rule for larger counts.

Global base plus 0x54 is overwritten with working base. A word read at working base plus 0x40 precedes stores of the saved offset-4 word at global base plus 0x74, the freshly read offset-0x40 word at working base plus 0x10, and record base at global base plus 0x44.

A second fresh 0x67 read supplies a byte store at working base plus 9. Record base plus 9 then receives working base’s low byte, and working base plus 0x44 receives the full retained 0xA9 word.

A word read at working base plus 0x54 supplies late destination base. A third fresh 0x67 read supplies another byte store at working base plus 9. These three reads are distinct observations; unconditional equality is not established.

### Finalization and transfer

Late destination base plus 0x0D receives the low byte of the retained selected-value-derived offset-0x74 word.

Record base plus 0x54 receives the saved word from record base plus 0x14. This is not an established restoration: no original offset-0x54 word was saved, and an earlier explicit store there carried record base itself.

A fresh word is then read at working base plus 0x14. Record base plus 0x11 receives the selected-value-derived word’s low byte. Working base plus 9 receives late destination base’s low byte, and working base plus 0x44 receives that full base.

A word read at record base plus 4 supplies new base. Twice new base receives halfword 0x0065. A word read at new base plus 0x64 supplies the second transfer argument.

Late destination base plus 0x11 receives the fresh working-base offset-0x14 word’s low byte; its offset-0x54 word receives decimal 100.

The final ordering is:

1. Read new base plus 0x74, retaining the fourth transfer argument.
2. Read record base plus 0x64, retaining final destination.
3. Write new base’s low byte at record base plus 9.
4. Write the second transfer argument as a word at final destination plus 0x54.
5. Write final destination as a word at twice the second argument.
6. Invoke F-unresolved-003 with the four interface arguments.

The third transfer argument remains the saved record-base offset-0x14 observation, not a fresh read.

## Side effects

Identifiable absolute accesses involve 0x05, 0x01, 0x54, 0x15, 0x087FBBBC, 0x60, 0xA9, 0x3A, 0x67, and 0x21.

Computed offsets include 1, 4, 5, 8, 9, 0x0D, 0x10, 0x11, 0x14, 0x15, 0x19, 0x20, 0x24, 0x34, 0x40, 0x44, 0x54, 0x64, and 0x74. These do not establish structure layouts.

Different-width accesses may overlap. A byte at base plus 0x15 overlaps a word beginning at base plus 0x14; endianness determines the affected bits. Later stores may overwrite earlier byte effects without erasing their occurrence.

No explicit allocation, release, ownership transfer, lock, atomic operation, or barrier is established. Dependencies may perform such actions.

There is no represented rollback, retry, bounds recovery, or explicit error reporting. Failure may leave partial effects. Atomicity, thread safety, reentrancy, and idempotence are not promised.

## Dependencies

| Opaque ID | Role | Unresolved properties |
|---|---|---|
| F-unresolved-001 | Initial incoming-control-state-derived invocation | Target, implicit arguments, results, effects |
| F-unresolved-002 | Remaining indirect invocation family | Individual identities, signatures, secondary results, effects |
| F-unresolved-003 | Final indirect transfer | Target, transfer semantics, effects, return behavior |

F-unresolved-002 denotes a family, not one shared function. No dependency is assumed to preserve globals, own returned memory, release supplied memory, or return normally.

## Acceptance examples

These are conditional static consequences, not executed tests. They assume the stated observations occur and execution reaches the described accesses.

### Narrow fields and fixed writes

An extracted index of 0x104 gives index base 0x208. The absolute byte writes at 0x01 and later 0x15 each carry 0x08; intervening accesses remain required.

For saved link 0x1234, where the represented signed shift is defined, the word store at absolute 0x54 carries 0x2468.

### Reachable adjusted quantity

Saved link 38 gives adjusted quantity \(4 \times 38 - 150 = 2\), and working base 8. The absolute 0x3A byte payload is \(8 \times 2 + 0x30 = 0x40\).

That write follows the absolute 0xA9 word read and the selected-value-derived offset-0x74 word read. Adjusted quantity 5 is not reachable under defined signed or 32-bit wrapping arithmetic.

### Captured and fresh values

If absolute 0x087FBBBC reads as 0x2000, the repeated region uses that captured global base without a represented reload after the 0x60 store.

If the retained 0xA9 word ends in 0xAB, the initial 0x67 store carries 0xAB. If intervening aliasing or environmental effects change 0x67 to 0xCD, a later fresh read may supply 0xCD. Stable, nonaliasing ordinary memory may yield 0xAB at all three observations.

### Assignment, not restoration

If record base is 0x2000 and its offset-0x14 word is observed as 0x12345678, the earlier explicit offset-0x54 store carries 0x00002000. The later assignment carries the retained 0x12345678, regardless of original offset-0x54 contents. The third transfer argument also carries 0x12345678.

### Reachable distinct offset-9 payloads

Saved link 42 gives adjusted quantity 18 and working base 72, or 0x48. The earlier record-base offset-9 byte store therefore carries 0x48.

If the later new-base observation is 0x123456A7, the finalization store at that same offset carries 0xA7. It follows the new-base offset-0x74 and record-base offset-0x64 reads, and precedes both final word stores.

Those stores may subsequently overlap the byte; persistence of 0xA7 is not unconditional. Working-base low byte 0x40 is excluded by the arithmetic constraints above.

### Width boundaries

A numeric byte observation of 0xE7 widens to 0x000000E7. A halfword store of 0x0065 affects two bytes, not four; byte order is target-dependent.

If the final transfer is reached, its first represented argument is 0x72 and its third remains the retained offset-0x14 observation.

## Uncertainties

The following remain **documented_only**:

- Domain purpose and complete machine-level interface.
- Dependency identities, calling conventions, secondary register provenance, and failure behavior.
- Signed-overflow and signed-shift semantics, address interpretation, alignment, and endianness.
- Variable signed-right-shift behavior, particularly for reachable counts at least 32.
- Memory mappings, alias relationships, object boundaries, concurrency, and device effects.
- Exact repeated-region cardinality as an enumerated requirement of this contract.
- Ownership, allocation, release, and final-transfer return behavior.

Arithmetic uncertainty does not remove the demonstrated congruence constraints under defined signed or 32-bit wrapping interpretations.

**[INFERENCE]** Packed-data manipulation or record maintenance are possible interpretations only; neither is an acceptance requirement.

## Evidence

The supplied static representation supports the described widths, retained observations, arithmetic relationships, memory effects, and final four-argument transfer.

The adjusted quantity derives from four times saved link minus 150, and working base from four times that result. These relationships establish the reachable corrected examples and their modulo constraints under the stated arithmetic interpretations.

The final region separately identifies the saved offset-0x14 payload, the earlier record-base payload at offset 0x54, and the late new-base-derived offset-9 byte write.

Prior prose, comments, and reviewer assertions are not independent evidence. No dependency implementations, validated calling convention, target memory map, or execution trace were supplied.

**No runtime tests have been performed.**
