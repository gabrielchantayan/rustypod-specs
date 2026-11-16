<!-- Trusted provenance: F-8ecc8cf8c7e7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8ecc8cf8c7e7

## Purpose

F-8ecc8cf8c7e7 conditionally initializes an object from binary input. It processes a directory, optionally transfers a selected name, retains a selected mapping block, processes metadata, follows nested positions, derives three metrics, and marks the object initialized.

Initialization is not transactional. Failure can leave earlier object writes intact. The common failure path clears the initialized flag, issues selected release requests, and invokes a failure operation.

The structures suggest font-related data, but that is **inferred intent**, not an established format contract. The behavior below follows the supplied static representation without assuming an external format or dependency implementation.

## Interface

The function accepts one object reference and returns no value. The represented model has 32-bit addresses and 32-bit object words. Offsets are byte offsets; narrower stores preserve surrounding bytes except for additional dependency effects.

| Object offset | Access width | Observed role |
|---|---:|---|
| 0 | 32 bits | Dispatch-table reference |
| 6 | Pointer argument | Passed to F-fa06621395af with numeric argument 64; write extent is unknown |
| 0x48 | 32 bits | Zero-extended record byte |
| 0x4c | 8 bits | Projection of three metadata flag bits |
| 0x4d | 8 bits | Record byte |
| 0x4e | 8 bits | Explicitly cleared |
| 0x50 | 16 bits | Zero-extended record byte |
| 0x52, 0x54, 0x56 | 16 bits each | Arithmetic results |
| 0x60 | 32 bits | Directory lookup’s position output |
| 0x64 | 16 bits | Nested-header field |
| 0x68 | 32 bits | Nested input position plus eight |
| 0x6c, 0x6e | 16 bits each | Intermediate-header fields |
| 0x70 | 32 bits | Nested-header field |
| 0x74 | 32 bits | Mapping-allocation result |
| 0x89 | 8 bits | Initialized flag |
| 0x9c | 32 bits | Finalization result |

The dispatch operation at table offset 0x2c gates initialization. The operation at offset 0x30 handles failure. Both receive the object.

Required accessible storage includes the object, dispatch table, input backing state, and every buffer and address subsequently accessed. The function supplies no general null, extent, consistency, or arithmetic-overflow validation.

The initialized flag is the direct completion indicator, not a returned status. Dependency effects may also alter object state.

## Behavior

### Initial state and gate

Before checking the flag or calling the gate, the function explicitly initializes every local directory-lookup position and length output slot to zero. These comprise five position/length pairs and the local length paired with object offset 0x60. It also initializes its tracked temporary-allocation references to zero.

Object offset 0x60 is **not** initialized here. A lookup that leaves that output untouched can therefore preserve an existing object value. By contrast, an untouched local output retains zero unless an earlier operation has changed the same slot.

If the initialized byte is nonzero, the function returns without a gate call or direct input operation. Otherwise it calls the gate. A zero gate result returns without common failure handling. A nonzero result starts initialization.

There is no direct flag clear at attempt entry. Gate-induced changes are not undone merely because the gate returns zero.

### Initial header and directory

The first input request is for 12 bytes at the current position; no initial seek occurs. For reads and seeks, the examined status is tested only against zero. Zero selects failure; nonzero permits continuation. No requested-length comparison is visible.

The header contains a 32-bit field followed by four 16-bit fields. They are converted in layout order using F-10cfc86a3233 and F-4a5c45c1e447, with each result written back.

The first 16-bit field supplies an unsigned count. A sample of that count determines both the category-zero allocation size and subsequent directory read size: count multiplied by 16. This is at most 1,048,560 bytes. A null allocation result fails. A zero-count request therefore depends on allocator and read behavior.

After the directory read, the header count is sampled again for the conversion-loop bound. Entries have a 16-byte stride and four 32-bit fields. Each field is converted and replaced before the next field is processed. The masked index update has ordinary ascending-index behavior over the unsigned 16-bit range.

The allocation/read count and conversion-loop count are not established to remain equal across intervening calls. Furthermore, every later lookup obtains its count argument from the then-current converted header field, rather than from a permanently preserved initial count.

### Global sampling and lookup order

The following independent opaque labels identify distinct global locations. Each selector is loaded at its own lookup call site, only if execution reaches that site.

