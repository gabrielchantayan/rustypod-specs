<!-- Trusted provenance: F-9c0cc1713fca; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9c0cc1713fca

## Purpose

F-9c0cc1713fca conditionally replaces the logical contents of a caller-supplied sequence with records from a sentinel-terminated table obtained through a lookup dependency.

The established behavior has three outcomes:

- An initial prerequisite check returns zero: report status 3 without performing the lookup or explicitly modifying the destination sequence.
- The prerequisite succeeds, but lookup returns zero: report status 27 without explicitly modifying the destination sequence.
- Both dependencies return nonzero: reset the destination’s logical end, append table records until a record with a zero second word is encountered, and report status 0.

The result is a 64-bit packed value, not merely a status. Its low 32 bits contain the status; its high 32 bits contain a mutable working value initially taken from the second argument.

“Replacing a sequence” describes the apparent intent. Allocation, element ownership, and the exact effects of the insertion dependencies are not established by the supplied evidence.

## Interface

### Arguments

The function takes four arguments in the supplied 32-bit representation:

| Argument | Width and interpretation | Established use |
|---|---|---|
| Context | 32-bit integer/address value | The first dependency receives this value plus 40 bytes. |
| Initial result word | 32 bits, uninterpreted | Initializes the high half of the eventual return value and the first word of temporary record storage. |
| Destination descriptor | Pointer to 32-bit words | Used as a sequence descriptor with beginning, logical end, and capacity-boundary fields. |
| Lookup selector | 32 bits, uninterpreted | Passed to the lookup dependency with a second argument of zero. |

The destination descriptor has the following observed layout:

| Byte offset | Width | Interpretation |
|---|---|---|
| 0 | 32 bits | Beginning address |
| 4 | 32 bits | Logical end address |
| 8 | 32 bits | Capacity boundary address |

The visible sequence operations use 8-byte records, each consisting of two 32-bit words. The second source-table word is interpreted as an address-shaped value when copied into temporary storage, but this function does not dereference it.

The lookup result is treated as an address. A 32-bit table-base value is read at byte offset 32 from that result.

### Return value

The return is assembled as:

- Bits 0–31: status.
- Bits 32–63: the current first word of the temporary record storage.

Established low-half statuses are:

| Status | Meaning |
|---|---|
| 3 | Initial prerequisite dependency returned zero. |
| 27 | Lookup dependency returned zero. |
| 0 | Table processing reached a zero second-word sentinel. |

On either early return, the high half retains the second argument, subject to the ordinary assumption that dependencies do not corrupt unrelated local storage.

On successful processing of an empty table, the high half also retains the second argument.

For a nonempty table, the high half is loaded from each processed record’s first word. Because insertion dependencies receive the address of this temporary storage, their contracts are needed to determine whether they may change that word before it is returned.

### Preconditions

Defined completion requires valid memory and dependency behavior, including:

- The initial dependency must accept the context-derived argument.
- A nonzero lookup result must permit a 32-bit read at offset 32.
- Once both checks succeed, the destination descriptor must permit the required reads and writes.
- A nonempty destination range must support the observed 8-byte advancement to its end.
- The source table must provide readable record fields until a terminating zero second word is reached.
- Each insertion dependency must accept the descriptor, insertion-position value, and pointer to temporary record storage.
- Address and index calculations must remain meaningful for the represented execution model.

There are no visible explicit validations for null descriptors, invalid ranges, missing sentinels, insufficient readable storage, or arithmetic overflow.

## Behavior

### 1. Establish working storage and check the prerequisite

The function initializes temporary storage from the second, third, and fourth arguments. The first two temporary words subsequently serve as an insertion record.

It then calls F-c17dbd9c1a69 with the context value increased by 40 bytes.

If the dependency returns zero:

- Processing stops.
- F-667717e3fb3e is not called.
- The function performs no explicit destination-descriptor access.
- The returned low half is 3.
- The returned high half is the initial result word.

A nonzero result permits processing to continue. Its numeric value is otherwise unused.

### 2. Resolve the source table

The function calls F-667717e3fb3e with the lookup selector and a zero second argument.

If this dependency returns zero:

- Processing stops.
- The destination is not explicitly reset or populated.
- The returned low half is 27.
- The returned high half is the initial result word.

For a nonzero result, the visible access order is:

1. Read the destination logical end at descriptor offset 4.
2. Read the destination beginning at descriptor offset 0.
3. Read the source-table base from lookup-result offset 32.

