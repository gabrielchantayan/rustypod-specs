<!-- Trusted provenance: F-cb6da2934e8e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cb6da2934e8e

## Purpose

F-cb6da2934e8e conditionally validates or copies data according to a byte name held in a descriptor. Its three ordered stages are:

1. For a matching three-byte name, obtain a dependency result and validate its metadata and contents.
2. For another matching three-byte name, obtain an outer result and a nested result, then conditionally copy 20 bytes into descriptor-associated storage.
3. For either of two matching ten-byte names, scan up to 64 payload bytes for a caret and conditionally copy the prefix through that caret.

These are sequential stages, not mutually exclusive dispatch alternatives. Descriptor fields are reread between stages, and dependency calls may affect later observations.

The direct result is a 32-bit unsigned value: zero, a dependency return value, or an entry-captured fallback value. No universal success or error interpretation is established.

All behavior below is established by static inspection unless identified as inferred or `documented_only`. No runtime tests have been performed.

## Interface

### Arguments and relevant layouts

The function takes two 32-bit address-bearing arguments.

The first identifies a descriptor:

| Byte offset | Represented width | Meaning used here |
|---|---:|---|
| 0x24 | 32 bits | Selector, compared with 3 and 10 |
| 0x28 | 32 bits | Address of name bytes |
| 0x30 | 32 bits | Address of associated storage |

The associated storage contains these relevant regions:

| Offset from storage address | Extent | Use |
|---|---:|---|
| 1000 / 0x3e8 | 20 bytes | Validation reference |
| 1020 / 0x3fc | 20 bytes | Nested-result destination |
| 1324 / 0x52c | Up to 64 bytes | Caret-prefix destination |

These offsets identify data regions, not pointer fields.

The second argument identifies an input structure:

| Byte offset | Represented width | Meaning used here |
|---|---:|---|
| 0x04 | 32 bits | Signed factor used in extent calculations |
| 0x08 | 32 bits | Payload address or address-like dependency input |

Global configuration supplies a signed 32-bit scale, a 32-bit fallback value, four name-reference table addresses, and a separate 32-bit guarded return value. Their concrete contents are unavailable.

### Results and preconditions

The function returns one unsigned 32-bit value. It can also write 20 destination bytes, write a prefix of 1–64 bytes, and cause whatever effects F-1e8578a40e00 performs. It does not report a copied length separately.

There is no visible validation of pointers, structure sizes, destination capacities, or payload length. Safe completion requires accessibility of every location actually read or written. In particular:

- A final-stage match can cause 64 payload-byte reads independently of the input factor.
- Dependency result addresses must remain readable for subsequent validation or copying.
- Destination regions must accommodate the selected copy.
- Reference and name storage must accommodate the offsets actually inspected.

Extents, addresses, and dependency output slots are represented with 32-bit quantities. Scale products use signed integer representations. Overflow, invalid addresses, and wrapping storage ranges have no established portable behavior.

## Behavior

### Entry captures and comparison rules

The function reads the scale global and then the fallback global, retaining both values. Its return candidate starts at zero.

Four dependency output slots are reused throughout the invocation:

- Result extent: 32-bit signed representation, initially zero.
- Result address: 32-bit unsigned representation, initially zero.
- Consumed extent: 32-bit signed representation, reset to zero immediately before every dependency call.
- Type-like value: 32-bit signed representation, initially zero.

The result extent, result address, and type-like slots are not reset before every call.

The name transformations are:

| Comparison | Required selector | Transformation of reference character |
|---|---:|---|
| First three-byte name | 3 | Multiply by −119, then narrow to a character |
| Second three-byte name | 3 | Multiply by 51, then narrow to a character |
| First ten-byte name | 10 | Integer-promote and negate; no explicit character-width narrowing |
| Second ten-byte name | 10 | Multiply by 37, then narrow to a character |

For each newly inspected name position, the descriptor’s name-address field is loaded again, followed by the name character at that position. Each reference lookup separately reloads the applicable global table address and reads its reference character. Neither the name address nor the table address is captured for an entire comparison.

The two three-byte comparisons and the second ten-byte comparison compare retained character values after those reads. Their repeated mismatch decisions do not perform a second memory lookup. Consequently, a mismatch rejects that comparison, and acceptance requires all its positions to match.

The first ten-byte comparison has different reread behavior, specified below.

For narrowed character transformations, the represented eight-bit model yields the low-eight-bit pattern of the arithmetic result. Character signedness remains relevant to promoted comparisons, especially the unnarrowed negation.

### First stage: validation

The function reads the descriptor selector. Only a value of 3 enables the first three-byte comparison.

If the selector or name fails to match, this stage makes no dependency call and leaves the candidate at zero.

On a match, the consumed-extent slot is reset and F-1e8578a40e00 is called with:

- The payload argument from input offset 0x08.
- An extent formed from the **live scale global** multiplied by the factor at input offset 0x04.
- Addresses of the four shared output slots.

The call expression does not establish an evaluation order between reading the payload argument and evaluating the scale product. Nor does it establish an order between the scale and factor operand reads. These reads precede the dependency invocation, but must not be specified as a payload-first sequence.

