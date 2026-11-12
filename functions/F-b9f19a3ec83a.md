<!-- Trusted provenance: F-b9f19a3ec83a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b9f19a3ec83a

## Purpose

F-b9f19a3ec83a admits a length-prefixed sequence into a mutable container, conditionally reusing an existing entry or inserting a new one. Local successful nonempty paths assign a one-based entry identifier. An admitted empty sequence returns success without local lookup or insertion.

Established behavior concerns byte lengths, storage records, free-list manipulation, optional reference increments, and calls to opaque dependencies. Text encoding, normalization, and comparison semantics are not established.

The 32-bit result is zero on accepted completion, −50 on local admission rejection, or a checked nonzero result propagated from a storage-adjustment dependency. Failure does not imply unchanged state.

## Interface

### Arguments and result

The function receives three 32-bit addresses:

1. **Container:** mutable state with the layout below.
2. **Sequence:** null, or an address whose first two bytes hold an unsigned 16-bit length.
3. **Identifier output:** null, or a writable 32-bit destination, subject to path-dependent restrictions.

For a nonnull sequence, the retained payload address is two bytes after the prefix. The retained byte length is twice the unsigned prefix, ranging from zero through 131,070 bytes. This length calculation fits in 32 bits. A null sequence produces zero byte length.

The function itself performs no terminator check. It passes the retained payload address and, for comparison, indexed lookup, and transfer, the retained byte length to opaque dependencies. Their termination requirements and readable-memory preconditions are unknown. A length argument does not establish that exactly that payload extent is sufficient for every dependency. The post-insertion dependency receives the payload address without an explicit length argument.

The prefix is read before the optional output reset. Subsequent operations retain the derived address and length rather than rereading the prefix.

A nonnull output receives zero before the first dependency call. Local successful nonempty linear reuse and insertion write an identifier without checking the output for null. Consequently, null output is not generally valid for nonempty use. Indexed lookup receives the output address directly; its handling of null is unknown.

### Container and storage layout

Offsets are bytes from the container address. Field names describe observed roles, not independently verified object semantics.

| Offset | Width | Observed role |
|---|---:|---|
| +0x04 | 32 bits | Flags; masks 0x01, 0x04, and 0x40 are examined |
| +0x08 | 32-bit address | Entry-array handle |
| +0x0C | 32-bit address | Reference-count-array handle |
| +0x10 | 32-bit address | Payload-storage handle |
| +0x1C | 32 bits | Entry extent |
| +0x20 | 32 bits | One-based free-entry head; zero means no listed entry |
| +0x24 | 32 bits | Available payload quantity |
| +0x28 | 32 bits | Payload-growth quantity |
| +0x2C | 32 bits | Entry-growth quantity |
| +0x30 | 32 bits | Admission-status field |

Each handle addresses a 32-bit word containing its storage base.

Entries occupy eight bytes. A committed entry contains a 32-bit payload offset followed by a 32-bit byte length. A locally marked free entry has first word 0x80000000; its second word is used as the next free identifier.

Reference counts occupy four bytes per entry. Identifier 1 selects the first entry and first reference-count word.

### Preconditions and arithmetic

The chosen path requires accessible container fields, handles, storage, and output destinations wherever accessed. Dependency-specific memory requirements are additional and unresolved. In particular, this function does not establish that a readable prefix and length-sized payload alone satisfy all callees.

There is no general local validation of free-list links, capacity consistency, payload offsets, output aliasing, or address validity.

Address arithmetic, scaled indices, growth requests, extent changes, and reference increments operate on 32-bit quantities without visible overflow checks. Safe use requires results to identify valid storage. The representation does not establish portable behavior for every overflowing signed operation or invalid address.

Signedness differs by operation: the prefix is unsigned; linear-search continuation and the available-payload comparison are signed; growth-threshold comparisons and initialization continuation are unsigned.

## Behavior

### Admission and retained decisions

The function derives the payload address and byte length, resets a supplied output, and calls F-0aff6a5050f5 with the container.

Admission requires, in order:

1. A nonzero dependency result.
2. A subsequently read zero value at +0x30.
3. A subsequently read flags value with mask 0x40 clear.