| Global | Use |
|---|---|
| G-01 | Name-section selector |
| G-02 | Primary data-section selector |
| G-03 | Primary selector whose position output is object offset 0x60 |
| G-04 | Alternate data-section selector |
| G-05 | Alternate selector whose position output is object offset 0x60 |
| G-06 | Mapping-section selector |
| G-07 | 36-byte metadata-section selector |
| G-08 | Primary 56-byte metadata-section selector |
| G-09 | Alternate 56-byte metadata-section selector |
| G-10 | Initial value for the temporary name descriptor |

These are loads, not established immutable constants. Later loads can observe changes made by earlier dependencies. In particular, G-03 is loaded after the G-02 lookup returns; G-05 is loaded after the G-04 lookup returns.

The reached selector-load order is G-01, G-02, G-03, optionally G-04 and G-05, G-06, G-07, G-08, and optionally G-09. G-10 is loaded during name processing, after a matching candidate is found and before G-02.

### Name processing

The G-01 lookup receives the directory, the current header count, and initially zero local position/length outputs. Its return is ignored.

The function allocates the reported length with category zero, seeks to the reported position, reads the reported length, and invokes F-3a675d9f37dc. Null allocation or zero seek/read status fails. If lookup outputs remain untouched, these requests can use zero length and position; no independent “section missing” branch intervenes.

After processing, a 16-bit count at section offset 2 controls the candidate scan. Candidates have a 12-byte stride. For candidate zero, the tested 16-bit field is at offset 12, length at 14, and relative position at 16.

The first candidate whose tested field equals one is selected. The function then:

- Loads G-10 into the first 32-bit descriptor field and clears the next 32-bit field.
- Samples the unsigned 16-bit string length and relative position.
- Forms a position from the local section position, the unsigned 16-bit section field at offset 4, and the candidate-relative position.
- Requests writable descriptor storage for length plus one.
- Seeks to the calculated position and requests exactly the sampled length.
- Writes a zero byte after those bytes.
- Calls F-655893e4606f with the descriptor and zero.
- Obtains a value through F-44362605ab79.
- Calls F-fa06621395af with object offset 6, that value, and numeric argument 64.
- Calls F-289ec4311c64 for descriptor cleanup.

Length plus one ranges from 1 through 65,536. The writable-storage result is not explicitly checked before use.

A failed string seek or read causes descriptor cleanup before common failure handling. No matching candidate skips descriptor initialization and name transfer but does not itself fail initialization.

The name-section allocation is released with category zero, then its tracked reference is cleared. The numeric argument 64 does not establish a maximum write extent, truncation policy, or termination guarantee for the destination.

### Data selection and retained mapping

The G-02 lookup writes a local data position/length pair. The G-03 lookup writes its position directly to object offset 0x60 and its length to a local slot. Both calls occur.

Only if both return zero are the G-04 and G-05 alternate lookups issued, in that order, reusing the respective output slots. Thus either successful primary suppresses both alternates. Failed calls may leave outputs unchanged or alter them; that behavior is unknown.

The G-06 lookup supplies the mapping position and length; its result is ignored. The function allocates with category zero, seeks, reads, and invokes F-e8ca5c412224, with the usual checked allocation and status failures.

A 16-bit count at section offset 2 controls an eight-byte-stride descriptor scan. Descriptor zero has selectors at offsets 4 and 6 and a 32-bit relative position at offset 8. The first descriptor qualifies if its first selector is zero, or if its selector pair is three and one.

The selected address is the section base plus the relative position. If no descriptor qualifies, the selected address is zero and execution nevertheless attempts to read a length at selected-address offset 2. No graceful absence path exists.

Mapping retention has two distinct length samples:

1. Read the 16-bit field at selected-block offset 2 for a category-0x2e allocation.
2. Store the allocation result at object offset 0x74.
3. Read that 16-bit field again for the external copy-like call.

The destination argument is the allocation result; the source argument is the selected block. Allocation failure is not explicitly checked. The second length need not equal the first if allocation changes previously exposed section storage. No equality or destination-capacity check intervenes.

The enclosing mapping section is subsequently released with category zero, and its tracked reference is cleared.

### Metadata and record processing

The G-07 lookup result is ignored. The function seeks to its output position, requests 36 bytes regardless of reported section length, and invokes F-5882e0ac208c.

The G-08 lookup is followed by G-09 only if G-08 returns zero. The function seeks to the resulting position, requests 56 bytes regardless of reported length, and invokes F-754c6b0c279c.

The directory allocation is then released and its tracked reference cleared.

