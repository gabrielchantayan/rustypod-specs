<!-- Trusted provenance: F-e1a1acc0e3f5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e1a1acc0e3f5

## Purpose

F-e1a1acc0e3f5 performs an ordered sequence of indirect invocations and integer transformations, followed by fixed-location and computed-address memory mutations and a final unresolved indirect transfer.

No application-level purpose is established. Character-like constants do not establish text generation; self-valued stores do not establish construction of a valid object or linked structure.

This contract separates:

- **Established static evidence:** represented operations, widths, value dependencies, and ordering.
- **[INFERENCE]:** conclusions conditional on a stated target-machine model.
- **documented_only:** behavior not sufficiently established to impose an executable acceptance requirement.

The supplied representation is evidence about behavior, not proof that every represented operation is reachable or faithfully captures the underlying execution.

## Interface

### Inputs and outputs

The represented interface declares no parameters and no return value. It exposes no explicit buffer, length, status result, or error object.

Execution nevertheless depends on implicit inputs:

- Incoming link state, which determines the first indirect target.
- Other incoming register state used in early shifts and argument preparation.
- Primary, combined, and secondary results of indirect invocations.
- Existing contents of fixed and computed memory locations.
- A later fixed-location address-base capture.
- The unresolved final transfer target.

Consequently, this is not established to be an ordinary parameterless operation independent of entry state.

### Widths and arithmetic

Explicit computed-memory accesses distinguish 8-bit bytes, 16-bit halfwords, and 32-bit words. Address-bearing integer calculations are represented at 32 bits. Combined invocation results are represented at 64 bits and frequently split into low and high 32-bit components.

Unsigned byte-to-word conversions zero-extend. Explicit one-byte truncations retain eight bits; halfword stores retain sixteen bits. Character conversions require additional qualification where signedness or out-of-range conversion matters.

**[INFERENCE — target-word model]:** If the representation describes conventional 32-bit machine arithmetic, additions, multiplications, and left shifts retain the low 32 bits at each intermediate step. This model must not be silently substituted for portable signed-arithmetic semantics: several expressions use signed operands, and overflow behavior is not independently established.

Unsigned 64-bit right shifts by 34, 36, and 44 extract the high word shifted logically by 2, 4, and 12 respectively. They are not excessive shifts of an already narrowed word.

Global declarations are absent. A 32-bit receiving value or assignment expression does not, by itself, establish the physical access width of an undeclared fixed-location symbol. The fixed-location contract below distinguishes represented value widths from that limitation.

### Preconditions

Reaching the endpoint requires executable indirect targets, returning dependencies wherever execution continues, valid memory mappings for every access, and target arithmetic and calling-convention semantics compatible with the represented operations.

No explicit validation enforces these conditions. Byte order, alignment requirements, object boundaries, and the distinction between ordinary memory and device-visible storage are unknown.

## Behavior

### Execution structure and invocations

The visible body has an invocation-heavy prefix, a direct-memory phase, and a final indirect transfer. No explicit conditional branch, bounded processing loop, or early success return appears. Dependencies may branch, mutate memory, fault, or fail to return.

The first invocation target is incoming link state plus 0xFFE. Later targets predominantly derive from embedded code-related values plus the same displacement. Represented argument lists contain zero through four explicit values. An empty argument list does not prove that a target consumes no implicit register state.

Argument preparation includes shifted primary results, high words of combined results, explicit zeros, equal arguments in multiple positions, saved earlier results, and secondary-register outputs without an established ordinary return signature. Invocation ordering remains significant even when arguments simplify to zero. No dependency is established to be pure or globally preserving.

Masks select fields of several widths, including 24, 22, 20, 19, 17, 14, 13, 12, and 11 bits. Their placement relative to shifts matters. A 20-bit mask followed by a logical shift right by 10 yields at most 1023; a 24-bit mask followed by a shift right by 20 yields at most 15. Successive shifts retain intermediate truncation under the target-word model.

### Descriptive address values

The following names describe value provenance, not recovered variables or object types:

| Descriptive value | Provenance |
|---|---|
| Final result | Last combined invocation result before direct stores |
| Scaled base | Earlier primary result multiplied by 0x02000000 |
| Shifted capture | Another earlier primary result shifted left by 8 |
| Carried field | Saved low result shifted left by 25 |
| Small index | A primary result shifted left by 8, masked with 0x7FFF, then logically shifted right by 6 |
| Small base | Twice the small index |
| Saved word | Word captured at offset 0x54 from the base obtained through the fixed-location 0x5 read |
| Quadrupled saved base | Four times the saved word |
| Adjusted address | Quadrupled saved base minus 0x96 |
| Adjusted scaled base | Four times the adjusted address |
| Fixed-loaded base | Later address-base capture from fixed location 0x087FBBBC |

