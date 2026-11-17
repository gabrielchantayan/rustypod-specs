<!-- Trusted provenance: F-e4740a5fd1aa; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e4740a5fd1aa

## Purpose

F-e4740a5fd1aa performs an ordered series of indirect interactions, derives addresses and values from their results, modifies memory at several widths, and reaches a final unresolved indirect transfer.

No application-level purpose is established. Repeated field offsets and character-like constants do not establish formatting, serialization, allocation, or a conventional object model.

This contract separates:

- **Established static evidence:** represented inputs, arithmetic dependencies, accesses, widths, and ordering.
- **Conditional consequences:** conclusions requiring the stated arithmetic and memory models.
- **Inferred intent:** no higher-level interpretation is adopted.
- **documented_only:** properties not resolved by the supplied representation.

No runtime tests have been performed.

## Interface

### Explicit inputs

The represented interface has four positional inputs and no declared result.

| Position | Represented width | Established use |
|---|---:|---|
| First | 32 bits, otherwise untyped | No explicit use |
| Second | 32 bits, otherwise untyped | No explicit use |
| Third | Signed 32-bit integer | Shifted left four positions for an early interaction |
| Fourth | Signed 32-bit integer | Shifted left 28 positions for that interaction, then transformed further |

None is established as a pointer, size, handle, or owned resource. Lack of explicit use does not establish that unresolved calling-convention state hides an input from dependencies.

### Implicit inputs and outputs

Execution also depends on incoming control-flow state, indirect targets, primary and secondary interaction results, existing memory, fixed-address storage, and a separately stored address captured during the memory stage.

Secondary result words have no independently established initialization or production contract. They must not be replaced with zero merely because their origin is unresolved.

There is no represented success flag or translated error result. Outputs include dependency effects, direct memory mutations, and the final transfer. That transfer receives four represented words:

1. 0x72.
2. A retained word captured late from the argument-bearing address plus 0x64.
3. A retained word captured earlier from the earlier object plus 0x14.
4. A retained word captured late from the argument-bearing address plus 0x74.

The argument-bearing address is loaded from the earlier object plus four. Captured arguments are not refreshed by later memory changes.

If the final interaction returns, the represented function returns without an explicit result. Actual transfer and return semantics remain **documented_only**.

### Preconditions and ownership

Continuation requires executable targets and accessible storage for every reached interaction, load, and store. No corresponding validation appears.

There is no established single buffer, minimum allocation size, or set of disjoint objects. Addresses arise from interaction results, scaled words, loaded words, widened bytes, and fixed storage, including low absolute addresses.

Ownership, allocation boundaries, alignment requirements, and resource lifetimes remain **documented_only**. No dependency is assumed to preserve globals, retain memory, or release memory.

## Behavior

### Arithmetic and sequencing model

The represented path has an indirect-interaction stage, a direct-memory stage, and the final transfer. No explicit branch or loop appears. Every later effect depends on preceding interactions and accesses completing; a dependency may fault, branch internally, or never return.

A word is 32 bits, a halfword 16 bits, and a byte eight bits. Pair-sized results are represented as 64-bit quantities. High-word extraction uses an unsigned shift by 32. Other extractions shift a full 64-bit quantity by 34, 36, or 44 before narrowing; they are not oversized word shifts.

Conditional on fixed-width 32-bit machine arithmetic:

- Left shifts, multiplication, and address arithmetic retain the low 32 bits.
- Unsigned right shifts introduce zeros.
- Byte and halfword stores retain the low eight and 16 bits.
- Unsigned byte loads widen to values from zero through 255.
- Shifting the fourth input left 28 retains its low nibble; shifting that result left another four produces zero.
- A word shifted left eight and then right eight unsigned retains its original low 24 bits.
- A 24-bit mask before a left shift by six excludes original bits 24 and 25 that an unrestricted word shift could retain.
- A 22-bit masked quantity shifted left seven or eight fits within an unsigned word.
- Two successive word-width left shifts by 28 produce zero.

These are not portable guarantees for signed arithmetic. Negative signed left operands, signed overflow, address overflow, and narrowing require target rules.