Next, the function seeks to the selected local data-section position, reads eight bytes, and converts its two 32-bit fields. It requests one 48-byte record and samples the second converted header field as an unsigned 32-bit loop count after that read returns.

For each index below the sampled count, conversion accesses a record at a 48-byte stride:

- Four 32-bit fields at offsets 0, 4, 8, and 12.
- Two 16-bit fields at offsets 40 and 42.

Only one 48-byte input read is established. Counts above one access beyond that explicitly read record; safe storage and initialized contents are not established. Count zero skips conversions but still permits later use of the first record’s first 32-bit field.

### Nested positions and publication

The first record’s first 32-bit field plus the local data-section position supplies the next seek position. An eight-byte intermediate header is read and converted in order: two 16-bit fields, then one 32-bit field.

Its two 16-bit fields are stored at object offsets 0x6c and 0x6e. The next position is then calculated from the current first record field, current local data position, and converted intermediate 32-bit field. It is not established to use an immutable copy of the earlier sum.

The function seeks there and reads another eight-byte header, converting the same three field widths. Direct publication then proceeds in this order:

1. Nested first 16-bit field to 0x64.
2. Calculated nested position plus eight to 0x68.
3. Nested 32-bit field to 0x70.
4. Record byte 45, zero-extended, to 0x48.
5. Low three bits of the processed 56-byte buffer’s 16-bit field at offset 44 to byte 0x4c.
6. Record byte 46 to byte 0x4d.
7. Zero to byte 0x4e.
8. Record byte 18, zero-extended, to 16-bit field 0x50.
9. Three arithmetic results to 0x52, 0x54, and 0x56, respectively.
10. F-d2e51f07aabd’s result to 0x9c.
11. One to the initialized byte.

The nested header’s second 16-bit field is converted but not directly published.

### Metric sampling and arithmetic

After the preceding metadata stores, the function samples the unsigned 16-bit divisor from processed 56-byte-buffer offset 18. It caches that divisor and its integer half, discarding the remainder, across all three arithmetic calls.

For each call separately, it reloads the corresponding signed 16-bit operand from processed 36-byte-buffer offsets 4, 6, and 8, and reloads record byte 45 as an unsigned scale:

| Call | Numerator before 32-bit representation | Destination |
|---|---|---|
| First | First operand × freshly loaded scale + cached half-divisor | 0x52 |
| Second | Negated second operand × freshly loaded scale + cached half-divisor | 0x54 |
| Third | Third operand × freshly loaded scale + cached half-divisor | 0x56 |

F-870c4fcf7295 receives each numerator and the cached divisor. Each result is stored before the following call.

Operand products and rounding additions fit signed 32-bit mathematical range for these widths. Mixed unsigned arithmetic nevertheless matters for negative numerator bit patterns; their interpretation by F-870c4fcf7295 is unknown. There is no zero-divisor check.

Earlier arithmetic calls may change the exposed metadata or record storage. Consequently, later operands and scales need not match earlier samples, and no metric is unconditionally tied to the scale previously published at 0x48.

## Side effects

Direct effects include ordered reads and seeks, global loads, allocation/release requests, temporary-buffer conversions, descriptor operations, and object writes. Input position is not restored.

The mapping-allocation result is stored without first releasing an existing value at 0x74. There is no direct success-path release of that retained allocation. Eventual ownership, dependency-induced replacement, and release responsibility remain unknown.

Common failure handling, after an accepted gate, proceeds as follows:

1. Clear the initialized byte.
2. Release the tracked directory allocation if nonzero.
3. Release the tracked name allocation if nonzero.
4. Release the tracked mapping-section allocation if nonzero.
5. Inspect the then-current object field at 0x74; if nonzero, release it with category 0x2e and then clear the field.
6. Invoke the failure dispatch operation.

Tracked temporary releases use category zero. Failure before mapping replacement can release a pre-existing object value at 0x74.

There is no general rollback of name contents, lookup outputs, intermediate-header stores, or other object changes. Release, finalization, and failure operations are not assumed to preserve globals, retain ownership, or release any additional memory.

## Dependencies

The final three IDs below are document-local opaque designations for calls lacking supplied dependency identities.