Under the target-word model, the small index is a multiple of four from 0 through 508; the small base is a multiple of eight from 0 through 1016. No underflow or range check guards the adjusted address.

All these regions may alias each other and fixed locations.

### Initial memory effects

The opening stores associate the carried field with the final result’s high component and the scaled base with the final result’s low component scaled by 0x100. In order:

- A word at high-component offset 0x14 receives the carried field.
- Carried-field offset 9 receives the high component’s low character value.
- A word at scaled-low-component offset 0x54 receives the scaled base.
- The same carried-field byte destination receives the carried field’s low character value.

The function clears scaled-base offset 0x14, captures the byte at scaled-base offset 5, clears small-base offset 0x14, and writes the captured byte zero-extended at scaled-base offset 0x34. It then captures the word at scaled-base offset 0x14, clears small-base offset 0x14 again, and performs two identical byte stores at small-base offset 0x15 using that captured word.

These reads observe memory at their represented positions. Earlier stores may affect them through overlap.

### Fixed-location access contract

The following accesses are individually significant. Their ordering includes surrounding computed accesses; none denotes an independent abstract global.

| Fixed location | Value interaction and position |
|---|---|
| 0x5 | Read after the initial pair of small-base-offset-0x15 byte stores. Its value is converted to an unsigned 32-bit address-bearing value. The supplied declarations do not establish the read width or whether widening is from a byte. |
| 0x1 | Receives the low eight bits of the small base immediately after the 0x5 capture. A computed byte write at captured-base offset 0x15 follows, using the small index’s character value. |
| 0x54 | Receives the saved word shifted left by one. This word-valued assignment follows a small-base-offset-0x14 store derived from the earlier scaled-base-producing result shifted left by 26, and precedes two word stores of the saved word to that same small-base offset. The global declaration needed to independently establish transaction width is absent. |
| 0x15 | Receives the low eight bits of the small base after two byte stores of the quadrupled saved base’s character value to small-base offset 0x15. A word store of the shifted capture to quadrupled-saved-base offset 0x54 follows. |
| 0x087FBBBC | Read into a signed 32-bit address-bearing capture after the adjusted-scaled-base-offset-0x20 overwrite sequence. This read precedes the fixed-location 0x60 assignment. The source declaration of the fixed symbol is absent, so physical read width is not independently established. |
| 0x60 | Receives the word-valued constant 0x20 after that address-base capture and before the repeated store region. The global declaration is absent. |
| 0xA9 | Read into a 32-bit capture immediately after the repeated region. It is represented as a fixed-word capture, but its global declaration is absent. |
| 0x3A | Receives a character-valued result after the next computed offset-0x74 word read, and before the following computed offset-0x14 and offset-4 word reads. The value comes from character-converting the adjusted address, multiplying by eight, and adding 0x30. |
| 0x67 | Receives the low eight bits of the 0xA9 capture. Later accesses read this location afresh: once for 0x21, then separately for each of two computed-offset-9 byte stores. |
| 0x21 | Receives the value read from 0x67 after an intervening computed-offset-0x44 word read. It does not receive an unconditional copy directly from the original 0xA9 capture. |

Locations 0x1, 0x15, 0x3A, 0x67, and 0x21 are represented as byte locations through their assignments and uses. Their missing declarations still limit conclusions about generated transactions or volatile-access properties.

The 0x3A arithmetic includes a character conversion *before* multiplication. Character signedness, out-of-range conversion, and assignment conversion are target-dependent. Under eight-bit low-bit-preserving conversion and byte assignment, the resulting byte is the low eight bits of eight times the adjusted address plus 0x30. That conditional simplification does not remove the memory write.

### Intermediate computed-address mutations

After the 0x5 and 0x1 accesses, the function writes the shifted capture at scaled-base offset 0x11 and through the previously captured scaled-base-offset-0x14 word at offset 0x54. It then clears the former byte destination.

The saved word is captured from the 0x5-derived base’s offset 0x54 before that location is overwritten with the earlier offset-0x14 capture. Small-base offset 0x34 receives the small index.

Subsequent stores use words loaded from offsets 4, 0x44, 0x54, 0x64, and 0x74 to select later destinations. Some word bases come from unsigned byte reads followed by an offset-0x74 word read. There is no visible validity check.