The checks short-circuit. Rejection returns −50, without undoing the output reset or dependency effects.

From the admitted flags value, the function retains whether mask 0x01 is set. That retained decision controls lookup eligibility, reference-array growth, insertion reference increments, and eligibility for post-insertion indexing. Later changes to mask 0x01 do not replace it.

Zero byte length returns zero after admission. There is no local lookup, insertion, or reference increment. The admission dependency may nevertheless have changed reachable state, including an aliased output.

### Existing-entry lookup

Lookup is attempted only when the retained 0x01 decision is set and subsequently examined entry and payload handles are both nonzero. The reference-count handle is not part of this preliminary check.

F-efaaaab570a3 is called before lookup. Mask 0x04 is read afterward to select the mode.

**Linear mode, mask 0x04 clear.** The function loads the payload base, reference-count base, and entry-array base, in that order. These bases remain retained across candidate dependency calls.

Candidates are visited in increasing zero-based index order. Continuation compares the index against a freshly read signed +0x1C. A nonpositive extent therefore visits no candidates under otherwise stable state. After an unsuccessful candidate, the retained entry pointer advances eight bytes.

For each candidate:

- F-88337f8f5d95 receives the candidate address. Zero skips further checks.
- After a nonzero result, the candidate’s then-readable second word must equal the retained byte length.
- If lengths agree, the unmapped comparison dependency receives the retained input payload address, retained payload base plus the candidate’s then-read offset, and retained byte length.

A **nonzero** comparison result is accepted as a match. Zero is not a match; conventional zero-for-equality semantics must not be imposed.

On the first match, the function increments the corresponding count through the retained reference-count base, writes the one-based identifier, calls F-2a517f9b9ef3, and returns zero. Neither eligibility nor comparison refreshes the retained storage bases or candidate pointer.

**Indexed mode, mask 0x04 set.** F-849b785f50ce receives the container, retained payload address, retained byte length, and output address. Any nonzero result is accepted as successful reuse. The function then calls F-2a517f9b9ef3 and returns zero. This branch performs no separate local identifier assignment or reference increment.

A lookup miss also calls F-2a517f9b9ef3 before insertion. Its result is ignored. Its mutations can affect insertion state; after successful reuse, they can alter previously written output or counts.

### Free-entry selection

Each insertion iteration begins with F-30a0e9a392e7, receiving the container and zero. The free head is examined afterward.

A nonzero head selects an entry through the current entry-array base using one-based indexing. The selected entry’s second word is read and installed as the new free head **before** the calculated entry address is checked for null. A null calculated address therefore does not prevent the preceding access or head update.

For a nonnull selected address, the identifier is derived from that address relative to a newly read entry-array base, using a three-bit right shift and then adding one. With consistent state and valid arithmetic, it reproduces the selected identifier. That identity is not guaranteed if intervening writes alter the base through aliasing.

F-ec8cbc7f302e then receives the container. Its 32-bit result is retained as a payload offset without a local sentinel or error check.

The selected entry pointer remains retained across this call. A changed entry handle does not redirect subsequent record writes unless payload growth occurs.

### Payload growth and commit

After offset acquisition, +0x24 is compared with the retained byte length as a signed 32-bit value. A negative available quantity is insufficient.

If insufficient:

1. The function samples +0x28.
2. The growth request is the unsigned maximum of that sample and the byte length.
3. If the sample is below 0x20000, twice that sample is written back to +0x28 before the dependency call. This is not a clamp.
4. F-7eff13797b9b receives the address of the payload-handle field and the retained request.

The request uses the sampled quantity, not its doubled replacement.

After the call, on both success and failure, the entry pointer is recomputed using the current entry-array base and retained identifier.

A nonzero growth result causes the refreshed entry to be marked 0x80000000, linked to the then-current free head, and installed as the new head. The growth result is returned unchanged. Offset-acquisition effects, growth-field changes, and other dependency mutations are not generally restored.

On growth success, the retained request is added to the then-current +0x24. Without growth, no entry-pointer refresh occurs.

