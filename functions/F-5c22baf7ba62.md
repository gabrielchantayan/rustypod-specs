<!-- Trusted provenance: F-5c22baf7ba62; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5c22baf7ba62

## Purpose

F-5c22baf7ba62 accepts an opaque 32-bit input, invokes a long sequence of unresolved indirect dependencies, transforms dependency results, performs ordered mixed-width memory updates, and reaches a final unresolved indirect transfer.

No application-level purpose is established. Address-like results, self-referential stores, packed-field operations, and character-like constants do not establish allocation, initialization, parsing, formatting, or collection semantics.

This contract distinguishes:

- **Established static evidence:** behavior represented explicitly in the supplied material, conditional on that representation accurately describing execution.
- **Inferred intent:** possible interpretations, not acceptance requirements.
- **documented_only:** unresolved behavior for which no executable guarantee is asserted.

The explicit input does not independently determine the effects. Incoming execution state, dependency behavior, existing memory, arithmetic semantics, and aliasing also matter.

## Interface

### Explicit input and output

The explicit input is one 32-bit value. Its signedness, interpretation, and valid range are unresolved.

F-unknown-001 receives that value unchanged in its first displayed argument position. No validation precedes this invocation. The explicit input is not directly referenced afterward, although dependency results and side effects may retain its influence.

No return value is represented. This does not establish successful completion, a status convention, or termination.

### Implicit inputs

The represented behavior also consumes:

- Two incoming 32-bit register-held values before any represented assignment to them.
- Incoming link-related state used in the first target calculation.
- Secondary result-register values associated with dependency invocations.
- Existing dynamically addressed memory.
- Values obtained from fixed-location globals.
- An unresolved final transfer target.

The calling convention and initialization requirements for implicit execution state are **documented_only**.

### Widths, conversions, and layout

“Byte,” “halfword,” and “word” mean 8, 16, and 32 bits respectively.

Explicitly typed dynamic accesses establish all three widths. Some dependency results occupy represented 64-bit containers whose low and high 32-bit portions are consumed separately. This establishes two-word result consumption, not necessarily a language-level 64-bit return convention.

At explicitly 8-bit dynamic destinations, the stored value is narrowed to 8 bits. Explicit unsigned-byte observations widened to 32 bits are zero-extended.

**Fixed-location globals require a separate rule:** their declarations are absent. Their names, receiving local widths, and right-hand-side conversions do not establish the globals’ storage widths. This applies to locations 1, 5, 21, 33, 58, 84, 96, 103, 169, and 0x087FBBBC.

In particular:

- Assignment into a 32-bit local establishes the width of the retained local value, not the width of the preceding global access.
- An explicit 8-bit conversion establishes narrowing of the assignment’s source value, not an 8-bit destination object.
- A global-to-global assignment does not establish byte-sized reading or writing.
- Any additional conversion imposed by an absent global declaration is **documented_only**.

Dynamic offsets include 1, 4, 5, 8, 9, 13, 16, 17, 20, 21, 25, 32, 36, 52, 64, 68, 84, 100, and 116. These do not establish one record layout or allocation size. Bases differ, may originate as scaled integers, and may alias.

Byte order, alignment requirements, address validity, and pointer provenance are unresolved.

### Preconditions and errors

Fault-free execution requires every reached indirect target to be callable and every reached memory operation to be valid at its actual width. Arithmetic and shift operations also require execution-environment semantics consistent with the represented behavior.

No local null check, bounds check, checked-overflow path, diagnostic, recovery path, or explicit error return is represented. A failure can occur after earlier effects have taken place.

## Behavior

### Dependency-driven phase

The opening phase contains a long, straight-line sequence of indirect invocations. No explicit conditional branch, loop, retry, or early return is represented.

F-unknown-001 receives the explicit input and the first implicit incoming value shifted left by 5 and 11 bits. Later invocation groups consume the second implicit incoming value shifted left by 20 and 26 bits.

The first target depends on incoming link-related state. Most subsequent targets use represented code-related anchors plus 4094 bytes. Similar target calculations do not establish identical targets.

Invocation order is significant:

- Sites display between zero and four arguments.
- An empty displayed argument list does not establish absence of implicit inputs.
- Results can be retained across intervening invocations, transformed, and forwarded later.
- Some displayed arguments are explicit zero or derive from already-zero intermediates.
- No visible status check gates continuation.
- An unused displayed result does not make the invocation removable.