The source-table base is captured before the destination reset. Subsequent iterations use that captured base; there is no visible repeated lookup of the table-base field.

The prerequisite and lookup dependencies execute before these destination reads. Their effects on shared or aliased state cannot be excluded.

### 3. Reset the destination’s logical end

If the captured beginning equals the captured logical end, this stage performs no explicit descriptor write.

Otherwise, the function includes a nominal record-copy loop whose traversal position starts at the captured end and whose condition requires that position to differ from the same captured end. Consequently, that loop performs no record copies in the supplied representation.

A following traversal advances from the captured beginning in 8-byte increments until it equals the destination’s logical end. The traversal body performs no visible element operation. In particular, it contains no visible destructor, release, or clearing call.

After that traversal terminates, the logical end is reduced by the captured byte distance between beginning and end, with that distance masked to a multiple of eight.

For a conventional valid range containing whole 8-byte records, this sets the logical end to the beginning. It does not explicitly overwrite the old backing records, change the capacity boundary, or release their contents.

The whole-record alignment is a behavioral precondition, not a checked condition. If the beginning cannot reach the end by repeated 8-byte advancement under the relevant address rules, this traversal has no established successful completion. The mask does not repair such a range: the end update occurs only after traversal finishes.

### 4. Process source records in order

The function starts with source-record index zero. Each record occupies eight bytes:

| Record offset | Width | Role |
|---|---|---|
| 0 | 32 bits | First record word; also updates the eventual high return half. |
| 4 | 32 bits | Second record word; zero terminates processing. |

For each index:

1. Read the second word.
2. If it is zero, terminate successfully without appending that record.
3. Otherwise, read the first word into temporary record storage.
4. Read the second word again into the second temporary word.
5. Read the destination’s current logical end.
6. Compare that end with the destination’s current capacity boundary.
7. Invoke the appropriate insertion dependency.
8. Advance to the next source-record index.

The sentinel check precedes the first-word read. Thus, a sentinel’s first word does not replace the high return half.

The second word is read twice for a processed record. The value used to decide continuation is therefore a distinct read from the value supplied for insertion. The function does not enforce equality between those observations if another actor or an earlier dependency changes the storage.

Records are processed in ascending table order. There is no visible sorting, filtering beyond the sentinel, duplicate elimination, or interpretation of either record word.

### 5. Select the insertion path

If the current logical end equals the current capacity boundary, the function calls F-c93a8f3c0af6 with:

- The destination descriptor.
- The captured current end.
- The address of the temporary record storage.

The caller performs no explicit end advance before this call.

If the current logical end differs from the capacity boundary:

- The function first writes logical end plus eight to descriptor offset 4.
- It then calls F-c3d28e48088c with the same three categories of argument.

The comparison is equality only. An end beyond capacity is not visibly rejected.

The early end advance is observable and occurs before F-c3d28e48088c executes. No rollback is visible if that dependency fails to return normally.

Neither insertion dependency has a visible return value used for status selection. The function reports status 0 after reaching the sentinel, regardless of any dependency result that is not otherwise reflected through memory or control flow.

## Side effects

### Directly established effects

On the successful-lookup path, the function may:

- Read and write the destination logical-end field.
- Read the destination beginning and capacity-boundary fields.
- Read the source table’s second words and, for processed records, first words.
- Overwrite temporary record storage.
- Invoke insertion dependencies once per nonsentinel record.

There is no direct visible allocation, deallocation, record-payload dereference, global write, or release of old destination elements.

### Dependency-mediated effects

All four dependencies may have effects not shown here. This contract does not assume that they preserve globals, preserve the source table, preserve descriptor fields, allocate storage, or release memory.

In particular:

- F-c93a8f3c0af6 may be intended to handle capacity exhaustion, but actual growth and allocation behavior are unknown.
- F-c3d28e48088c may be intended to construct or place a record at the supplied position, but its exact write and ownership behavior is unknown.
- Either insertion dependency can potentially modify the temporary record because it receives its address.
- Later iterations reread the current destination end and capacity boundary, so dependency changes to those fields can affect subsequent path selection.

### Ownership and lifetime

The caller retains responsibility for supplying valid descriptor and source-related storage unless dependency contracts establish another arrangement.

No ownership transfer of either record word is established here. Passing an address-shaped second word to an insertion dependency does not prove copying, reference counting, adoption, or release.

