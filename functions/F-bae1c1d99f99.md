<!-- Trusted provenance: F-bae1c1d99f99; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bae1c1d99f99

## Purpose

F-bae1c1d99f99 requests storage and constructs a descriptor-derived representation using a heading, an opening-parenthesis byte, record content, selected separators, and a final formatting fragment. It first measures contributions from the descriptor and its records, then chooses one of two formatting regimes using a signed threshold of 50.

This is an established construction sequence, not an established text format. The supplied representation does not reveal the constant fragments or dependency implementations. It therefore does not establish exact output text, a closing parenthesis, null termination, or agreement between measured size and emitted content.

The only explicit local failure response is for a zero storage-request result: write a failure byte in the supplied context and return a packed result whose low word is zero. Other failure handling and resource ownership remain **documented_only**.

## Interface

### Arguments and accessed layout

The visible interface contains four arguments:

| Argument | Width and interpretation | Visible use |
|---|---|---|
| Context address | 32-bit address-like value | On zero allocation, receives a one-byte write at byte offset 30. |
| Descriptor address | 32-bit address-like value | Identifies the descriptor described below; also occupies the high word of the result. |
| Opaque value | 32 bits | No visible use. |
| Initial position | 32 bits | Initially assigned to local position storage, then overwritten before any visible use of that position. It does not supply an effective starting output offset. |

The descriptor has these directly accessed fields:

| Byte offset | Width | Meaning established by use |
|---|---:|---|
| 0 | 32 bits | Heading value supplied to measurement and append dependencies. |
| 4 | 32 bits | Record count, interpreted as signed for loop comparisons. |
| 8 | 32 bits | Record-array address. |

Records have a 20-byte stride:

| Record-relative byte offset | Width | Meaning established by use |
|---|---:|---|
| 0 | 32 bits | Primary value supplied to measurement and append dependencies. |
| 8 | 32 bits | Optional value; zero suppresses its associated processing. |

The remaining record words are not directly interpreted. Heading, primary, and optional values are opaque words. Their usage is compatible with text-related values, but their semantic types are not established.

### Return value

The result is a 64-bit concatenation:

- High 32 bits: the descriptor-address argument.
- Low 32 bits: the storage-request result, including zero on its explicit failure branch.

The result must not be reduced to a single returned output pointer. Allocation failure does not clear its high word.

The function visibly performs no release. Ownership, lifetime, allocation extent, and the appropriate release operation are **documented_only**.

### Preconditions and arithmetic boundaries

Intended operation requires valid storage for every descriptor and record access, every dependency access, and every output write. The context byte at offset 30 must be writable if the storage request returns zero.

No visible checks establish:

- Descriptor or record-array validity.
- Record-array extent.
- Validity of values passed to dependencies.
- Nonnegative or bounded record counts.
- Safe capacity or position arithmetic.
- Sufficient allocation extent.
- Successful formatting, valid termination, or safe dependency-produced positions.

The represented arithmetic and address-like values are 32-bit; no widening or overflow checks are visible. Signed loop and threshold comparisons are established. Exact behavior on arithmetic overflow is not established by this representation and remains **documented_only**.

## Behavior

### Measurement and regime selection

The function reads the record-array address before beginning measurement. That traversal retains its selected base and advances through records at 20-byte intervals. Each loop condition compares a signed index, initially zero, with the descriptor count read for that condition.

Each visited record contributes the return from F-3189985170da applied to its primary word. After that call returns, the optional word is read. If nonzero, F-1cb585febd84 is called with **no visible explicit arguments**, and its return plus one is added to the running contribution.

The optional sizing call must not be described as receiving the optional word. That word controls whether the call occurs, but no explicit argument is shown.

After record measurement, the heading is read and passed to F-3189985170da. Its return is combined with the record contribution to produce the measured total.

The measured total is compared as a signed 32-bit value:

- Less than 50 selects the short regime.
- At least 50 selects the long regime.

Each regime selects three fragment addresses: an initial record separator, a subsequent record separator, and a final fragment. Their literal bytes are unavailable. The threshold excludes both the count-based allowance and the fixed capacity allowance.

Requested capacity is formed from the measured total, six times the descriptor count read for this calculation, and 35. For values without overflow, this is the measured total plus six times the count plus 35. The count used here is a separate descriptor read, not necessarily the count observed by any earlier loop condition.

A count that remains negative prevents both record traversals from executing. It nevertheless participates in capacity arithmetic and is not a validated empty-input encoding.

### Storage request and explicit failure

F-391e3d30e634 receives the computed capacity.