Unknown dependencies must not be assumed pure, memory preserving, globally preserving, or returning. The complete site-by-site dependency API is **documented_only**.

### Arithmetic and extraction

Common transformations shift a low result word left by 8 bits or a high result word left by 6 bits. Other shifts extend to 31 bits. Field masks cover widths including 1, 11 through 22, and 24 bits.

Intermediate widths and mask placement matter. Sequential shifts must not be replaced with widened arithmetic that retains bits discarded by a 32-bit intermediate. Likewise, masking before shifting is not generally interchangeable with shifting an unmasked value.

Logical right shifts of 64-bit result containers include counts of 34, 36, and 44. These consume portions of the high result word; they are not 32-bit shifts by those counts. Some extractions use retained results rather than the immediately preceding invocation’s result.

Address-related arithmetic includes multiplication by 2, 4, 8, 256, and 33,554,432, together with subtraction of 150 before further scaling.

Several operands are signed-looking. Portable signed-overflow and signed-shift behavior is not established. Low-32-bit arithmetic is therefore a conditional machine-width interpretation, not an unconditional rule for every expression. No saturation or overflow reporting is represented.

### Opening direct-memory updates

Direct memory effects begin after the dependency-driven phase’s last invocation.

A word store at offset 20 through the final high result portion precedes an 8-bit store at offset 9 of a retained transformed base. A word store then uses the final low result portion multiplied by 256, at offset 84. The earlier offset-9 byte destination is subsequently overwritten from a different source expression.

The source expressions need not yield different bytes. The requirement concerns their selection and order, not numerical inequality.

Next, a word at offset 20 of a large scaled base is zeroed, and a byte at offset 5 of that base is observed. Another scaled base receives a zero word at offset 20. The saved byte is zero-extended into a word store at offset 52 of the first base.

The first zeroed word is read only after those intervening accesses. Its observation cannot be replaced with a presumed zero without proving that intervening writes do not affect it. Another zero word store precedes two consecutive byte stores to the same offset-21 destination, both using the saved word observation’s low byte.

### Early fixed-location effects and dynamic linking

The next fixed-location observation is from location **5**, explicitly converted into a retained unsigned 32-bit value. The global read width and any conversion before that explicit conversion remain **documented_only**.

An assignment at location **1** follows. Its source is explicitly narrowed to 8 bits; its destination storage width and any further assignment conversion are unresolved.

The retained value derived from location 5 is used as an address base, including for a dynamic byte store at offset 21 and a later word read at offset 84. That word observation is saved before its source is overwritten.

Subsequent dynamic effects include:

- Word reads at offsets 84, 100, and 116.
- Word stores through loaded address-like values.
- Self-referential word stores, including at offset 36.
- Repeated stores to identical byte destinations.
- Separate byte observations at offset 5 with intervening writes.

These are ordered observations, not permission to reuse one earlier load for all later occurrences.

An assignment at fixed location **84** receives a saved dynamic word shifted left by one bit. It occurs between dynamic offset-20 stores: first from an earlier shifted expression, then from the saved word, followed by another store of that saved word. The fixed assignment’s storage width and destination conversion are **documented_only**.

After two further byte stores to the same dynamic offset-21 destination, location **21** receives a source explicitly narrowed to 8 bits from a retained scaled address. This does not establish the width of the global at 21.

### Constants and retained fixed-location-derived base

The dynamic phase stores `0x3A` separately through an 8-bit destination, a 16-bit destination, and a 32-bit destination. Their differently scaled addresses do not establish a string operation.

At another dynamic word destination, the represented value order is:

- `0x20`.
- `0x2F`.
- A zero-extended byte observed at offset 4 of the calculated base.
- `0x20` again.

The byte observation follows the first two writes. Aliasing can therefore affect the value it supplies.

After this sequence, a global value associated with **0x087FBBBC** is assigned into a retained **32-bit local value** used as an address base. The source global’s access width, signedness, and any extension or conversion into the local are **documented_only**. The local’s 32-bit width does not prove a 32-bit memory read.

The next represented memory effect is an assignment of `0x20` at fixed location **96**, before the expanded repeated-store region. Its storage width and assignment conversion are also unresolved.