A late signed-word right shift uses the derived base’s low eight bits as its count. Negative-operand semantics and counts of at least 32 remain **documented_only**.

### Indirect interactions

The entry target depends on incoming control-flow state plus displacement 0xFFE. Subsequent targets use represented code-location values plus that displacement. No validated dispatch table is established.

An early interaction receives two explicit zeros. After intervening interactions, another receives the third input shifted left four, an unresolved secondary result, and the fourth input shifted left 28.

Later interactions receive shifted results, repeated equal arguments, low and high result words, masked fields, and explicit zeros. Interactions without explicit arguments may still have implicit inputs.

Their represented order remains material, including interactions with unused primary results and interactions whose argument arithmetic conditionally reduces to zero. No interaction is established as pure.

### Address landmarks

The following names identify retained values and address derivations, not validated objects.

| Landmark | Definition |
|---|---|
| Primary-origin word | Interaction-produced word used to form the primary base |
| Primary base | Primary-origin word multiplied by 2²⁵ |
| Compact component | Another result shifted left eight, masked to 15 bits, then shifted right six unsigned |
| Compact base | Twice the compact component |
| Shifted token | Separate interaction-produced word shifted left 25 |
| Saved shifted word | Another interaction-produced word shifted left eight |
| Scalar-derived base | Word conversion of the scalar read at absolute address 0x05 |
| Retained root | Word captured from scalar-derived base plus 0x54 before replacement |
| Fourfold root | Four times the retained root |
| Eightfold root | Eight times the retained root |
| Adjusted root | Fourfold root minus 0x96 |
| Derived base | Four times the adjusted root |
| Linked word | Later word loaded through an unsigned-byte-derived base at offset 0x74 |
| Earlier object | Twice the linked word |
| Payload base | Four times the linked word |
| Stored base | Separately stored address captured immediately before the absolute 0x60 word write |

Under fixed-width arithmetic, the compact base is a multiple of eight between zero and 1016. The primary base depends only on the primary-origin word’s low seven bits. Neither property establishes address validity.

Offsets are byte offsets. Word fields include offsets four, 0x10, 0x14, 0x20, 0x24, 0x34, 0x40, 0x44, 0x54, 0x64, and 0x74. Byte accesses overlap some word fields. Offset 0x15 lies within the word beginning at offset 0x14 regardless of byte order.

### Initial mutations and the zero read-back invariant

The last pair-sized result before direct stores supplies a high word used as an address. Its offset-0x14 word receives the shifted token.

The shifted-token offset-nine byte receives the high result word’s low byte. Before that byte is overwritten with the shifted token’s low byte, the word at 256 times the low result word plus 0x54 receives the primary base.

The function then clears primary-base offset 0x14, reads an unsigned byte at primary base plus five, clears compact-base offset 0x14, and stores the widened byte as a word at primary-base offset 0x34. It next reads primary-base offset 0x14.

**Under ordinary sequential byte-addressed memory semantics, that read-back is zero.** Between the clear and read-back:

- The compact-base store writes only zero bits, even if its range overlaps the cleared word.
- The widened-byte word store occupies primary-base offsets 0x34–0x37, disjoint from offsets 0x14–0x17.
- The two ranges remain disjoint under modulo-2³² address arithmetic.
- No dependency call intervenes.

Ordinary address-range aliasing among these represented stores cannot explain a nonzero read-back.

Another compact-base offset-0x14 clear precedes two byte stores at compact-base offset 0x15 using the retained read-back’s low byte. Both therefore write zero under the same memory assumptions.

The scalar at absolute address 0x05 is captured next; its access width is unresolved. Absolute byte 0x01 receives the compact base’s low byte, and scalar-derived offset 0x15 receives the compact component’s low byte.

The saved shifted word supplies a byte store at primary-base offset 0x11 and a word store at read-back plus 0x54. The primary-base byte is then cleared. Under ordinary memory semantics, the word destination is **absolute address 0x54**, because the retained read-back is zero. Under fixed-width arithmetic, the saved shifted word’s low byte is also zero.

Finally, the retained root is captured from scalar-derived base plus 0x54 before that field receives the retained read-back—zero under ordinary memory semantics. Earlier overlapping effects can influence the captured root, but later writes cannot change that retained value.