The region includes repeated writes to offsets 0x14 and 0x54; self-valued stores at offsets 0x14, 0x24, and 0x54; and byte writes at offsets 9, 0x0D, 0x11, and 0x15. These establish access patterns, not structural invariants.

Two reads from small-base offset 0x74 occur within a single represented store expression, one selecting the destination base and one supplying the stored value. Their relative evaluation order is not established. They must not automatically be treated as one immutable capture.

### Constants and repeated stores

A later nested byte-derived word capture determines three stores of 0x3A: a byte at four times the capture, a halfword at eight times the capture, and a word at four times the capture plus 0x20.

The adjusted-scaled-base-offset-0x20 word destination then receives, in order, 0x20, 0x2F, an unsigned byte read from adjusted-scaled-base offset 4 widened to a word, and 0x20 again. The fixed-loaded-base read and 0x60 assignment follow.

The repeated region associates scaled saved-word addresses with the fixed-loaded base. Each group contains a word store at a scaled address’s offset 0x54, a byte store at fixed-loaded-base offset 0x19, a byte store at adjusted-scaled-base offset 0x11, and a word store at fixed-loaded-base offset 0x34.

The first group uses eight times the saved word. Subsequent groups use eight times the adjusted address. The byte values are captured low bytes of the adjusted scaled base and fixed-loaded base respectively. The repeated groups do not visibly advance a traversal pointer.

This contract does not quantify the region’s multiplicity. That omission is not permission to collapse the represented repetitions into one group.

### Final mutation phase

For this phase, the **traversal capture** is the later nested byte-derived word described above. The **doubled traversal base** and **quadrupled traversal base** are twice and four times that capture.

The opening order is:

1. Capture the fixed word at 0xA9.
2. Capture the word at quadrupled-traversal-base offset 0x74.
3. Write the arithmetic character result to fixed byte location 0x3A.
4. Capture the word at doubled-traversal-base offset 0x14.
5. Capture the word at adjusted-scaled-base offset 4.

The intervening 0x3A write can affect either subsequent read through aliasing.

Next, byte destinations at doubled-traversal-base offset 0x0D and adjusted-scaled-base offsets 8, 0x0D, and 9 receive, respectively, the captured low byte of the adjusted scaled base, 0x6C, and the captured low byte of the fixed-loaded base for both remaining destinations. A self-valued word store targets doubled-traversal-base offset 0x54.

The low byte of the 0xA9 capture is then written to 0x67. A word read at doubled-traversal-base offset 0x44 follows. Only then is 0x67 read for the byte write to 0x21. The computed read may itself be observable; no substitution of immutable captured state is established.

That offset-0x44 capture supplies a byte at adjusted-scaled-base offset 0x10 and a word at fixed-loaded-base offset 0x54. It also supplies a signed right shift whose count is the low eight bits of the adjusted scaled base, with the resulting character written at doubled-traversal-base offset 1. Counts above 31 have unresolved semantics. Fixed-loaded-base offset 0x54 is subsequently overwritten with the adjusted scaled base.

A word from adjusted-scaled-base offset 0x40 is captured before fixed-loaded-base offset 0x74 receives the earlier offset-4 capture. The offset-0x40 capture then becomes a word at adjusted-scaled-base offset 0x10. Fixed-loaded-base offset 0x44 receives the doubled traversal base.

The two remaining 0x67 uses are distinct memory reads:

- The first supplies a byte store at adjusted-scaled-base offset 9.
- A byte store at doubled-traversal-base offset 9 and a word store of the original 0xA9 capture at adjusted-scaled-base offset 0x44 intervene.
- A word at adjusted-scaled-base offset 0x54 is captured as a new destination base.
- A fresh read of 0x67 supplies the second byte store at adjusted-scaled-base offset 9.

The intervening stores may alias 0x67. Equal byte values across these uses are not guaranteed.

Later mutations use the new destination base and retain the earlier snapshots: the quadrupled-traversal-base-offset-0x74 capture supplies low-byte writes, while the doubled-traversal-base-offset-0x14 capture is restored at that base’s offset 0x54.

The closing region captures adjusted-scaled-base offset 0x14, loads a word from doubled-traversal-base offset 4, writes halfword 0x65 at twice that loaded value, and reads its offset 0x64. The new destination base receives the low byte of the adjusted-scaled-base-offset-0x14 capture at offset 0x11 and word 100 at offset 0x54. Further offset-0x74 and offset-0x64 captures precede the final word stores, including a store at twice the earlier offset-0x64 capture.