The transfer destination is formed from the current payload-storage base plus the retained offset. F-70a021fc7c24 receives the retained source address, destination, and byte length. Its result is ignored.

After transfer returns, local writes occur in this order:

1. Reread +0x24 and subtract the byte length.
2. Write the byte length to the selected entry’s second word.
3. Write the retained offset to its first word.
4. Write the retained identifier to the output.

Transfer does not trigger another entry-pointer refresh. Record writes use the original selection pointer, or the pointer refreshed immediately after payload growth.

If the retained 0x01 decision is clear, the function returns zero. Otherwise, it obtains the current reference-count base **after the output write**, increments the selected count from its existing value, and rereads mask 0x04. If that mask is set, F-4cffcc8d6e9b receives the container and retained input payload address. Its result is ignored.

### Entry-array growth

If there is no free head, or selection produces a null entry address after the earlier link access, entry-array growth follows.

If unsigned +0x2C is below 0x1000, the field is doubled before requesting storage. Doubling can exceed the threshold; zero remains zero.

The growth process samples state independently:

| Stage | State used |
|---|---|
| Entry request | Eight times +0x2C read before the entry-array adjustment call |
| Reference request, if retained 0x01 is set | Four times +0x2C reread after entry growth |
| Entry rollback after reference failure | Negative eight times +0x2C reread after that failure |
| Initialization start | Current entry base and current +0x1C after required successful calls |
| Initialization continuation | +0x2C reread for each unsigned bound comparison |
| Initialization links | +0x1C reread for each link assignment |
| Final free head | Current +0x1C plus one |
| Final extent | Subsequently read +0x1C plus current +0x2C |

Each storage adjustment uses F-7eff13797b9b and the address of the corresponding handle field. Entry-growth failure returns immediately. Reference-growth failure requests the entry rollback, ignores its result, and returns the reference-growth error.

Initialization advances a retained pointer in eight-byte steps. Each visited entry receives first word 0x80000000. When the preceding entry pointer is nonnull, its second word receives the current extent plus the current zero-based initialization index plus one.

The last initialized entry’s second word is not explicitly written. Newly added reference counts are not explicitly initialized.

No single retained batch size governs the requests, rollback, and initialization. Dependency mutations can make them disagree. During initialization, retained pointer advancement coexists with repeated field reads; overlapping storage can affect subsequent samples.

After assigning the final head and extent, insertion repeats, beginning with another F-30a0e9a392e7 call. Zero growth still reaches these assignments. Local progress and termination are not guaranteed.

## Side effects

Established local writes include output resets and assignments, reference increments, free-head and free-link changes, entry records, growth quantities, and available-payload accounting.

There is no direct input write or input deallocation in the function. Input addresses are passed to opaque dependencies. This does not establish immutability, ownership transfer, copying semantics, release behavior, or any lifetime guarantee.

The function is not locally transactional. It removes a free entry before offset acquisition, changes growth quantities before checked failures, ignores rollback results, and commits an entry before optional post-insertion indexing.

Aliasing is not locally prohibited. Consequences include:

- An output reset can change payload bytes after the prefix has been read.
- A transfer can change +0x24 before the subsequent subtraction.
- An identifier write can change the reference handle or flags before their later reads.
- A free-head write can affect fields subsequently used to derive the identifier.

Retaining an address does not establish that dependencies keep its storage allocated or valid. No local memory-fault recovery is shown.

## Dependencies

| Opaque ID | Established local use |
|---|---|
| F-0aff6a5050f5 | Admission call; zero causes rejection |
| F-efaaaab570a3 | Called before lookup-mode selection |
| F-2a517f9b9ef3 | Called after lookup success or miss; result ignored |
| F-88337f8f5d95 | Candidate eligibility; nonzero permits further checks |
| F-849b785f50ce | Indexed lookup; nonzero is accepted as success |
| F-30a0e9a392e7 | Starts each insertion iteration with second argument zero |
| F-ec8cbc7f302e | Supplies the unchecked retained payload offset |
| F-7eff13797b9b | Storage adjustments; specified nonzero results are propagated |
| F-70a021fc7c24 | Transfer request using source, destination, and byte length |
| F-4cffcc8d6e9b | Conditional post-commit call using container and payload address |