A departure from the zero read-back invariant would require unresolved memory attributes, external mutation, or inaccurate recovery of the represented operations; the intervening stores themselves do not provide such a departure.

### Root-dependent mutations and absolute writes

Compact-base offset 0x34 receives the compact component. A word captured from primary-base offset 0x74 supplies a destination whose offset 0x54 receives the primary-origin word shifted left 26.

A nested assignment uses compact-base offset 0x74 both to obtain a destination and to obtain the value written at that destination plus 0x54. Internal evaluation order and physical load multiplicity remain **documented_only**; atomicity is not established.

Retained-root offset 0x24 receives the retained root twice. Interleaved byte stores place its low byte at the primary-offset-0x74-derived destination plus nine; that destination’s offset 0x54 also receives the saved shifted word.

Two absolute stores have specific placement constraints:

- After compact-base offset 0x14 receives the primary-origin word shifted left 26, absolute word 0x54 receives twice the retained root. Two further compact-base offset-0x14 stores then receive the retained root.
- Two compact-base offset-0x15 byte stores receive the fourfold root’s low byte. Absolute byte 0x15 then receives the compact base’s low byte, before the saved shifted word reaches fourfold-root offset 0x54.

The later absolute 0x54 store is distinct from the earlier saved-shifted-word store established by the zero read-back.

### Dependent links

Immediately following the saved-shifted-word store, the represented access order is:

1. Compact-base offset 0x34 receives the fourfold root as a word.
2. A fresh compact-base offset-0x74 word is captured and doubled to obtain a retained temporary base.
3. Fourfold-root offset 0x54 receives the fourfold root.
4. Fourfold-root offset 0x15 receives the eightfold root’s low byte.
5. Retained-root offset 0x44 receives the retained temporary base.

The capture precedes the replacement even if their address ranges overlap. The replacement and intervening byte store do not refresh the retained temporary.

An unsigned byte at fourfold root plus five supplies the base for an offset-0x74 word load. The resulting linked destination’s offset 0x44 receives the retained temporary.

Subsequent mutations write the fourfold root’s low byte at fourfold-root offset 0x15, the linked destination’s low byte at fourfold-root offset 0x0D, and the fourfold root at its offsets 0x14 and 0x54.

A word loaded from fourfold-root offset 0x64 supplies another destination. Its offset 0x0D receives the linked destination’s low byte; its offset 0x14 receives its own base.

A separate word captured from fourfold root plus four supplies the base for two unsigned byte reads at offset five. Before the first read, temporary-base offset nine receives the linked destination’s low byte and temporary-base offset 0x44 receives the eightfold root.

Between the two byte reads, fourfold-root offset nine receives the temporary base’s low byte, the linked destination’s offset 0x44 receives the first widened byte, fourfold-root offset 0x14 receives the eightfold root, and temporary-base offsets 0x15 and 0x44 receive respectively the captured base’s low byte and the fourfold root. Aliasing can therefore make the two byte reads differ.

After the second byte capture, fourfold-root offset 0x15 receives the eightfold root’s low byte. The captured byte supplies the base for the offset-0x74 load defining the linked word. Further stores replace fourfold-root offset 0x15 with the captured base’s low byte, offset 0x54 with the fourfold root, offset nine with the earlier object’s low byte, and offset 0x14 with the eightfold root.

### Constant stores and repeated region

The constants are byte 0x3A at the payload base, halfword 0x003A at eight times the linked word, and word 0x0000003A at payload base plus 0x20.

Derived-base offset 0x20 receives four ordered word stores: 0x20, 0x2F, a widened byte freshly read from derived base plus four, and 0x20 again. Matching first and last values do not erase intermediate accesses.

The separately stored address is then captured as the stored base. Only afterward does absolute word 0x60 receive 0x20. Earlier effects can influence this capture; it is not an entry-time snapshot.

Each repeated group has four ordered effects:

| Position | Effect |
|---|---|
| First | Scaled-base offset 0x54 receives the payload base as a word |
| Second | Stored-base offset 0x19 receives the derived base’s low byte |
| Third | Derived-base offset 0x11 receives the stored base’s low byte |
| Fourth | Stored-base offset 0x34 receives the scaled base as a word |