If it returns zero, the function writes byte value 1 at context offset 30 and returns the packed descriptor-address/zero result. No output-construction calls follow. Measurement calls and their effects have already occurred; no rollback is visible.

A nonzero result is treated as the output base. There is no corresponding direct success-path write clearing the context failure byte.

### Heading construction and position exposure

On a nonzero storage result, F-ea27adaa2931 is first called with two visible arguments: capacity and output base. The evidence does not establish that this clears storage, creates an empty string, or performs an ordinary formatted write.

F-1cb585febd84 is then called with the output base. Its return replaces the local position, superseding the fourth argument.

The heading is read again and supplied to F-aa839370a689 together with the output base and the address of the local position. This makes the position address available to that dependency. No contract establishes that the dependency declines to retain it or prevents later dependencies from accessing it.

After F-aa839370a689 returns, the function uses the position then present to write byte value 40, an opening parenthesis. It next rereads the descriptor’s record-array address and increments the position by one. The construction traversal therefore starts from a newly read record-array base, not from the measurement traversal’s ending address.

### Record construction and mutable-position rules

Construction uses signed index-versus-current-count comparisons. Its first record uses the regime’s initial separator; later records use its subsequent separator.

The position is shared mutable storage after its address has escaped. Each later use must be interpreted at its actual point of use. A dependency need not receive the position address in its current argument list to affect it through a previously retained address. Preservation is not established.

**Separator production.** F-ea27adaa2931 receives remaining capacity, an output destination formed from the current position, and the selected separator.

**Separator measurement.** After that formatting call returns, the function recomputes output base plus the position then present and passes this recomputed address to F-1cb585febd84. It does not retain and reuse the preceding formatting destination. Consequently, the measured address can differ from the address supplied for separator production.

After F-1cb585febd84 returns, its result is added to the position present **after that dependency returns**. The update does not necessarily use either the pre-formatting position or the position used to form the measurement argument.

**Primary content.** The current record’s primary word is then read and supplied to F-aa839370a689 with the output base and position address. Subsequent processing uses whatever position exists after that call.

**Optional content.** The optional record word is read after the primary-content call. If zero, there is no optional space, formatting call, or optional-length call.

For a nonzero optional word, the function writes byte value 32, a space, at the current output position and advances the position by one. F-ea27adaa2931 receives the remaining capacity and destination corresponding to that advanced position, a fixed formatting fragment, and the captured optional word.

The function subsequently calls F-1cb585febd84 with that captured optional word—not with an output destination and not with the formatting call’s return. The formatting dependency may already have modified the escaped position before this length call. The length dependency may modify it again. The returned length contribution is added to the position present after the length dependency returns.

The record address advances by 20 bytes, and the selected separator becomes the subsequent-separator fragment.

These rules do not establish that position tracks actual emitted bytes. Both the dependency semantics and preservation of the escaped position are unresolved.

### Final production

After the construction loop ends, F-ea27adaa2931 receives:

- Remaining capacity computed from the position then present.
- Output base plus that position.
- The fixed formatting fragment also used for optional content.
- The regime-selected final-fragment address.

No subsequent position update, direct terminator write, formatting-result check, or output validation is visible. The function returns the packed descriptor-address/output-base result.

## Side effects

### Direct writes

The established direct external writes are:

- Byte value 1 at context offset 30 on a zero storage result.
- Byte value 40 at the post-heading output position on the success path.
- Byte value 32 for each optional value processed during construction.

The local position is also updated and exposed by address to F-aa839370a689. Dependencies may write output, modify reachable structures, retain addresses, or affect other state; their implementations do not establish limits here.

No direct descriptor or record-array writes are visible.

### Read ordering and mutation exposure

The function does not obtain a coherent descriptor snapshot:

- Each traversal starts with its own record-array-address read.
- Loop conditions reread the count.
- Capacity calculation separately reads the count.
- Heading measurement and construction separately read the heading.
- Record fields are read during each traversal.
- Within a record, the optional word is read after the primary dependency returns.

Dependency mutation can therefore change later observations. For example, modifying the count can change traversal length, while modifying the descriptor’s record-array address does not replace the base already selected for an ongoing traversal. The later traversal can select that changed base.

Likewise, an escaped-position change can alter later destinations, remaining-capacity arguments, and additions. A call’s argument destination and the destination recomputed after its return are not interchangeable.

### Errors and resources

There is one visible storage request and no visible release. Formatting-like return values are ignored. No local cleanup, rollback, or additional failure indication is established for later dependency failures, truncation, or inconsistent positions.

Passing a remaining-capacity argument does not prove bounds safety. Direct byte writes are unchecked, subtraction may produce an unsuitable value, and dependency-driven position changes may invalidate the intended size relationship.