The dependency return becomes the candidate. Exactly zero permits validation; every nonzero return bypasses validation.

After a zero return, validation requires, in short-circuit order:

1. Type-like value equals 4.
2. Consumed extent equals the **entry-captured scale** multiplied by a newly read factor.
3. Result extent equals 20.
4. Content validation accepts.

A failed requirement changes the candidate to the entry-captured fallback.

For content validation, positions are considered in increasing order from 0 through 19. At each newly considered position:

- The descriptor’s associated-storage address is reloaded.
- The reference character at storage offset 1000 plus the position is read.
- That character is multiplied by −75, increased by 36, and narrowed to a character.
- The corresponding result character is read and compared with that computed character.

If the first result read matches, validation proceeds to the next position.

If the first result read mismatches, the function reads the same result position again, without recomputing the reference character:

- A second mismatch rejects validation.
- A second read that matches causes **immediate successful completion of content validation**. No later reference or result positions are checked.

Thus, with stable memory, acceptance requires all 20 characters to match. With changing observed result contents, a mismatch followed by a matching reread can accept a prefix and leave the remaining positions unexamined.

Successful validation leaves the candidate at zero. This stage performs no explicit destination copy.

### Second stage: nested extraction

The descriptor selector is read again. A value of 3 enables the second three-byte comparison. A failed selector or comparison preserves the preceding candidate.

On a match, the consumed-extent slot is reset and F-1e8578a40e00 is called with the current payload argument and the entry-captured scale multiplied by a newly read factor. Ordering among independent argument evaluations is not established.

The return replaces the candidate. The outer result extent is saved separately after the call, including when the return is nonzero.

A nonzero return ends this stage without copying. After a zero return, the candidate becomes the captured fallback. Further processing requires:

1. Type-like value equals 0x30.
2. Consumed extent equals the captured scale multiplied by the factor read for this check.

If both hold, the consumed-extent slot is reset and the dependency is called again. This nested call uses the outer result address and outer result extent as its input. It reuses all four output slots.

The nested return replaces the candidate. A nonzero return ends the stage. After a zero return, the candidate becomes the captured fallback, and copying requires:

1. New type-like value equals 0x80.
2. New consumed extent equals the separately saved outer result extent.
3. New result extent equals 20.

On acceptance, the descriptor’s associated-storage address is loaded to establish the destination at offset 0x3fc. Exactly 20 bytes are copied from the nested result address:

- If the unsigned source address is less than or equal to the unsigned destination address, offsets 19 through 0 are copied.
- Otherwise, offsets 0 through 19 are copied.

Each source byte is read before its corresponding destination-byte write. There is no whole-result snapshot. The candidate becomes the nested call’s zero return.

For valid, non-wrapping ranges with stable source storage, these directions preserve the original source bytes under overlap. The function neither releases the result storage nor establishes its ownership.

### Third stage: name checks and caret-prefix copy

The descriptor selector is read again and retained for both ten-byte comparisons.

If that retained value is 10, the first ten-byte comparison examines increasing positions. At each position it reloads the descriptor’s name address, reads and retains the name character, then reloads the first ten-byte reference-table address and compares against the negated promoted reference character.

On an initial mismatch, it performs a second lookup of the reference side:

- The table-address global is reloaded.
- The reference character at the same position is reread and negated.
- The already-retained name character is compared again.

A second mismatch rejects the name. A matching second comparison **accepts the name immediately**, without examining later positions. The name-address field and name character are not reread for this retry.

With stable memory, all ten positions must match. Under changing reference observations, the retry can accept after only a prefix.

The second ten-byte comparison is then performed whenever the retained selector is 10, even if the first comparison accepted. It uses multiplication by 37 and character narrowing. It has no mismatch-triggered reference reread; all ten positions must match for acceptance.

If neither name accepts, the function returns the preceding candidate.

If either accepts, the separate guarded-return global is loaded and the retained selector is tested for being greater than 64. That branch cannot select the loaded value on the represented path: acceptance required the same retained selector to equal 10. The global load nevertheless occurs.

The payload address is then read once from input offset 0x08. Starting at offset zero, the function scans for the caret byte, 0x5e:

- A caret at offset 0–63 selects a copy extent of that offset plus one.
- Sixty-four non-caret observations select a copy extent of zero.
- Offset 64 is not read by the scan.

The associated-storage address is loaded to establish the destination at offset 0x52c, including in the zero-extent case. Copy direction uses unsigned addresses:

- Destination strictly below source: increasing offsets.
- Otherwise: decreasing offsets.

The copy rereads source bytes; it does not reuse a snapshot from scanning. With stable contents, the copied prefix includes the caret. No terminator is appended. If no caret was found, there are no destination-byte writes.

The final stage returns the candidate preserved from the preceding stages.

## Side effects

The major sequence is entry captures, first-stage processing, second-stage processing, and final-stage processing. Later stages can observe descriptor or global changes made by a dependency. Later dependency calls can replace an earlier candidate.

Important read distinctions are:

