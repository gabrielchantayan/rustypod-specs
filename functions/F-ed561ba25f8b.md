<!-- Trusted provenance: F-ed561ba25f8b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ed561ba25f8b

## Purpose

F-ed561ba25f8b accepts two 32-bit arguments, performs an extensive ordered sequence of unresolved indirect interactions, modifies memory through fixed and computed addresses, and ends with a four-operand indirect interaction.

**Established static evidence:** argument forwarding, fixed-width transformations, retained interaction results, mixed-width memory accesses, repeated stores, and a return without an explicit result if the terminal interaction returns.

**Inferred intent [INFERENCE]:** some transformations resemble packed-field processing; some computed memory updates resemble linked-record manipulation. Neither interpretation establishes an application-level purpose.

This contract specifies the visible dependencies and ordering without assigning object, allocation, protocol, or success semantics.

## Interface

### Explicit inputs and output

| Item | Width | Established use |
|---|---:|---|
| First argument | 32 bits | First operand of the initial indirect interaction, unchanged. |
| Second argument | 32 bits | Second operand unchanged; also supplies the third operand after a six-bit left shift. |
| Explicit result | None | Normal return follows the terminal interaction only if that interaction returns normally. |

Neither argument is established as a pointer, size, handle, or ownership-bearing reference. No argument validation is visible.

### Implicit inputs and preconditions

Execution also depends on incoming control-flow state, unresolved interaction results, additional call-associated machine values, fixed-location memory, and computed-address memory.

The initial indirect target incorporates incoming control-flow state plus 0xFFE. Later interactions use embedded control-flow values with the same offset. The explicit arguments do not independently determine the trace.

Completion requires all reached targets to be executable and all reached memory accesses to be permitted at their actual widths. Low addresses, overlapping accesses, and potentially unaligned word accesses are not rejected.

Valid allocation extents, alignment requirements, lifetimes, and ownership are **documented_only**.

### Width and arithmetic conventions

Bytes, halfwords, and words are respectively 8, 16, and 32 bits. Offsets are byte offsets. Byte stores retain the low eight bits; byte-derived integer address bases are zero-extended.

Some interaction results are represented as 64-bit quantities, with numerical low and high 32-bit words. This decomposition does not establish memory byte order.

Arithmetic examples use a **conditional 32-bit machine interpretation**: additions, multiplication, and left shifts retain their low 32 bits. Explicit unsigned right shifts are logical. This interpretation is not a claim that every represented signed expression has portable language semantics.

## Behavior

### Indirect-interaction phase

The initial interaction receives the first argument, the second argument, and the second argument shifted left six bits, in that order.

A long straight-line interaction sequence follows. Observable operand relationships include:

- Interactions with no explicit operands, explicit zero operands, or preceding results.
- Low result words shifted left eight bits, and other left shifts extending to 31 bits.
- High words of 64-bit results, sometimes masked or shifted.
- Additional call-associated machine values whose production rules are unresolved.
- Values retained across intervening interactions.
- Identical transformed values supplied in multiple operand positions.

No explicit operands does not mean no hidden inputs or effects. An unused explicit result does not make an interaction dispensable. No target is established as pure or as preserving global state.

Field extraction uses masks of several widths. A 24-bit field shifted left six bits and a 22-bit field shifted left eight bits each occupy at most 30 significant bits. Unsigned extraction from a 64-bit result after right shifts of 34, 36, or 44 bits corresponds to its high word shifted right 2, 4, or 12 bits. A left shift followed by a right shift does not restore discarded bits.

The exact complete interaction-site inventory and unresolved calling conventions remain **documented_only**; these operand patterns are not a replacement identity map.

### Retained quantities

The following names describe captured quantities, not recovered object types:

| Term | Definition |
|---|---|
| **Packed base** | A retained result word shifted left 25 bits. |
| **Scaled source** | A separate retained result word. |
| **Scaled base** | Scaled source multiplied by 2²⁵. |
| **Retained shifted word** | Another retained result shifted left eight bits. |
| **Bounded index** | A result shifted left eight bits, masked to 15 bits, then logically shifted right six bits. |
| **Doubled index base** | Twice the bounded index. |
| **Retained link word** | A later word captured through a byte-derived base. |
| **Secondary link** | A still later word obtained through a live byte-derived address. |
| **Tail base** | Twice the secondary link. |
| **Quadrupled secondary link** | Four times the secondary link. |
| **Adjusted quantity** | Four times the retained link word minus 150. |
| **Adjusted base** | Four times the adjusted quantity. |