Later accesses use the retained 32-bit base. They do not represent fresh reads of 0x087FBBBC. Conversely, retaining that base does not guarantee that memory associated with 0x087FBBBC remains unchanged.

### Expanded repeated-store region

The repeated region uses five descriptive quantities:

- A pivot saved from the earlier dynamic offset-84 word read.
- An adjusted value corresponding to four times the pivot minus 150.
- A scaled base corresponding to four times that adjusted value.
- The retained 32-bit base derived from the global at 0x087FBBBC.
- A retained word obtained by multiplying another dynamically loaded value by four.

These arithmetic descriptions remain subject to the width and signed-arithmetic qualifications above.

The opening group stores the retained word at offset 84 of eight times the pivot. It then stores the scaled base’s low byte at offset 25 of the fixed-location-derived base, stores that retained base’s low byte at offset 17 of the scaled base, and stores eight times the pivot as a word at offset 52 of the retained base.

Subsequent groups preserve those byte-store roles but use eight times the **adjusted value**, rather than eight times the pivot, for both the offset-84 address calculation and the offset-52 word value.

Those expressions are distinct even if particular values or arithmetic semantics make their results coincide.

This is expanded repetition, not a represented data-dependent loop. Repeated accesses cannot be collapsed when intermediate writes are observable. The exact repetition count is **documented_only** here; this contract does not claim a complete bus-level trace.

### Location 169, location 58, and the first assignment at 103

After the repeated region, a global value associated with location **169** is assigned into a saved **32-bit local value**.

The global read width and the conversion producing that local value are **documented_only**. The established retained object is the 32-bit local value, not a proven full-word memory observation at 169.

Before an assignment at location 103, the function:

- Reads a dynamic word at offset 116 of a retained scaled base.
- Assigns a character-arithmetic expression at location **58**.
- Saves dynamic words at offsets 20 and 4 of two other bases.
- Performs several dynamic byte stores and a self-referential word store.

For location 58, the adjusted value is first subjected to a character conversion, then multiplied by 8 and increased by `0x30` under integer promotion. This does not establish an 8-bit arithmetic result or an 8-bit global write. Character signedness, relevant conversion details, and the destination global’s width remain **documented_only**.

The assignment at location **103** uses an explicit 8-bit conversion of the saved 32-bit local value derived from 169. The global’s storage width and any additional assignment conversion are unresolved.

That source narrowing does not discard the saved local: its full 32-bit value remains available for a later explicitly word-sized dynamic store at offset 68. No fresh read of location 169 is represented in the remainder.

### Fresh observations of location 103

After the assignment at 103, a dynamic word at offset 68 is loaded. A fresh global observation of **103** then supplies an assignment at fixed location **33**.

Neither global’s access width nor the conversion between them is established. In particular, this is not established as a byte-to-byte copy.

Further dynamic effects intervene: byte and word stores, a variable-shift byte store, a word read at offset 64, and stores using previously retained values. A second fresh observation of **103** then supplies an explicitly 8-bit dynamic store at offset 9 of the scaled base.

That store is followed by another dynamic byte store, a word store of the saved 32-bit local derived from 169 at offset 68, and a dynamic word read at offset 84. A third fresh observation of **103** supplies another explicitly 8-bit store to the same offset-9 destination.

The three global observations are distinct. Their source access widths and declaration-dependent conversions are **documented_only**. The two dynamic destinations are established as 8 bits; where an observed global value is an integer, their stored value is its low 8 bits.

Intervening accesses may alias the global at 103. Replacing later observations with the initially assigned narrowed value is not justified.

### Tail and final transfer

Later stores overwrite earlier byte and word destinations. A saved offset-4 word is copied after intervening accesses rather than reread. The saved offset-20 word likewise survives to become a final displayed argument.

One dynamic byte store uses a signed-looking word right-shifted by a count derived from the scaled base’s low 8 bits. The count may exceed 31. Large-count behavior and sign propagation are **documented_only**; masking the count to five bits is unsupported.

The tail includes a halfword store of `0x65` through twice a loaded base and a word store of decimal 100 at offset 84 of another loaded base.

F-unknown-003 receives four displayed arguments:

1. `0x72`.
2. A saved word loaded at offset 100 of a late-loaded base.
3. The earlier saved offset-20 word.
4. A word subsequently loaded at offset 116 of that same late-loaded base.

