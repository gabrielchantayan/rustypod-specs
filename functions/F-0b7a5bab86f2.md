<!-- Trusted provenance: F-0b7a5bab86f2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0b7a5bab86f2

## Purpose

F-0b7a5bab86f2 traverses a linked sequence of records, publishes a per-record selection value into context-associated state, and dispatches recognized record tags to dependency functions. It also emits an entry event, paired events around three dispatch paths, and an end-of-sequence event.

The established behavior is orchestration: choosing values, accessing fixed record fields, sequencing dependency calls, and following links. The domain meaning of the records, event codes, resolved values, and context state is not established.

Normal completion occurs when the current record pointer is null. The function then returns a 32-bit zero. There is no explicit error return, cycle detection, or validation of the referenced objects.

## Interface

### Inputs

The function has three inputs:

1. **Context reference.** A reference to storage accessed as 32-bit words. The function directly uses:
   - Byte offset **0**: a value cached at entry and supplied as the first argument to several dependencies.
   - Byte offset **12**: a value cached at entry and supplied as the first argument to all event calls made by this function.
   - Byte offset **392**: a pointer-valued word read during each non-null record iteration. The function writes a 32-bit selection value at byte offset **20** from that pointer.

2. **Initial record reference.** Either null or a reference to a record containing the fields required by its dispatch path. The next-record reference is stored at byte offset **36**.

3. **Selection input.** A 32-bit integer. The value **99** is a sentinel: for each record, it selects the record’s own word at byte offset **4**. Every other value is used directly.

The supplied representation uses 32-bit words and 32-bit integer fields. Pointer-valued words are represented in those same word-sized slots. The specification does not establish portability to a different pointer width or ABI.

### Record layout used by this function

| Byte offset | Width | Use |
|---|---:|---|
| 0 | 32 bits | Dispatch tag |
| 4 | 32 bits | Per-record selection, read only when the selection input is 99 |
| 12 | 32 bits | Input to F-5f042ee1ac4e for tags 101 and 110 |
| 24 | 32 bits | Input to F-1a5398bcc6b3 for tags 99 and 100 |
| 28 | 32 bits | Input to F-1698e90c5d31 for tags 100 and 101 |
| 32 | 32 bits | Input to F-516ec4b4c789 for tag 101 |
| 36 | Pointer-valued 32-bit word | Next-record reference |

Fields not listed here are not directly interpreted by this function. Dependencies receiving the context or record reference may access additional storage.

### Output and preconditions

The only explicit return value is **32-bit zero**, reached after encountering a null current-record reference and making the end-of-sequence event call.

For execution to complete without an invalid memory access:

- The context must support its entry reads.
- Every non-null current record must support the fields actually accessed on that iteration.
- The context’s word at offset 392 must designate writable storage at offset 20 whenever a record is processed.
- Dependency-specific preconditions must hold.
- The sequence must eventually reach null, and every invoked dependency must return normally.

These are execution preconditions, not conditions checked by the function. A null initial record avoids the per-record state access and all record reads, but does not avoid context entry reads or event calls.

## Behavior

### Entry and cached values

Before checking whether the initial record is null, the function:

1. Reads and caches the context word at offset 12 as the **event value**.
2. Reads and caches the context word at offset 0 as the **lookup value**.
3. Calls F-2ea344827b50 with the five arguments: event value, **56**, **0**, **0**, **0**.

Both cached values are subsequently passed by value. This function does not reread their context slots, even if a dependency changes those slots. Caching does not establish that any object designated by either value remains unchanged or valid.

### Per-record selection and state publication

At the start of each iteration, a null current-record reference causes an end event and return, as described below.

For a non-null record:

- If the selection input equals **99**, the selection for this iteration is read from the record at offset 4.
- Otherwise, the selection is the original selection input. The record’s offset-4 field is not directly read for this choice.
- The function reads the pointer-valued context word at offset 392 and writes the selection as a 32-bit value at offset 20 in the referenced storage.
- Only after that write does it read the record’s tag at offset 0.

This publication occurs for every non-null record, including unrecognized tags and tag 110 records whose later lookup produces zero.

The publication target is obtained anew on each iteration. It is not cached across dependency calls. If a dependency changes the context’s offset-392 field, a later iteration can write to a different target.

The ordering also matters when storage aliases. For example, if the publication destination overlaps the current record’s tag or link, the later reads observe storage after the publication write, subject to intervening dependency effects.

### Tag 99

For tag **99**, the function performs the following sequence:

1. Calls F-2ea344827b50 with event value, **54**, **0**, **0**, **0**.
2. Calls F-d0ebdbf53b10 with the context reference and current-record reference.
3. Reads the current record’s word at offset 24 and calls F-1a5398bcc6b3 with the cached lookup value and that word.
4. Calls F-e66b115a837f with:
   - The context reference.
   - The result of F-d0ebdbf53b10.
   - The result of F-1a5398bcc6b3.