The temporary record is local working storage. Whether a dependency copies its contents or retains its address is unknown. Safe retention cannot be assumed.

## Dependencies

| Opaque ID | Observed call contract | Unknowns |
|---|---|---|
| F-c17dbd9c1a69 | Receives context plus 40; zero selects status 3, nonzero continues. | Meaning of the check and all side effects. |
| F-667717e3fb3e | Receives lookup selector and zero; zero selects status 27; nonzero is used as an address with a readable field at offset 32. | Lookup semantics, object lifetime, ownership, and side effects. |
| F-c93a8f3c0af6 | Called when destination end equals capacity; receives descriptor, current end, and temporary-record address. | Allocation, insertion, descriptor updates, temporary mutation, failure behavior, and ownership. |
| F-c3d28e48088c | Called after the caller advances end by eight on the non-equal-capacity path; receives descriptor, former end, and temporary-record address. | Record construction, temporary mutation, failure behavior, and ownership. |

## Acceptance examples

These are conditional acceptance scenarios derived from static evidence, not executed tests.

### Initial prerequisite fails

Given an initial result word of 0x12345678, if F-c17dbd9c1a69 returns zero:

- The return value is 0x1234567800000003.
- No lookup or insertion dependency is called.
- This function performs no explicit destination access.

This does not assert that the prerequisite dependency itself has no side effects.

### Lookup fails

If the prerequisite returns nonzero and F-667717e3fb3e returns zero:

- The low return half is 27.
- The high return half is the second argument.
- No destination reset or insertion occurs in this function.

### Empty source table replaces a populated destination

Given a valid destination containing three whole 8-byte records, and a source table whose first second word is zero:

- The destination logical end is reset to its beginning.
- No insertion dependency is called.
- Old backing bytes are not explicitly erased.
- The low return half is zero.
- The high return half remains the initial result word.

### One record with spare capacity

Given an initially empty destination whose end differs from capacity, followed by one nonsentinel source record and a sentinel:

- The record’s second word is checked before its first word is loaded.
- The destination end advances by eight before F-c3d28e48088c is called.
- The dependency receives the prior end as its insertion position.
- If the dependency leaves the temporary first word unchanged, the high return half equals that record’s first word.
- The low return half is zero.

### One record at capacity

If a nonsentinel record is encountered when end equals capacity:

- F-c93a8f3c0af6 is called.
- This function does not advance end before that call.
- Any successful insertion or capacity change depends on the dependency’s behavior.

### Sentinel first word is irrelevant

If the final table entry has a nonzero first word and a zero second word:

- That entry is not appended.
- Its first word is not loaded by the visible table-processing path.
- It does not replace the high return half.

### Insertion changes the temporary word

If the last insertion dependency changes the temporary first word before returning, and the next table entry is a sentinel:

- The changed word becomes the high return half.
- The wrapper does not restore the original source first word.

### Misaligned destination range

If beginning and end differ by a distance not reachable through 8-byte advancement, successful reset is not established. Acceptance must not require truncation into a valid empty range; the visible traversal occurs before the masked subtraction.

## Uncertainties

The following are **documented_only**:

- The semantic meaning of the context, selector, record words, and numeric statuses.
- Whether the destination is a conventional growable sequence beyond its observed three-word layout.
- Whether insertion dependencies allocate, copy, construct, retain, release, or transfer ownership.
- Whether dependencies preserve global state, descriptor state, source memory, or temporary record contents.
- Behavior under concurrent mutation, invalid pointers, inaccessible memory, dependency exceptions, or nonreturning calls.
- Exact machine-level behavior for signed index overflow and out-of-range address arithmetic. The supplied representation uses 32-bit integer arithmetic, but it does not justify declaring every overflowing expression to wrap safely.
- Whether the nominal copy loop and empty traversal reflect optimized container operations or artifacts of source recovery. Their inferred origin does not establish additional element destruction or copying.

The apparent intent is to clear and refill a destination sequence. That intent must not override the established ordering, packed return value, sentinel behavior, or unknown dependency contracts.

## Evidence

This specification is based exclusively on the supplied static function representation and dependency-ID mapping.

Established evidence includes the two early-exit checks, the three-word destination layout, lookup-result offset 32, 8-byte record stride, second-word sentinel, logical-end update ordering, insertion-path selection, and 64-bit return concatenation.

No dependency implementations, authoritative type declarations, runtime traces, or execution results were supplied. No runtime tests have been performed.
