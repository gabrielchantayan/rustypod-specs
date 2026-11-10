<!-- Trusted provenance: F-f5b72f3fca12; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f5b72f3fca12

## Purpose

F-f5b72f3fca12 fills a caller-provided descriptor list using data obtained through a dynamically selected callback. It recognizes two alternative formats:

- A four-entry format selected by a 16-bit marker at an externally configured displacement within a working buffer.
- A record-oriented format selected by a 16-bit marker at the beginning of that buffer.

Recognized input produces a format discriminator, a 32-bit descriptor count, and compacted 12-byte descriptor slots. Each populated slot contains a one-byte classification and two 32-bit fields.

**Established static behavior:** callback selection, metadata initialization, marker precedence, entry filtering, record-field transformation, descriptor writes, scan limits, and status selection.

**Inferred intent:** the layouts and comparison-reference labels suggest storage-region discovery. Neither field units nor storage-format semantics are established. In particular, the record-field transformation must not be described as ordinary ASCII uppercasing: it changes bytes outside the lowercase-letter range.

**Verification status:** static analysis only. No runtime tests have been performed.

## Interface

### Arguments and return value

The function receives three arguments:

1. **Selector:** an opaque 32-bit value passed to F-fbcf824e7197.
2. **Output buffer:** writable caller-provided storage for metadata and descriptors.
3. **Working buffer:** caller-provided storage passed to callbacks, then inspected and sometimes modified directly.

The return value is a 32-bit integer. Completed recognized-format processing returns zero. Checked callback failures and an unrecognized format select values from external error locations rather than returning the callback status itself.

The numeric error values are unavailable. The function does not require those external values to be nonzero, so a zero return alone is not proven to distinguish every error path.

### Output layout

Offsets below are byte offsets from the output-buffer base.

| Offset | Width | Meaning |
|---|---:|---|
| 0 | 1 byte | Format discriminator: directly initialized to 0; completed formats use 1 or 2 |
| 1–3 | 3 bytes | Not directly initialized |
| 4 | 4 bytes | Descriptor count: directly initialized to 0 |
| 8 + 12 × index | 1 byte | Descriptor classification |
| 9 + 12 × index through 11 + 12 × index | 3 bytes | Descriptor padding, not directly populated |
| 12 + 12 × index | 4 bytes | First descriptor field |
| 16 + 12 × index | 4 bytes | Second descriptor field |

The four-entry path can populate four descriptors; the record-oriented path can populate eight. At least 104 bytes accommodate every output field directly addressed by this function. Dependencies may impose additional requirements.

Unused slots and padding retain their previous contents unless affected by dependencies or aliasing.

### Working-buffer extent and access requirements

The four-entry path has fixed accesses through byte offset 509, requiring at least 510 accessible bytes for those accesses. Its marker read additionally requires two accessible bytes at an externally supplied displacement.

The record-oriented path directly accesses bytes through offset 79. Callback and helper requirements are not established.

The supplied representation uses 16-bit and 32-bit accesses, including 32-bit fields at offsets that are only two-byte aligned relative to the working-buffer base. Valid execution requires suitable addresses and an execution environment supporting the represented accesses. Native multi-byte interpretation is visible; exact target endianness is not supplied.

### Ownership and validity

Both buffers are caller-provided. No allocation, deallocation, or explicit ownership transfer appears in the function itself. Dependency ownership and lifetime behavior remain unknown.

There are no visible checks for null pointers, capacity, overlap, valid callback storage, or a valid marker displacement. The ordinary parsing contract assumes valid storage, stable input during repeated observations, and non-overlapping buffers. With overlap, output initialization or descriptor writes may alter data before later reads.

These assumptions do not imply that dependencies preserve buffers, globals, callback objects, or memory ownership.

## Behavior

### Initial resolution and metadata initialization

The first observable dependency call is F-fbcf824e7197 with the selector, the output-buffer address, and a zero third argument.