The first scaled base is eight times the retained root. Subsequent groups use eight times the adjusted root. These expressions can evaluate equally: retained root 50 makes both scaled bases 400.

The repeated accesses are not interchangeable with one final-state update. Exact group multiplicity is **documented_only** in this contract.

### Late captures and fixed-byte dependencies

After the repeated region, absolute word 0xA9 is captured, followed by payload-base offset 0x74. Absolute byte 0x3A then receives the low eight bits of eight times the adjusted root plus 0x30.

The third transfer argument is captured from earlier-object offset 0x14 **after** that absolute-byte write. Derived-base offset four is captured next.

Earlier-object offset 0x0D receives the derived base’s low byte. Derived-base offsets eight, 0x0D, and nine receive 0x6C, the stored base’s low byte, and the stored base’s low byte. Earlier-object offset 0x54 receives its own base.

Absolute byte 0x67 receives the captured 0xA9 word’s low byte. Earlier-object offset 0x44 is then read before a fresh byte-0x67 read supplies absolute byte 0x21.

The offset-0x44 capture supplies a byte at derived-base offset 0x10, a word at stored-base offset 0x54, and—after the variable-count signed shift—a byte at earlier object plus one. Stored-base offset 0x54 is then replaced with the derived base.

A word is captured from derived-base offset 0x40. Before its word store at derived-base offset 0x10, stored-base offset 0x74 receives the **previously captured derived-base offset-four word**. Stored-base offset 0x44 then receives the earlier object’s base.

Two later derived-base offset-nine byte stores each freshly read absolute byte 0x67. Between those reads, earlier-object offset nine receives the derived base’s low byte, derived-base offset 0x44 receives the captured 0xA9 word, and a destination is captured from derived-base offset 0x54. Neither read is specified to reuse the value previously written to 0x21.

### Final mutations and transfer

The destination captured through derived-base offset 0x54 receives the retained payload-offset-0x74 word’s low byte at offset 0x0D. Earlier-object offset 0x54 receives the retained third argument.

A word is captured from derived-base offset 0x14. Subsequent writes place the retained payload word’s low byte at earlier-object offset 0x11, and the captured destination’s low byte and full word at derived-base offsets nine and 0x44.

The argument-bearing address is loaded from earlier object plus four. Halfword 0x0065 is written at twice that address **before** the second argument is captured from argument-bearing offset 0x64.

Before capturing the fourth argument, the destination reached through derived-base offset 0x54 receives the retained derived-offset-0x14 word’s low byte at offset 0x11 and word 100 at offset 0x54.

After the fourth argument is captured from argument-bearing offset 0x74, terminal effects still occur: another destination is loaded from earlier-object offset 0x64; earlier-object offset nine receives the argument-bearing base’s low byte; the new destination’s offset 0x54 receives the retained second argument; and the word at twice the second argument receives the new destination base.

The final transfer follows. No rollback or cleanup path is represented.

## Side effects

Direct stores have eight-, 16-, and 32-bit widths. Fixed-address activity includes:

- A scalar read at 0x05 and a word read at 0xA9.
- Byte writes at 0x01, 0x15, 0x21, 0x3A, and 0x67.
- Word writes at 0x54 and 0x60.
- Three represented byte reads at 0x67.

Under ordinary sequential memory semantics, the read-back-derived saved-word destination is also absolute 0x54. Subsequent aliases can overwrite it.

Computed fields, fixed storage, and dependency-accessible memory need not be disjoint. Nevertheless, aliasing must be assessed for the particular ranges and values involved; it does not invalidate the initial zero read-back proof.

No atomicity, synchronization, barriers, or thread-safety guarantee is established. Represented ordering does not establish external visibility of every intermediate store.

No explicit allocation, release, bounds check, null check, permission check, or translated failure result appears. Faults or non-return may leave only a prefix of effects completed.

## Dependencies

F-e4740a5fd1aa is the only mapped opaque function ID supplied. No dependency IDs are available for the indirect targets; their identities remain **documented_only**, rather than receiving invented identifiers.