Under the conditional machine interpretation, the bounded index is a multiple of four from 0 through 508. Its doubled base is a multiple of eight from 0 through 1016. These bounds establish no accessible allocation.

The retained link word and secondary link are distinct captures. Four times the retained link word must not be confused with the quadrupled secondary link.

### Opening memory effects and byte-derived capture

The final interaction before the explicit memory phase produces a 64-bit result. Its high word supplies an address base: the word at that base plus 0x14 receives the packed base.

Packed base plus 9 receives the low byte of the returned high word. Next, the word at 256 times the returned low word plus 0x54 receives the scaled base. Packed base plus 9 then receives the low byte of the packed base. These are two separate byte writes with an intervening potentially aliasing word write.

The word at scaled base plus 0x14 is cleared before a byte is captured from scaled base plus 5. Doubled index base plus 0x14 is then cleared, and the captured byte is zero-extended into the word at scaled base plus 0x34.

A word is captured from scaled base plus 0x14. After another clear at doubled index base plus 0x14, two byte stores at doubled index base plus 0x15 both receive the low byte of that captured word. With ordinary stable memory, the captured word is zero; concurrent or device-memory modification can invalidate that premise.

Only after these two byte stores is the byte at fixed address 0x05 read and zero-extended into a retained base. Fixed byte address 0x01 then receives the low byte of the doubled index base. Earlier computed writes may already have affected 0x05, but subsequent changes do not refresh this capture.

The captured byte-derived base plus 0x15 receives the low byte of the bounded index. Scaled base plus 0x11 receives the low byte of the retained shifted word; the earlier clearing-region captured word plus 0x54 receives the full retained shifted word; scaled base plus 0x11 is then cleared as a byte.

The word at captured byte-derived base plus 0x54 is saved as the **retained link word**, before that location is overwritten by the clearing-region captured word.

### Retained-link and fixed-address updates

Doubled index base plus 0x34 receives the bounded index before link reads at offset 0x74. Computed updates include self-address stores at offsets 0x24 and 0x54 and repeated byte stores at offset 9.

One compound update reads doubled index base plus 0x74 for both its destination calculation and its stored word. A single immutable snapshot for that entire expression is not established.

The word at doubled index base plus 0x14 receives scaled source shifted left 26 bits. Immediately afterward, fixed word address **0x54** receives twice the retained link word, truncated to 32 bits.

Doubled index base plus 0x14 then receives the retained link word twice. Two byte stores at doubled index base plus 0x15 receive the low byte of four times the retained link word.

Fixed byte address **0x15** next receives the low byte of the doubled index base. This occurs before the word at four times the retained link word plus 0x54 receives the retained shifted word.

These positions are significant if fixed and computed destinations overlap.

### Computed links and literal updates

The next region uses four and eight times the retained link word, computed words at offsets 4, 0x64, and 0x74, and zero-extended bytes read at offset 5.

Visible effects include self-address stores at offsets 0x14 and 0x24, word updates at 0x34, 0x44, and 0x54, and byte updates at 9, 0x0D, and 0x15. Repeated offset-5 byte reads are separate observations, with computed stores between them.

A later word reached through one such live byte supplies the secondary link. Its doubled value is retained as the tail base; its quadrupled value supplies repeated stored words.

Literal 0x3A is written with three widths: a byte at four times the secondary link, a halfword at eight times that link, and a word at four times that link plus 0x20. The halfword is numeric 0x003A; its byte layout depends on endianness.

At adjusted base plus 0x20, the word receives, in order, 0x20, 0x2F, a zero-extended byte read from adjusted base plus 4, and 0x20 again.

After that final computed 0x20 store, a 32-bit base is captured from fixed location **0x087FBBBC**. Fixed word address **0x60** then receives 0x20. The captured fixed-location base remains retained even if a later store changes its source location.

### Repeated update region

The opening group has four ordered stores:

- A word at eight times the retained link word plus 0x54 receives the quadrupled secondary link.
- A byte at fixed-location base plus 0x19 receives the low byte of the adjusted base.
- A byte at adjusted base plus 0x11 receives the low byte of the fixed-location base.
- A word at fixed-location base plus 0x34 receives eight times the retained link word.