After it returns, the function directly writes the output discriminator to zero, then writes the 32-bit output count to zero. It subsequently reads the first callable entry from the returned object and invokes it with the working-buffer address.

Thus, the resolver runs before direct metadata initialization, while the initial callback runs afterward. Neither dependency is assumed to preserve output bytes.

A nonzero initial callback result causes an immediate return of an externally stored initial-operation error value. No format inspection follows.

### Format selection and precedence

After a zero initial callback result, the function reads a 16-bit marker at the working-buffer base plus an external displacement.

A marker bit pattern of 0xAA55 selects the four-entry path. This is represented by a signed 16-bit comparison against −0x55AB.

Only if that comparison fails does the function inspect the leading 16-bit value. A leading value other than 0x5245 returns a separate externally stored format-error value.

Consequently, the displaced marker has precedence. When both markers match, only the four-entry path executes.

### Four-entry filtering and extraction

Exactly four source entries are considered, in increasing source-index order. For source index \(j\), from 0 through 3:

| Field | Working-buffer byte offset |
|---|---:|
| Classification source | 450 + 16 × \(j\) |
| First descriptor field | 454 + 16 × \(j\) |
| Second descriptor field | 458 + 16 × \(j\) |

The classification is the low eight bits of a 16-bit value. The two descriptor fields are 32-bit values.

Eligibility first depends on a nonzero second field. Only when that test succeeds is the classification checked against the accepted set.

| Source classification | Output classification |
|---|---:|
| 0x00 | 2 |
| 0x0C | 3 |
| 0x0B | 4 |
| 0x3F | 1 |

Other classifications are rejected. Accepted entries occupy consecutive output slots, preserving source order.

For an accepted entry, the observable field-access order is:

1. Read and retain the first field.
2. Read the second field again and write it to the output slot.
3. Write the retained first field.
4. Read the classification again, select its mapped value, and write the one-byte classification.

The second field and classification therefore have repeated observations. The ordinary mapping guarantee requires those observations to remain consistent. If the classification changes to an unsupported value between eligibility and extraction, the supplied representation does not establish a fresh classification assignment for that case.

After all four entries, the function writes discriminator 1, then the accepted count, and returns zero. Recognition succeeds even when no entries qualify.

### Record-oriented setup

If the displaced marker fails and the leading value is 0x5245, the function calls F-fbcf824e7197 again. Only the selector is visibly supplied on this invocation.

The first callable entry of the returned object is invoked with the working-buffer address, record number 1, and a final argument of 1.

After this callback returns, the function reads a shared external error value. A nonzero callback result returns that already-read value. No final format or count is published.

The differing resolver call forms are established, but no authoritative dependency prototype is supplied to explain them.

### Scan limit and record acquisition

After successful acquisition of record 1:

- F-4055be9c3f18 receives the working-buffer address plus 4.
- F-b81910dbb229 supplies a value interpreted as an unsigned 32-bit count.
- Values greater than 255 are reduced to 255.

The scan covers record numbers 1 through the clamped count, inclusive. There is no arithmetic wrap in that bounded record-number progression.

Record 1 has already been acquired, so it is not requested again at loop entry. A count of zero skips all per-record processing, but does not undo or avoid the preceding record-1 callback.

For each subsequent record, the function reads the callable entry from the second resolver’s returned object and invokes it with the working-buffer address, that record number, and 1. The resolver is not called again for every record.

A nonzero later callback result immediately returns the shared error value read after that failure. Previously written descriptor slots are not rolled back, and final metadata publication is skipped.

### Record-field byte transformation

Before either classification comparison, the function visits every byte at working-buffer offsets 48 through 79, in increasing address order. This is a fixed 32-byte operation, not a null-terminated traversal.

Each byte is read as an unsigned eight-bit value. Integer promotion occurs before the subtraction used by the condition. The promoted difference from 0x61 is compared with 0x1A as a signed integer. Consequently, all input values from 0x00 through 0x7A satisfy the condition, including values below 0x61.