The additional comparison dependency has no supplied opaque ID. Its observed arguments are two addresses and a byte length; nonzero is accepted as a match.

No dependency is assumed to preserve fields, handles, globals, reachable storage, or ownership. Lookup-surrounding calls suggest a guard, but synchronization is not established.

## Acceptance examples

These are conditional static criteria, not executed tests. Unless mutation is the subject, assume valid nonoverlapping storage, satisfied dependency preconditions, and unchanged unrelated state.

1. **Empty admission.** Null input or a zero prefix returns zero after successful admission, with no local entry creation. A supplied output remains zero if dependencies leave it unchanged. Failed admission returns −50 even for empty input.

2. **Maximum prefix and termination.** A prefix of 65,535 yields a retained byte length of 131,070. No local terminator check occurs. This does not establish that a buffer ending exactly there satisfies any dependency’s memory or termination requirements.

3. **Comparison polarity.** An eligible equal-length candidate is rejected when comparison returns zero. A nonzero result accepts it, increments its count, and assigns its identifier before F-2a517f9b9ef3.

4. **Linear relocation.** If eligibility changes handles while preserving old storage, local candidate accesses and matching count increments still use retained bases. A changed extent affects the next continuation check.

5. **No payload growth.** If offset acquisition changes the entry handle while preserving selected storage, record writes still target the original selected pointer.

6. **Payload growth and later relocation.** Growth refreshes the selected pointer from the current entry base. If transfer changes that handle again, record writes still use the post-growth pointer.

7. **Post-transfer accounting.** If transfer leaves +0x24 equal to 100 and the byte length is 12, the local subtraction writes 88, regardless of the earlier available quantity.

8. **Output-before-reference sampling.** With retained tracking enabled, an output assignment that changes the reference handle affects the base subsequently fetched for the increment.

9. **Payload-growth failure.** The selected identifier is returned to the free list through the refreshed pointer and current head. The growth error propagates; the growth-quantity change is not locally undone.

10. **Independent growth samples.** If +0x2C is 0x1000 for entry growth, then becomes 0x1001, the reference request is 0x4004. If reference growth fails after changing it to 0x1002, rollback requests −0x8010.

11. **Post-growth initialization.** Changes to entry base or extent by successful growth calls affect initialization’s starting address. Its bound uses subsequent +0x2C reads, not a retained request size.

## Uncertainties

- **documented_only — dependency memory contracts:** Termination requirements, readable extents, alignment requirements, and possible reads beyond the stated length are unknown. The payload-only post-insertion call supplies no explicit length.
- **documented_only — dependency semantics:** Comparison meaning, allocation units, relocation safety, initialization, ownership, release behavior, synchronization, and mutation limits remain unresolved.
- **documented_only — free-list termination:** The last initialized link is not explicitly cleared. A valid terminating link requires an external invariant or initialization behavior.
- **documented_only — reference initialization:** Incrementing existing words does not establish their initial values.
- **documented_only — arithmetic extremes:** No complete safe domain is established for address differences, shifts, multiplication, reference increments, or extent overflow.
- **documented_only — error namespace:** The symbolic meaning of −50 and possible overlap with dependency errors are unknown.
- **documented_only — concurrency:** Repeated field reads establish sampling points, not atomicity or thread safety.
- **Inferred intent:** The layout resembles a sequence pool with optional interning and reference tracking. That interpretation adds no behavioral guarantees.

## Evidence

Established statements derive from static inspection of the supplied function representation and dependency mapping: branch conditions, widths, access order, retained values, refreshed pointers, arguments, local writes, and checked results.

The absence of a local terminator check establishes only that local observation. Neither length-bearing calls nor the prior draft establish callee termination or readable-memory requirements.

Prior documentation and reviewer feedback are not independent evidence of dependency behavior. Unresolved properties remain documented rather than assumed.

No runtime tests, execution traces, allocation-failure experiments, concurrency checks, or memory-layout probes have been performed. Acceptance examples describe conditional consequences of the static representation, not observed executions.