- Every newly inspected name position reloads the descriptor’s name-address field.
- Every name-reference lookup reloads its table-address global.
- The first ten-byte comparison can perform a second reference lookup at one position, but retains that position’s name character.
- Every newly computed first-stage validation character reloads the associated-storage address.
- A validation mismatch triggers another result-byte read, not another reference computation.
- Each explicit copy establishes its destination address before copying; it does not reload the descriptor storage field for every byte.

Copy reads and writes are interleaved byte by byte. Source contents can therefore differ between an earlier scan or validation and a later read. No synchronization or stable-memory guarantee is visible.

There is no explicit allocation, deallocation, reference counting, ownership transfer, or cleanup. Dependency-produced storage may be borrowed or owned; the function does not establish which.

No local error recovery is visible for inaccessible storage or arithmetic faults. A fallback return indicates rejection of certain dependency outputs, not a general memory-safety failure mechanism.

## Dependencies

### F-1e8578a40e00

This dependency is called zero to three times.

Its observed interface comprises:

- One 32-bit input address or address-like argument.
- One 32-bit input extent.
- Writable addresses for result extent, result address, consumed extent, and type-like output.
- One unsigned 32-bit return.

Exactly zero enables output inspection. Nonzero returns become candidates, subject to later stages replacing them.

Only the consumed-extent output is explicitly cleared before every invocation. If another output is left untouched, subsequent processing can observe its entry initialization or a value from an earlier call.

The dependency’s format, error meanings, output completeness, memory lifetimes, ownership, and effects on globals or caller-visible storage are unknown. The constants 4, 0x30, and 0x80 are acceptance values here; their semantic names are not established.

## Acceptance examples

These are conditional static expectations, not executed tests.

1. **No applicable selector.** With stable storage and a selector other than 3 or 10, there is no dependency call, payload scan, or explicit copy. The return is zero.

2. **First dependency fails.** If only the first three-byte name matches and its call returns 7, first-stage validation is skipped. The return is 7 if later stages do not replace it.

3. **Stable first-stage success.** A zero return, type 4, matching consumed extent, result extent 20, and 20 matching transformed reference characters leave the candidate at zero.

4. **Stable validation mismatch.** If validation reaches position 19 and both reads there mismatch the computed reference character, the candidate becomes the captured fallback. There is no explicit destination write.

5. **Validation reread accepts early.** Suppose metadata passes and positions before 4 match. At position 4, the first result read mismatches but the second matches the already-computed reference character. Validation succeeds without inspecting positions 5–19. This example requires changing observations; it is not possible with stable result contents.

6. **Outer extraction rejected.** A zero outer return with type other than 0x30 leaves the fallback candidate, performs no nested call, and makes no explicit 20-byte copy.

7. **Nested failure.** After accepted outer metadata, a nested return of 9 becomes the candidate. No nested-result destination copy occurs.

8. **Overlapping nested copy.** Accepted nested metadata with source below an overlapping destination causes writes from offset 19 down to 0. Exactly 20 bytes are copied, and the candidate becomes zero.

9. **Ten-byte reference reread accepts early.** At position 2, the retained name character initially differs from the negated reference character. A second table lookup yields a matching negation. The first ten-byte name accepts without reading its positions 3–9. The second ten-byte comparison still runs.

10. **Caret boundaries.** With a final-stage name accepted, a caret at offset zero selects one copied byte; a caret at offset 63 selects 64. Neither case appends a terminator.

11. **No caret.** Sixty-four non-caret observations cause reads at offsets 0–63, followed by zero copied bytes. The destination address is still obtained, and the preceding candidate is returned.

## Uncertainties

- **documented_only — Configuration:** Literal accepted names and concrete scale, fallback, and guarded-return values are unavailable.
- **documented_only — Dependency contract:** Parsing semantics, output guarantees, storage ownership, lifetimes, and mutation effects are unknown.
- **documented_only — Character model:** Signedness is not independently established. The first ten-byte transformation compares promoted negation directly; it must not be replaced by modulo-256 byte negation.
- **documented_only — Arithmetic:** Signed product overflow, address overflow, and invalid-access behavior are not established as portable guarantees.
- **documented_only — Argument evaluation:** The supplied call expressions do not establish ordering among independent argument evaluations, including the first call’s payload read and live-scale product evaluation.
- **documented_only — Changing memory:** The represented rereads establish possible observation points, not a synchronization protocol or a guarantee of defined behavior for unsynchronized concurrent mutation.
- **Inferred intent:** Nested extraction and type constants suggest structured decoding, but no particular encoding is established.
- **Inferred intent:** Directional copies appear intended to support overlap within valid, non-wrapping storage.

## Evidence

Static evidence establishes the field offsets, represented widths, arithmetic constants, dependency sequence, candidate updates, output-slot reuse, copy extents, and copy directions.

It also establishes the two distinct mismatch-triggered reread paths: first-stage validation rereads a result byte against a retained reference character; the first ten-byte comparison reloads its reference table and character against a retained name character. Either path can accept early when its second observation matches.

The evidence does not establish concrete configuration contents, dependency internals, ownership, portable overflow behavior, or defined execution under invalid memory access or unsynchronized mutation.

No runtime tests have been performed.