The exact transformation is:

| Input byte | Action and resulting byte |
|---|---|
| 0x00–0x1F | Subtract 0x20; conversion back to eight bits yields 0xE0–0xFF |
| 0x20–0x7A | Subtract 0x20; result is 0x00–0x5A |
| 0x7B–0xFF | No direct write; byte remains unchanged |

For example, 0x00 becomes 0xE0, 0x20 becomes 0x00, 0x41 becomes 0x21, and 0x61 becomes 0x41.

The subtraction is within the promoted integer range; the negative results for inputs below 0x20 are reduced modulo 256 when stored as bytes. This is not an unsigned lower-and-upper-bound test.

Although lowercase ASCII letters become uppercase letters, uppercase ASCII letters, punctuation, control bytes, and zero bytes can also change. The operation may introduce or remove zero bytes. No ordinary string semantics may be inferred from that fact.

### Classification comparisons

After transforming all 32 bytes, the function calls F-fca51b715561 with:

- The field beginning at working-buffer offset 48.
- A first constant reference address.
- Length 9.

A zero result selects classification 5 and suppresses the second comparison.

Otherwise, the same dependency receives the field address, a second reference address, and length 10. A zero result selects classification 2. Two nonzero results reject the record.

The first reference address is one byte beyond its named constant base. Exact reference bytes and comparison semantics are unavailable. Comparison operates after the established transformation; it must not be modeled as comparison against the original field or against a presumed ordinarily uppercased field.

### Accepted record extraction and completion

For either accepted classification, the function performs the following externally observable sequence:

1. Call F-4055be9c3f18 with working-buffer offset 8.
2. Call F-b81910dbb229 and write its 32-bit result to the slot’s first field.
3. Call F-4055be9c3f18 with working-buffer offset 12.
4. Call F-b81910dbb229 and write its 32-bit result to the slot’s second field.
5. Increment the accepted count.
6. Write classification 5 or 2 to the slot.

No second-field nonzero requirement applies on this path. Dependencies may change state between these operations; the contract does not infer that both retrieved values come from an immutable snapshot.

Scanning stops immediately after eight descriptors have been accepted, or when the clamped record range is exhausted. Rejected records consume record numbers but not output slots.

On completion, discriminator 2 is written before the final 32-bit count, and the function returns zero.

## Side effects

### Output and publication

The function initially clears only the discriminator and count. Descriptor fields are written before final metadata publication, with no rollback.

After a later record-read failure, previously written slots remain without final publication by this function. Metadata would remain at its initialized values only if dependencies and aliasing have not changed it. Consumers must not treat leftover slots as a completed result.

No atomic publication, synchronization, or concurrent-reader safety is established.

### Working-buffer mutation

Callbacks receive the working buffer and may modify it.

The record-oriented path directly transforms all 32 bytes of every processed classification field according to the promoted-integer rule above. This includes rejected records and bytes following any zero byte. Bytes from 0x00 through 0x7A are written with transformed values; bytes from 0x7B through 0xFF are not directly written by this step.

Records after the eighth accepted descriptor are neither requested nor transformed. The four-entry path contains no direct working-buffer writes.

### External state

The function reads the marker displacement and external error values. No direct global writes are shown.

Dependencies may alter external state, callback storage, or either buffer. A callable entry is fetched again for later record invocations; immutability of the returned callback object is not guaranteed.

## Dependencies

| Opaque ID | Established use | Unresolved behavior |
|---|---|---|
| F-fbcf824e7197 | Returns an object whose first entry supplies a callable address; invoked initially and again for record-oriented processing | Complete prototype, selector meaning, object lifetime, ownership, side effects |
| F-4055be9c3f18 | Receives working-buffer addresses at offsets 4, 8, and 12 before retrieval calls | Conversion rules, memory effects, state changes, failure behavior |
| F-b81910dbb229 | Supplies 32-bit results used as a count or descriptor fields | Byte order, state coupling, internal error reporting, side effects |
| F-fca51b715561 | Receives a field address, reference address, and explicit length; zero chooses a match branch | Reference contents, comparison rules, memory and global effects |