The offset-100 observation precedes the offset-116 observation, with other accesses intervening. Further linking stores occur before the final invocation. No cleanup follows before the represented return. Whether the final transfer returns normally is unresolved.

## Side effects

The function performs explicitly byte-, halfword-, and word-sized dynamic accesses, together with fixed-location global accesses of unresolved widths.

Reads determine later addresses and values. Stores can change later observations through aliasing. The represented ordering does not establish a universally stable final memory image, nor does it independently establish volatile, atomic, or hardware transaction semantics.

No rollback is shown. Earlier writes need not be restored after a fault.

No allocation, release, retention, borrowing, or ownership-transfer contract is established. A result used as an address is not thereby valid or newly allocated. Dependencies may mutate globals, invalidate memory, allocate, release, retain values, fail, or not return.

No locking, barrier, reentrancy, or thread-safety guarantee is established.

## Dependencies

Only F-5c22baf7ba62 has a supplied resolved opaque identity.

| Opaque ID | Role |
|---|---|
| F-unknown-001 | First incoming-link-derived invocation |
| F-unknown-002 | Collection of subsequent unresolved indirect invocation sites |
| F-unknown-003 | Final unresolved indirect transfer |

F-unknown-002 does not imply that its sites resolve to one function.

For every dependency, actual calling convention, argument meaning, ownership behavior, global effects, failure behavior, and termination are **documented_only**.

## Acceptance examples

These are conditional static obligations, not executed tests.

### Input forwarding

Given explicit input `0x12345678`, if execution reaches a callable F-unknown-001, its first displayed argument is `0x12345678`. Zero input does not select a represented local shortcut.

### Explicit dynamic widths

An unsigned byte observation of `0xE7` widened for an explicit dynamic word store supplies `0x000000E7`, not `0xFFFFFFE7`.

Two consecutive opening byte-store sources may both yield low byte `0x40`. Both represented stores remain relevant; acceptance must not require unequal values.

### Retained value derived from 169

Suppose the global evaluation and unresolved conversion associated with location 169 produce saved local value `0x123456A5`. The explicit source conversion for the assignment at 103 yields `0xA5`. The later dynamic word store at offset 68 uses saved value `0x123456A5`.

This example does not require a four-byte read at 169 or a one-byte write at 103.

### Fresh global observations

Suppose an intervening aliased write changes the value observed from 103. The assignment at 33 must use its own later observation, subject to the unresolved global declarations and conversions.

If a subsequent observation supplies integer value `0x17C` to an explicitly 8-bit dynamic destination, that destination receives `0x7C`. This does not establish the width of the read from 103.

### Retained address base

If evaluation of the global at 0x087FBBBC produces a valid retained 32-bit base, later changes to that global do not automatically replace the saved base. This obligation does not specify the initial read width.

### Partial execution

If a dependency does not return, later effects need not occur. If an access faults after earlier writes, completion and restoration must not be assumed.

## Uncertainties

The following remain **documented_only**:

- Application-level purpose and genuine executable-function boundaries.
- Calling convention, implicit-state initialization, and target identities.
- Complete dependency trace and exact expanded repetition count.
- All absent fixed-location global declarations, access widths, and declaration-dependent conversions.
- Conversions into retained 32-bit locals from 169 and 0x087FBBBC.
- Character signedness and conversion details affecting location 58.
- Address validity, provenance, alignment, byte order, and aliasing.
- Signed overflow, signed shifts, and large variable-shift semantics.
- Ownership, dependency effects, diagnostics, recovery, concurrency, and termination.
- Complete final-state guarantees and the final transfer’s actual control-flow role.

Linked structures, packed data, and text-like output are **inferred intent only**, not normative interpretations.

## Evidence

The evidence is the supplied static representation: explicit input width, displayed invocations, result-container consumption, arithmetic, explicitly typed dynamic accesses, global references, retained local widths, and represented ordering.

A receiving local’s width is not evidence of a global read’s width. A narrowed assignment source is not evidence of a global destination’s width. Integer promotion in character arithmetic must not be mistaken for byte-width arithmetic.

Comments, warnings, prior prose, reviewer feedback, and symbol naming are not independent evidence of executable semantics or missing global declarations.

No implementation access, independent instruction analysis, runtime execution, or runtime tests have been performed. No snapshot hash or independently verified provenance is asserted.