## Dependencies

| Opaque ID | Established call boundary | Unresolved contract |
|---|---|---|
| F-3189985170da | Receives a heading or primary-record word; its return contributes to measurement. | Units, range, errors, and side effects. |
| F-1cb585febd84 | Called without visible explicit arguments during optional measurement; elsewhere receives the output base, a recomputed output destination, or a captured optional word. Its return supplies an initial position or an added contribution. | Exact semantics, implicit inputs, errors, and possible modification of escaped position storage. |
| F-391e3d30e634 | Receives capacity; zero selects failure, nonzero becomes the output base. | Allocation extent, initialization, alignment, ownership, and release requirements. |
| F-ea27adaa2931 | Has two-, three-, and four-visible-argument calls associated with initialization and fragment production. | Formatting syntax, writes, termination, truncation, errors, and effects through retained addresses. |
| F-aa839370a689 | Receives output base, position address, and heading or primary word. | Emitted representation, position updates, bounds, pointer retention, and ownership effects. |

No dependency is assumed to preserve globals, input structures, context, or exposed local storage. None is assumed to acquire or release ownership.

## Acceptance examples

These are conditional static acceptance criteria, not observed executions. Dependencies are assumed to return normally, and storage must be valid for the described accesses.

### Stable zero count

If the count remains zero, neither record traversal executes. Heading measurement still occurs. Without overflow, capacity is its returned contribution plus 35.

On successful allocation, initialization, heading construction, the opening-parenthesis write, and final-fragment production still occur. No record separator or optional-field space is produced by the corresponding local paths. Exact resulting text is not specified.

### Threshold boundary

A measured total of 49 selects the short regime. A measured total of 50 selects the long regime. Count-based and fixed capacity additions do not change this selection.

### Stable two-record traversal

If the count remains two, measurement visits the first record, then the second, then the heading. Construction handles the heading before either record. Record addresses differ by 20 bytes. The first record receives the initial separator; the second receives the subsequent separator.

These ordering requirements do not imply dependency preservation of position or record contents.

### Changed position during separator processing

Suppose a separator-formatting call receives output base plus 7, and that dependency changes the escaped position to 11 before returning. The following F-1cb585febd84 call must receive output base plus 11, not output base plus 7.

If that length dependency returns 3 and leaves the position at 19, the subsequent addition produces position 22. It must not instead add 3 to 7 or 11.

### Changed position during optional processing

Suppose primary construction leaves position 22 and the optional word is nonzero. The direct space write occurs at output base plus 22, and the optional-formatting arguments use the advanced position 23.

If formatting changes the position to 31, the next length call still receives the captured optional word. If that call returns 4 and leaves position 40, the addition produces 44. Neither the pre-formatting position 23 nor the intermediate position 31 is guaranteed to be the addition’s base.

### Optional value absent

If the optional word read during a measurement visit is zero, that visit skips the no-explicit-argument sizing call and its additional one.

If the optional word read during a construction visit is zero, that visit skips its space and optional dependency calls. These decisions need not agree across traversals if the data changes.

### Zero storage result

If F-391e3d30e634 returns zero, context byte 30 is written with 1. Construction does not occur. The packed result retains the descriptor address in its high word and contains zero in its low word.

If allocation instead succeeds while that byte is already 1, this function contains no direct write clearing it. Dependency effects remain unresolved.

## Uncertainties

The following remain **documented_only**:

- Literal fragments, encoding, whitespace, final punctuation, and termination.
- Dependency semantics, implicit arguments, and error conventions.
- Whether measurement predicts actual output size.
- Whether dependencies retain the position address or modify it during later calls.
- Ownership, release API, lifetime, alignment, and allocation extent.
- Dependency mutation of descriptor, records, context, globals, or other reachable state.
- Overflow behavior and behavior for negative or extreme capacities.
- Behavior after invalid accesses, insufficient capacity, or dependency failure.

The inferred intent is a compact-versus-expanded textual representation of a heading and records. That interpretation does not establish literal output or justify assuming any preservation, bounds, or ownership contract.

## Evidence

Established static evidence comprises the accessed field widths and offsets, 20-byte record stride, signed loop and threshold comparisons, capacity expression, dependency-call boundaries, explicit byte writes, zero-allocation branch, and packed 64-bit result.

The supplied representation also establishes that separator measurement recomputes its destination after formatting returns, and that separator and optional-length additions use the position present after their respective length calls return. Because the position address has already escaped, those observations do not establish destination equality or position preservation.

No dependency implementation or fragment contents are supplied. No runtime tests have been performed. Exact output, memory safety, dependency preservation, and ownership have not been verified.