| Interaction class | Established relationship | Unresolved properties |
|---|---|---|
| Entry interaction | Incoming control-flow-derived target | Identity, signature, preservation |
| Intermediate interactions | Ordered zero, shifted, masked, and result-derived arguments | Effects, implicit inputs, ownership |
| Word-result interactions | Results drive arithmetic and addresses | Meaning, storage validity |
| Pair-result interactions | Represented 64-bit results supply halves and fields | Actual result convention |
| Final transfer | Receives 0x72 and three retained words | Dispatch, errors, return behavior |

Dependencies may alter globals and memory. Allocation, retention, ownership transfer, and release are not inferred.

## Acceptance examples

These are conditional static criteria, not executed tests. Each assumes preceding execution completes and relevant storage is accessible.

1. **Early arguments.** Third input 1 and fourth input 2 produce 0x10 and 0x20000000 in the relevant interaction’s first and third positions under fixed-width arithmetic. Its second argument remains secondary-result-derived.

2. **Zero despite overlap.** If primary and compact bases are both zero, both offset-0x14 clears address 0x14. Even if the byte at address five is 0xE7, its widened store addresses 0x34–0x37. The read-back at 0x14 is zero under ordinary sequential memory semantics.

3. **Zero-derived effects.** Under those memory assumptions, both subsequent compact-offset-0x15 byte stores write zero, the saved shifted word is stored at absolute 0x54, and the scalar-derived offset-0x54 replacement writes zero. A later overwrite does not change these earlier effects.

4. **Later absolute 0x54 placement.** Retained root 50 makes the later absolute-word store write 100, after the compact-offset-0x14 shifted-origin store and before its next two root stores.

5. **Capture before aliasing replacement.** With retained root 50 and compact base 0xA8, fourfold-root offset 0x54 and compact-base offset 0x74 both address 0x11C. If that word holds 0x300 when captured, the temporary is 0x600. The subsequent replacement with 200 must not retroactively make it 400.

6. **Eightfold byte.** Retained root 50 requires byte 0x90 at the dependent-link transition’s fourfold-root offset 0x15, before the retained-temporary store. Substituting 0xC8 at this occurrence is incorrect.

7. **Equal repeated derivations.** Retained root 50 makes first and subsequent scaled bases both 400. Distinct formulas do not require distinct addresses.

8. **Late and fresh reads.** An overlapping absolute-0x3A write can affect the subsequently captured third argument. Intervening writes to 0x67 can affect either later fresh byte read. Conversely, the stored-offset-0x74 store uses the retained derived-offset-four capture.

9. **Terminal effects.** Capturing the fourth argument does not skip terminal mutations. An invalid terminal destination need not produce a status, successful return, or subsequent transfer.

## Uncertainties

The following remain **documented_only**:

- Application purpose and consumer meaning.
- Indirect dependency identities, signatures, preservation, and ownership effects.
- Function-boundary and control-flow recovery accuracy.
- Secondary-result and pair-result conventions.
- Exact repeated-group multiplicity and nested-expression physical load multiplicity.
- Scalar access width at absolute 0x05.
- Byte order, alignment handling, memory attributes, and external visibility.
- Target-specific signed arithmetic, overflow, and variable-shift behavior.
- External mutation, concurrency, fault recovery, and partial-progress reporting.
- Final-transfer return behavior.

The initial read-back is zero under ordinary sequential memory semantics. Only unresolved memory attributes, external mutation, or recovery accuracy qualify that conclusion—not ordinary overlap among the represented intervening stores.

## Evidence

The contract derives from static examination of the supplied representation, not comments, prior-document authority, or feedback assertions.

The zero read-back follows from the clear, the intervening zero-only store, the disjoint widened-byte store, and the absence of an intervening dependency call. Its downstream consequences follow from retaining that zero as both a value and an address component.

Other established evidence includes mixed-width overlap, absolute-write placement, capture-before-replacement ordering, the intervening eightfold-root byte store, late stored-address capture, potentially equal repeated-base derivations, fresh 0x67 reads, retained transfer arguments, and terminal mutations before transfer.

Dependency implementations and runtime realization are unavailable. No runtime tests have been performed.