The dynamically selected callback has no separate supplied opaque ID. Its visible invocation forms are a one-argument initial operation and three-argument numbered-record operations.

## Acceptance examples

These are conditional static examples, not executed tests. Unless stated otherwise, buffers are valid, stable, and non-overlapping, and dependencies do not modify unrelated state.

1. **Initial callback failure.** A nonzero result selects the external initial-operation error value. Direct metadata initialization precedes the callback; no marker parsing follows.

2. **Mixed four-entry input.** With the displaced marker matching, classifications 0x0C, 0x07, 0x00, and 0x3F, and nonzero second fields, yield classifications 3, 2, and 1 in consecutive slots. Final discriminator is 1 and count is 3.

3. **Empty recognized four-entry result.** Four zero second fields yield discriminator 1 and count zero. Existing slot bytes are not cleared.

4. **Both markers match.** The four-entry path wins. The second resolver invocation does not occur.

5. **Neither marker matches.** Following a successful initial callback, the function returns the format-error value without directly populating descriptors.

6. **Zero record count.** Record 1 is acquired before the count is retrieved. A retrieved count of zero causes no field transformation or comparisons, then publishes discriminator 2 and count zero.

7. **Transformation boundaries.** A processed field containing bytes 0x00, 0x1F, 0x20, 0x41, 0x61, 0x7A, 0x7B, and 0xFF presents 0xE0, 0xFF, 0x00, 0x21, 0x41, 0x5A, 0x7B, and 0xFF at those positions to the subsequent comparison call. Transformation continues beyond zero bytes.

8. **First comparison precedence.** A zero first comparison result selects classification 5 without a second comparison. This remains true regardless of what the second comparison would have returned.

9. **Accepted zero-valued field.** A nonzero first comparison followed by a zero second comparison produces classification 2, even if the retrieved second descriptor field is zero.

10. **Unsigned count clamp.** A retrieved count of 256 or 0xFFFFFFFF establishes a scan limit of 255, not a negative or wrapped limit.

11. **Eight accepted records.** The eighth acceptance ends scanning. No later record is requested or transformed.

12. **Failure after partial extraction.** If two slots have been populated and the next record callback fails, those writes remain, final metadata publication is skipped, and the shared error value is returned.

## Uncertainties

The following remain **documented_only**:

- Numeric error values, their distinctness, and whether any equal zero.
- Target endianness, authoritative ABI declarations, and exact alignment requirements.
- Valid range and interpretation of the marker displacement.
- Exact comparison-reference bytes and comparison semantics.
- Full resolver and callback prototypes, including the differing resolver argument forms.
- Units and meaning of descriptor fields.
- Dependency effects on buffers, globals, callback objects, and ownership.
- Behavior outside valid-storage, stable-input, and dependency preconditions.
- Failure mechanisms other than the checked callback results.
- Intended purpose of the unusually broad record-field transformation.

An intent to uppercase ASCII letters would not change the established transformation. Matching outcomes for particular textual labels remain unresolved without reference bytes and dependency semantics.

## Evidence

This contract derives solely from the supplied static function representation and opaque dependency mapping. Prior prose and reviewer feedback are not treated as independent behavioral authority.

The representation establishes access widths and offsets, callback ordering, marker precedence, classification mappings, repeated four-entry reads, unsigned count clamping, the eight-descriptor limit, write ordering, and error-value selection.

The record-field rule follows from unsigned-byte loading, promotion to signed integer before subtraction, comparison of that signed difference with 26, and conversion of the stored result back to an eight-bit byte. Those operations establish transformation of inputs 0x00–0x7A, not merely lowercase ASCII letters.

Dependency implementations, concrete external values, reference contents, and runtime observations were not supplied. No runtime tests have been performed.