Subsequent explicit groups retain this four-store ordering. Their first destination instead uses eight times the adjusted quantity plus 0x54, and their fourth stored word is eight times the adjusted quantity. The other sources and destinations remain captured values.

The exact independently verified repetition count is **documented_only**. Repetition cannot be collapsed to final-state assignments without additional non-observability and aliasing premises.

### Final fixed-address observations and computed updates

After the repeated region, a word is captured from fixed address **0xA9**, before a word is captured from quadrupled secondary link plus 0x74. Call the latter the **saved secondary-link word**.

Fixed byte address **0x3A** then receives the low byte of eight times the adjusted quantity plus 0x30. This is address-derived, not literal 0x3A.

Words are captured from tail base plus 0x14 and adjusted base plus 4. The first is the **saved tail word**, retained for a later store and the third terminal operand. The second is retained for a later word copy.

Before the next fixed-address store:

- Tail base plus 0x0D receives the low byte of the adjusted base.
- Adjusted base plus 8 receives literal 0x6C.
- Adjusted base plus 0x0D and plus 9 receive the low byte of the fixed-location base.
- Tail base plus 0x54 receives the tail base.

Fixed byte address **0x67** then receives the low byte of the word captured from 0xA9. A word is subsequently read from tail base plus 0x44. Only then is byte address 0x67 read for the byte store at fixed address **0x21**.

The newly loaded tail-offset-0x44 word supplies a byte at adjusted base plus 0x10 and an intermediate word at fixed-location base plus 0x54. It also supplies the byte at tail base plus 1 through a right shift whose distance is the low eight bits of the adjusted base. Negative-value and excessive-distance shift behavior is unresolved.

The intermediate word at fixed-location base plus 0x54 is overwritten by the adjusted base. A word is captured from adjusted base plus 0x40; fixed-location base plus 0x74 receives the earlier captured word from adjusted base plus 4; adjusted base plus 0x10 receives the new word. Fixed-location base plus 0x44 receives the tail base.

A **fresh** read of byte address 0x67 supplies the next store at adjusted base plus 9. Tail base plus 9 then receives the low byte of the adjusted base, and adjusted base plus 0x44 receives the full word captured from 0xA9.

A word is captured from adjusted base plus 0x54 as the **further base**. Another **fresh** read of 0x67 supplies a second byte store at adjusted base plus 9. Intervening computed writes may alias 0x67, so all three reads—one for 0x21 and two for adjusted base plus 9—remain separate observations.

### Closing captures, overwrites, and terminal operands

Further base plus 0x0D receives the low byte of the saved secondary-link word. Tail base plus 0x54 then receives the saved tail word.

A word is next captured from adjusted base plus 0x14 as the **saved adjusted word**. **Immediately after this capture, tail base plus 0x11 receives the low byte of the saved secondary-link word.** This byte store is distinct from the later store at further base plus 0x11.

Adjusted base plus 9 then receives the low byte of the further base, and adjusted base plus 0x44 receives the full further base. These writes precede the late base capture and may affect it through aliasing.

The **late-loaded base** is captured as a word from tail base plus 4. A halfword of numeric **0x0065** is stored at twice that captured base. Only afterward is the **second terminal operand** captured as a word from late-loaded base plus 0x64. The halfword write can therefore affect this later read if their address ranges overlap.

Further base plus 0x11 next receives the low byte of the saved adjusted word, followed by a word store of **100** at further base plus 0x54. Only after these stores is the **fourth terminal operand** captured from late-loaded base plus 0x74. These stores may alias that later read location; the fourth operand is not an earlier snapshot.

The **closing destination base** is then captured as a word from tail base plus 0x64. After both late captures, **tail base plus 9 is overwritten again**, this time with the low byte of the late-loaded base previously captured from tail base plus 4. This is not the earlier write of the adjusted base’s low byte.

The final byte overwrite precedes both closing word stores:

- Closing destination base plus 0x54 receives the captured second terminal operand.
- Twice the captured second terminal operand receives the closing destination base.

These stores use retained captures; they do not refresh either base. Their order remains significant if the destinations overlap.

The terminal indirect interaction receives, in order:

1. 0x72.
2. The captured word from late-loaded base plus 0x64.
3. The saved tail word captured earlier from tail base plus 0x14.
4. The captured word from late-loaded base plus 0x74.

If that interaction returns normally, F-ed561ba25f8b returns without an explicit result. Return does not establish success.

## Side effects