| Opaque ID | Observed role; contract limit |
|---|---|
| F-a39fb6744487 | Read request; only zero/nonzero status is checked |
| F-0fd4363fa1f0 | Positioning request; coordinate convention unknown |
| F-638beded7699 | Allocation-shaped request with size and category |
| F-8349c252c168 | Release-shaped request with address and category |
| F-10cfc86a3233 | 32-bit conversion |
| F-4a5c45c1e447 | 16-bit conversion |
| F-105972290119 | Directory lookup with count, selector, and output locations |
| F-3a675d9f37dc | Name-section processing |
| F-e8ca5c412224 | Mapping-section processing |
| F-5882e0ac208c | 36-byte metadata processing |
| F-754c6b0c279c | 56-byte metadata processing |
| F-f5bbf100a1ee | Descriptor writable-storage request |
| F-655893e4606f | Descriptor operation with numeric argument zero |
| F-44362605ab79 | Obtains the name-transfer argument |
| F-fa06621395af | Receives destination pointer, obtained value, and 64 |
| F-289ec4311c64 | Descriptor cleanup |
| F-870c4fcf7295 | Arithmetic operation; signedness and division semantics unknown |
| F-d2e51f07aabd | Finalization; result stored without validation |
| F-U01 | Indirect entry gate at dispatch offset 0x2c |
| F-U02 | Indirect failure operation at dispatch offset 0x30 |
| F-U03 | External copy-like call; actual transfer contract unknown |

## Acceptance examples

These are conditional static expectations, not executed tests.

1. **Already initialized:** A nonzero initial flag suppresses the gate and direct input operations. Local output-slot initialization still precedes that check.

2. **Gate refusal:** With an initially zero flag and zero gate result, no common failure callback occurs. Gate effects are not rolled back.

3. **Untouched lookup outputs:** If the name lookup leaves its outputs untouched, its allocation request uses length zero and, if that succeeds, its seek uses position zero. An untouched position output at 0x60 instead preserves that object field’s existing value.

4. **Mixed primary results:** A successful G-02 lookup and failed G-03 lookup suppress both alternate calls.

5. **Changed selector:** If the G-02 lookup changes G-03, the next lookup loads the changed selector. There is no entry-time selector snapshot.

6. **First matching name:** Only the first qualifying name candidate is transferred. G-10 is loaded only after that match. The transfer’s numeric argument 64 alone establishes no destination bound.

7. **Different mapping lengths:** If allocation changes the selected length from 20 to 40, allocation was requested for 20 while F-U03 receives 40. No intervening consistency check rejects this.

8. **Changed metric scale:** If record byte 45 is 10 at publication and the first arithmetic call changes it to 20, the second call evaluates its numerator using 20. The direct publication at 0x48 is not repeated.

9. **Cached divisor:** If the first arithmetic call changes the metadata divisor, later calls still receive the previously cached divisor and half-divisor, while reloading their operands and scales.

10. **Later failure:** A failed fixed-size read after mapping retention clears the flag and releases the then-current nonzero 0x74 value before invoking F-U02.

11. **Finalization result zero:** If all preceding accesses and calls complete normally, a zero finalization result is stored at 0x9c and does not prevent the final initialized-byte store of one.

## Uncertainties

The following remain **documented_only**:

- Global initial contents and dependency-induced modifications.
- Exact format, selector meanings, conversion transformations, and byte order.
- Short-read semantics, positioning conventions, and unsuccessful lookup output behavior.
- Section bounds, embedded-offset validity, and consistency between sampled counts and allocated extents.
- Safe behavior for no accepted mapping, unchecked mapping allocation, or failed writable-storage acquisition.
- Safe record-count range beyond the single explicitly read record.
- F-U03’s actual copying behavior and allocation compatibility.
- Name encoding, destination write extent, truncation, and termination.
- Arithmetic signedness, rounding semantics, and zero-divisor behavior.
- Dependency ownership, global preservation, exception behavior, and non-returning behavior.

Positions and addresses use represented 32-bit arithmetic without explicit overflow checks. Overflowing mathematical sums are not guaranteed valid positions. Dependency mutation can also invalidate otherwise consistent input layouts.

## Evidence

**Established static evidence:** explicit initialization, branch conditions, global-load sites, call order, requested sizes, access widths, field offsets, repeated sampling, object stores, and cleanup paths in the supplied representation.

**Inferred intent:** descriptions such as input reading, seeking, allocation, release, copying, string handling, and font-like structure. These descriptions do not supply missing dependency guarantees.

**Unknown:** dependency implementations and the memory-layout guarantees needed to resolve the multi-record access ambiguity.

No runtime tests, dependency inspection, memory-safety validation, or external-format validation have been performed.