5. Calls F-2ea344827b50 with event value, **54**, **1**, **0**, **0**.

The selected value has already been written into context-associated state, but it is not an explicit argument to F-e66b115a837f.

The offset-24 field is read after F-d0ebdbf53b10 returns. It is not an entry snapshot of that field.

### Tag 100

For tag **100**, the sequence differs from tag 99 at the beginning:

1. Calls F-d0ebdbf53b10 with the context reference and current-record reference.
2. Calls F-2ea344827b50 with event value, **54**, **0**, **0**, **0**.
3. Reads the record’s word at offset 24 and calls F-1a5398bcc6b3 with the cached lookup value and that word.
4. Reads the record’s word at offset 28 and calls F-1698e90c5d31 with the cached lookup value and that word.
5. Calls F-361bb888720f with:
   - The context reference.
   - The result of F-d0ebdbf53b10.
   - The result of F-1698e90c5d31.
   - The result of F-1a5398bcc6b3.
   - The selection chosen for this iteration.
6. Calls F-2ea344827b50 with event value, **54**, **1**, **0**, **0**.

Thus, the first event with code 54 precedes F-d0ebdbf53b10 for tag 99 but follows it for tag 100. That difference is part of the established ordering.

### Tag 101

For tag **101**, the function:

1. Calls F-d0ebdbf53b10 with the context reference and current-record reference.
2. Calls F-2ea344827b50 with event value, **54**, **0**, **0**, **0**.
3. Reads the record’s word at offset 32 and calls F-516ec4b4c789 with the cached lookup value and that word.
4. Reads the record’s word at offset 12 and calls F-5f042ee1ac4e with the cached lookup value and that word.
5. Reads the record’s word at offset 28 and calls F-1698e90c5d31 with the cached lookup value and that word.
6. Calls F-b3c070ba0309 with:
   - The context reference.
   - The result of F-d0ebdbf53b10.
   - The result of F-1698e90c5d31.
   - The result of F-5f042ee1ac4e.
   - The result of F-516ec4b4c789.
   - The selection chosen for this iteration.
7. Calls F-2ea344827b50 with event value, **54**, **1**, **0**, **0**.

The three lookup calls occur in the order **offset 32, offset 12, offset 28**. Their results are supplied to F-b3c070ba0309 in a different order, as listed above.

### Tag 110

For tag **110**, the function reads the record’s word at offset 12 and calls F-5f042ee1ac4e with the cached lookup value and that word.

If the result is zero, there are no further tag-specific calls.

If the result is nonzero, the function:

1. Calls F-1e7ea6aa642e with a reference to a local **16-byte temporary region**, **4**, and **0**.
2. Calls F-f1d95153eae8 with the context reference, the nonzero lookup result, and **0**.
3. Calls F-7a1ceda22360 with the context reference, that same lookup result, the temporary-region reference, and four trailing zero arguments.
4. Calls F-6c7ac9769d41 with that same lookup result.

There are no code-54 event calls on this path.

The temporary region has no explicit initialization in the supplied representation before F-1e7ea6aa642e receives it. Its address is later passed to F-7a1ceda22360. The meaning of the value 4, the amount of temporary storage initialized, and the interpretation of that storage are not established.

The nonzero lookup result is reused after intervening calls. Its continued validity is a dependency contract requirement, not something this function checks.

### Other tags, traversal, and completion

Any tag other than **99**, **100**, **101**, or **110** causes no tag-specific dependency calls. The selection publication still occurs.

After the chosen path finishes, the function reads the current record’s next-record reference at offset 36 and uses it as the current record for the next iteration. This read occurs after all calls for the current record. The link is therefore not snapshotted before dispatch.

On encountering null, the function calls F-2ea344827b50 with event value, **102**, **0**, **0**, **0**, then returns zero.

There is no traversal limit. A cycle need not terminate. A dependency may affect traversal by changing links before they are read.

## Side effects

### Directly established effects

- One 32-bit selection write for each non-null record, before reading that record’s dispatch tag.
- One entry event call per invocation.
- One end-of-sequence event call on normal completion.
- A pair of code-54 event calls for each completed tag-99, tag-100, or tag-101 dispatch.
- Conditional access to a local 16-byte temporary region on the nonzero tag-110 path.
- Dependency calls in the orders specified above.

The function does not directly clear or restore the selection state after traversal. If no dependency changes it, the last publication remains. That outcome is conditional: dependencies receive references and may mutate state.

### Ownership and lifetime

The function contains no explicit allocation operation and no direct record-link rewrite. Neither fact proves that its dependencies allocate nothing or leave links unchanged.

No ownership transfer is established for the context, records, lookup results, or temporary-region reference. In particular:

- F-6c7ac9769d41 must not be described as a release operation solely because it is the final call on one path.
- F-1e7ea6aa642e must not be assumed to initialize all 16 temporary bytes.
- F-7a1ceda22360 must not be assumed to avoid retaining the temporary reference.
- Dependencies must not be assumed to preserve globals, context fields, records, or referenced storage.