The final unresolved transfer receives:

1. 0x72.
2. The word captured from offset 0x64 of the base loaded from doubled-traversal-base offset 4.
3. The earlier doubled-traversal-base-offset-0x14 capture.
4. The closing offset-0x74 capture from that loaded base.

A return is represented afterward. Actual return reachability and dispatch semantics remain documented_only.

## Side effects

Explicit effects include fixed-location and computed-address reads and writes, overlapping byte and word accesses, repeated overwrites, and memory-derived destinations. An offset-0x15 byte can overlap an offset-0x14 word. Numerical byte significance depends on byte order.

Dependencies may introduce additional global mutations. Neither argument lists nor captured results establish preservation of memory, ownership, or storage lifetime.

No explicit allocation, release, reference counting, or ownership transfer is identified. This does not establish that callees perform none. No caller-owned result is returned.

There is no explicit status handling, rollback, recovery, or cleanup path. Faults or nonreturning dependencies can leave earlier effects observable. Atomicity, idempotence, reentrancy, and thread safety are not established.

## Dependencies

The supplied opaque mapping identifies only F-e1a1acc0e3f5 itself. No authoritative opaque dependency IDs are available; none are invented here.

| Dependency category | Established interaction | documented_only |
|---|---|---|
| Entry-derived target | First invocation uses link state plus 0xFFE | Identity, signature, validity, effects |
| Embedded-value-derived targets | Ordered invocations with up to four explicit arguments | Identity, preservation, ownership, failure behavior |
| Result-producing targets | Primary, combined, and secondary outputs affect later behavior | Actual result convention and valid domains |
| Final unresolved target | Receives the four values specified above | Dispatch and return semantics |

Similar argument preparation does not prove shared dependency identity.

## Acceptance examples

These are conditional analytical examples, not executed tests.

1. **Unsigned widening:** A captured unsigned byte of 0xE7 produces word 0x000000E7 at its associated widening store.

2. **Small-index boundary:** Under the target-word model, 508 is possible but 511 is not; the resulting small base cannot exceed 1016.

3. **Fixed read timing:** If preceding computed stores change location 0x5, its capture uses the state at the actual read, not entry memory. Its unsigned conversion does not establish byte-width input.

4. **Fixed write provenance:** Under 32-bit truncating arithmetic, saved word 0x80000003 produces word-valued result 0x00000006 for the 0x54 assignment.

5. **Character arithmetic:** If the adjusted address converts to character value 2, the 0x3A assignment has arithmetic value 0x40 before byte assignment. Out-of-range character conversions require target semantics.

6. **Intervening mutation:** If a subsequent word read overlaps 0x3A, its accepted value incorporates the preceding character write according to byte order and access semantics.

7. **Fresh 0x67 reads:** If an intervening computed store changes 0x67 between the two offset-9 stores, the second uses the newly read byte. Reusing the original 0xA9 low byte is unsupported.

8. **Constant overwrite sequence:** Observable memory must retain the offset-0x20 stores of 0x20, 0x2F, widened captured byte, and 0x20 in order.

9. **Excessive shift:** A variable shift count of 40 remains documented_only; neither count masking nor a zero result may be assumed.

10. **Partial execution:** A nonreturning prefix dependency prevents requirements for later explicit stores, without implying that the dependency preserved memory.

## Uncertainties

The following remain documented_only:

- Application purpose, legitimate callers, and complete entry convention.
- Dependency identities, signatures, global effects, and ownership behavior.
- Signed overflow, character conversion, signed right shift, and excessive-shift semantics.
- Missing fixed-symbol declarations and associated transaction-width limitations.
- Byte order, alignment, address mapping, and device observability.
- Aliasing, object boundaries, storage validity, and lifetime.
- Final dispatch semantics and normal-return reachability.

The repeated region’s multiplicity is not quantified here. Its explicit ordering and repetitions remain behaviorally significant.

## Evidence

This contract uses the supplied static representation for explicit operations and value dependencies. Prior prose, comments, apparent signatures, and reviewer statements are not independent proof of behavior.

The strongest evidence is access-width annotation, constants, masks, captured-value provenance, and ordering between represented statements. Ordering within a compound expression and target-specific arithmetic require the qualifications above.

No tools, implementation access, runtime execution, smoke runs, or tests were available or performed. Acceptance examples are static conditions, not observed results.