The memory phase is non-transactional. No validation, rollback, commit marker, recovery path, diagnostic, or explicit error result is visible. Faults or non-returning interactions can leave earlier effects intact.

Access widths matter. A byte at base plus 0x15 overlaps a word beginning at base plus 0x14; a byte at base plus 0x11 overlaps a word beginning at base plus 0x10. Distinct descriptive bases are not established as disjoint.

Captured values must remain distinct from live memory reads. This applies especially to the bases from 0x05 and 0x087FBBBC, the word from 0xA9, the saved closing operands, and the separate reads from 0x67.

No allocation or release operation is identified. Unresolved interactions may allocate, release, retain inputs, or modify global state. Ownership transfer and lifetime effects are **documented_only**.

No locking, atomicity guarantee, barrier, or reentrancy guard is visible.

## Dependencies

Only F-ed561ba25f8b has a supplied opaque function ID. No opaque IDs or semantic contracts are supplied for the indirect targets; none are invented here.

Distinct sites need not resolve to distinct functions, and repeated operand shapes do not establish target identity. Target identities, calling conventions, hidden inputs, memory effects, ownership effects, and return behavior remain **documented_only**.

The terminal target is not assumed to be an ordinary returning function.

## Acceptance examples

These are conditional static criteria, not executed tests.

### Initial transformation and index boundary

With second argument 3, the initial interaction includes operands 3 and 192. With bit pattern 0x04000000, the shifted operand is zero under truncating 32-bit semantics.

A relevant index-producing result of 0x7F yields bounded index 508 and doubled index base 1016. The fixed byte store at 0x15 receives 0xF8.

### Retained low-address captures

If the byte captured from 0x05 is 0x80, its offset-0x15 and offset-0x54 accesses use 0x95 and 0xD4. Later changes to 0x05 do not redirect them. The capture precedes the store at 0x01.

If the retained link word is 0x80000003, the fixed word store at 0x54 receives 6 under truncation.

### Separate observations of 0x67

If 0x67 contains 0x12 for the copy to 0x21, then intervening aliasing stores change it to 0x34 and later 0x56 before the two computed byte reads, those byte stores receive 0x34 and 0x56. Reusing the first observation is insufficient.

### Distinct offset-0x11 writes

Suppose the saved secondary-link word ends in 0xAB and the saved adjusted word ends in 0xCD. The byte at tail base plus 0x11 receives 0xAB immediately after the adjusted-word capture. Later, further base plus 0x11 receives 0xCD, after the halfword store and second-terminal-operand capture.

If these destinations alias, both writes remain required in that order. With no other overlapping writes, the later byte is 0xCD.

### Late reads and final overwrite

If the late-loaded base is 0x1234, the halfword destination is 0x2468 and the final byte overwrite at tail base plus 9 receives 0x34. That byte overwrite occurs after the fourth-terminal-operand and closing-destination-base captures, but before both closing word stores.

If a further-base store changes late-loaded base plus 0x74, the fourth terminal operand reflects the later word read. It must not be captured before those stores.

### Intermediate effects and failure

An observer distinguishing the four writes at adjusted base plus 0x20 must observe their stated order. A required low-address access fault does not imply graceful rejection, rollback, or a returned error.

## Uncertainties

The following remain **documented_only**:

- Application purpose, valid input domain, and executable-boundary accuracy.
- Complete interaction-site inventory, dependency identities, calling conventions, and additional machine-value origins.
- Signed overflow and shift semantics beyond the conditional machine interpretation.
- Right-shift behavior for negative words or variable distances exceeding 31.
- Endianness, alignment, allocation extents, ownership, and lifetimes.
- Aliasing relationships, concurrent modification, and device-memory compatibility.
- Ordering of multiple reads within a compound expression where not established.
- Exact independently verified repetition count.
- Terminal target identity and return behavior.

These unknowns do not erase established widths, fixed-address effects, captures, live reads, or cross-access ordering.

## Evidence

This contract uses only the supplied static representation and opaque-ID mapping as evidence. Prior prose, comments, and reviewer feedback are not independent evidence.

The closing representation establishes two distinct offset-0x11 byte stores with different captured sources, a separate final overwrite at tail base plus 9, and late word captures separated by potentially aliasing stores. Those dependencies are retained explicitly rather than summarized as unordered link manipulation.

No tools, implementation execution, runtime tests, smoke runs, memory traces, or dependency executions have been performed. Acceptance examples are consequences of stated premises, not empirical results.