The local temporary’s storage lifetime is limited to this invocation. Any permitted retention of its address would require a dependency contract compatible with that lifetime.

### Errors and interrupted execution

No dependency result is tested for an error except the zero/nonzero branch on the tag-110 lookup result. Other returned values are forwarded without an explicit validity check.

There is no explicit recovery, rollback, or finalization path. If a dependency fails to return normally, the remaining calls, end event, and zero return are not established. A code-54 starting event does not guarantee its matching ending event under interrupted execution.

## Dependencies

All meanings below are limited to how this function uses the dependencies.

| Opaque function ID | Established role here |
|---|---|
| F-2ea344827b50 | Receives five-argument event calls using cached context data and numeric codes 56, 54, and 102 |
| F-d0ebdbf53b10 | Receives context and current record; its result is forwarded on tags 99–101 |
| F-1a5398bcc6b3 | Receives cached lookup value and record offset-24 word |
| F-1698e90c5d31 | Receives cached lookup value and record offset-28 word |
| F-516ec4b4c789 | Receives cached lookup value and record offset-32 word |
| F-5f042ee1ac4e | Receives cached lookup value and record offset-12 word; its result controls tag 110 |
| F-e66b115a837f | Receives the tag-99 dispatch arguments |
| F-361bb888720f | Receives the tag-100 dispatch arguments, including the selection |
| F-b3c070ba0309 | Receives the tag-101 dispatch arguments, including the selection |
| F-1e7ea6aa642e | Receives the temporary-region reference, 4, and 0 |
| F-f1d95153eae8 | Receives context, the nonzero tag-110 lookup result, and 0 |
| F-7a1ceda22360 | Receives context, that lookup result, the temporary reference, and four zeros |
| F-6c7ac9769d41 | Receives the tag-110 lookup result after the preceding three calls |

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests. Unless otherwise stated, dependencies return normally and do not mutate storage relevant to the example.

1. **Empty sequence.** With a valid context and a null initial record, the observable calls are the code-56 entry event followed by the code-102 end event. There is no selection publication or record access. The return is zero.

2. **Explicit selection overrides the record field.** With selection input 7 and one unrecognized-tag record ending in null, the function writes 7 to the publication target, does not directly read the record’s selection field, emits no code-54 events, and returns zero after the end event.

3. **Sentinel selection is evaluated per record.** With selection input 99 and two records whose offset-4 words are 12 and −3, the publications are 12 and then −3. The negative value is stored as a 32-bit integer; there is no range restriction or conversion to a Boolean.

4. **Tag-99 ordering.** For one tag-99 record, the code-54 starting event occurs before F-d0ebdbf53b10. F-e66b115a837f receives the two dependency results in the specified order. The code-54 ending event precedes reading the next link.

5. **Tag-100 ordering.** For one tag-100 record, F-d0ebdbf53b10 occurs before the code-54 starting event. F-361bb888720f receives the offset-28 lookup result before the offset-24 lookup result, followed by the selected value.

6. **Tag-110 zero result.** If F-5f042ee1ac4e returns zero, none of the four nonzero-path calls occurs. Selection publication has nevertheless already happened.

7. **Mutable next link.** If a tag-specific dependency changes the current record’s offset-36 link, traversal uses the link read after dispatch, not its earlier value.

8. **Changed context slots.** If a dependency changes context offset 0 or 12, later calls from this function still receive the cached lookup or event value respectively. If it changes context offset 392, the next record’s publication uses the newly read target pointer.

## Uncertainties

- **documented_only — Domain semantics:** Record types, event meanings, lookup namespaces, and the purpose of the published selection are unknown.
- **documented_only — Dependency effects:** Mutation, allocation, ownership transfer, synchronization, callbacks, and nonlocal failure behavior are unspecified.
- **documented_only — Temporary format:** The 16-byte region’s internal layout and initialization coverage are unknown.
- **documented_only — ABI details:** The representation establishes 32-bit slots, but not a portable calling convention, byte order, or alternate-width layout.
- **documented_only — Concurrency:** No synchronization or concurrent-mutation guarantees are established.
- **Inferred intent:** Codes 56 and 102 appear to bracket traversal, while code 54 with indicators 0 and 1 appears to bracket selected dispatch work. These interpretations do not add guarantees beyond call ordering.
- **Inferred intent:** The selection input’s value 99 appears to request per-record selection. This description is supported by the branch behavior but does not identify the application-level meaning of that selection.

## Evidence

This contract is based solely on the supplied static function representation and the supplied mapping to opaque dependency IDs. It describes directly visible field accesses, argument forwarding, branch conditions, call ordering, and normal return behavior.

No dependency implementations, external type definitions, or runtime observations were provided. No runtime tests have been performed. Acceptance examples are static consequences under their stated conditions, not reports of observed execution.
