<!-- Trusted provenance: F-d9b1de2df19f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d9b1de2df19f

## Purpose

F-d9b1de2df19f validates a directory-like sequence of fixed-stride entries against a stream-like object. It checks directory placement, decodes entries, applies selected extent checks, and recognizes qualifying combinations of entry types. Two selected entry types additionally require a minimum declared length and a matching value obtained from inside their referenced data.

Successful completion returns zero. Structural rejection returns 2. Nonzero statuses from dependencies are returned unchanged.

The interpretation as a directory validator is inferred intent. The established behavior is a sequence of field reads, arithmetic checks, dependency calls, type comparisons, and status returns. The supplied evidence does not identify the actual file format or the numeric values of its type markers.

## Interface

### Inputs

The function accepts two 32-bit address-valued arguments:

- **Descriptor address:** identifies a readable object containing:
  - At byte offset 4, an unsigned 16-bit entry count.
  - At byte offset 12, a 32-bit signed field used to derive the directory start.
- **Stream-object address:** identifies an object passed to every dependency. At byte offset 4, the function reads an unsigned 32-bit bound, interpreted here as the stream extent.

These are minimum accessed fields, not complete object definitions. Object sizes, alignment requirements, byte order, address validity, and the meaning of other fields are not established.

The directory start is derived by adding 12 to the descriptor field at offset 12. This derived value is retained for subsequent positioning and restoration operations.

The function uses a 16-byte directory-entry stride. The decoded entry state consumed by this function comprises:

| Relative position in decoded storage | Width | Use |
|---|---:|---|
| 0 | 32 bits | Entry type marker |
| 4 | 32 bits | Not inspected by this function |
| 8 | 32 bits | Referenced-data offset, represented as signed |
| 12 | 32 bits | Referenced-data length, represented as unsigned |

This is a layout of consumed local decoding storage. It is not proof that the encoded representation is four consecutive raw words or that the decoder performs no transformation.

### Result

The result is a 32-bit integer:

- **0:** all completed checks passed, and at least one accepted type combination was observed.
- **2:** an explicit structural or qualification check failed.
- **Another nonzero value:** a dependency reported that status, which is propagated unchanged.

A result of 2 cannot distinguish an explicit rejection from a dependency that itself returned 2.

### Marker roles

The supplied evidence refers to global values whose numeric contents are unavailable. This contract uses role labels:

- **Captured exemption marker:** read once at function entry and retained.
- **Live exemption marker:** read when the extent-check exception is evaluated.
- **Primary and secondary inspection markers:** identify entries requiring an interior-value check.
- **First and second companion markers:** independently satisfy the two halves of an alternative acceptance condition.
- **Expected interior value:** compared with the value returned by the interior reader.
- **Decode descriptor:** passed to the entry decoder.

These labels do not establish that their values are distinct, immutable, or semantically meaningful outside this function.

## Behavior

### Initial eligibility and positioning

Before evaluating directory eligibility, the function reads and retains the captured exemption marker, derives the directory start, and initializes three local recognition flags to false.

Processing proceeds only when both initial conditions hold:

1. The unsigned 16-bit entry count is nonzero.
2. The computed end of the directory does not exceed the stream object's unsigned 32-bit extent.

The directory-end calculation is the derived start plus 16 times the count. The count is read separately for the nonzero test and the end calculation; it is not captured into a single stable local count.

If either condition fails, the function returns 2 without calling a dependency.

When eligibility succeeds, the function calls F-e40a3ceddab2 with the stream object and the derived directory start. Any nonzero result is returned immediately. No entry is decoded after that failure.

### Entry traversal

Traversal begins with a zero-based unsigned 32-bit index. Before each iteration, the current index is compared with a fresh read of the descriptor's unsigned 16-bit count.

Consequently, the initial directory-size check does not establish a fixed iteration count. If a dependency changes the descriptor count, subsequent loop conditions can shorten or extend traversal. No second directory-wide bound check is performed.

For each entered iteration:

1. F-de70a5256fd2 is called with the stream object, the current decode descriptor, and writable local decoding storage.
2. A nonzero decoder status is returned immediately.
3. On decoder success, the function checks the decoded extent.
4. It classifies the entry and, for an inspection entry, performs additional positioning and reading.
5. Only after that path completes does the index advance.

The function depends on the decoder to populate the consumed decoding fields. There is no independent initialization or validation establishing their values if a decoder reports success without providing them.

### Decoded-extent check

For every successfully decoded entry, the function compares:

- The stream object's current unsigned 32-bit extent; and
- The decoded offset plus decoded length, calculated with unsigned 32-bit arithmetic.

If the computed end exceeds the extent, the function returns 2 unless the entry's type matches either the live exemption marker or the captured exemption marker.

The exemption affects only this extent rejection. It does not bypass later classification, minimum-length requirements, interior-value checks, or dependency errors.

The arithmetic is significant: the decoded offset is converted to unsigned for addition to the unsigned length. The sum wraps modulo \(2^{32}\). This is not an overflow-detecting range check. There is no separate requirement that the decoded offset be within the stream extent, and no explicit rejection of a negative signed representation of that offset.

### Classification and recognition

Under stable marker values, entries are classified in this priority order:

1. An entry matching either inspection marker is an **inspection entry**.
2. Otherwise, an entry matching the first companion marker sets the first companion flag.
3. Otherwise, an entry matching the second companion marker sets the second companion flag.
4. Other entries do not affect recognition.

The implementation loads a candidate inspection marker and performs further reads of the primary inspection marker during classification. Marker values are therefore not all captured at entry. The ordinary two-marker interpretation requires those values to remain stable during this decision.

The companion checks are mutually exclusive for an individual entry. If both companion markers have the same value, a matching non-inspection entry sets only the first companion flag. If a companion marker overlaps an inspection marker, the inspection path takes priority.

Recognition flags remain set once established. They do not short-circuit traversal: a later bad entry or dependency failure still causes failure after an earlier qualifying entry.

### Inspection-entry validation

For an inspection entry, the function sets the inspection-recognition flag and then requires the decoded unsigned length to be at least 54 bytes.

A length below 54 produces result 2 before any interior positioning call.

For a sufficient length, the function:

- Calls F-e40a3ceddab2 with the stream object and the decoded offset plus 12.
- Immediately propagates a nonzero positioning status.
- Calls F-10d2ce969adf with the stream object and writable local status storage.
- Immediately propagates a nonzero status written by that reader.
- Compares the reader's returned 32-bit value against the expected interior value read after the call.
- Returns 2 on mismatch.
- On a match, calls F-e40a3ceddab2 to position at the next directory slot.

The restoration target is the retained directory start plus 16 times the current index plus 16. It is computed from the original start and current index, not from an observed decoder position.

A nonzero restoration status is returned immediately.

No restoration call is attempted after a failed interior positioning operation, a nonzero reader status, or a mismatching interior value. The minimum declared length does not establish which bytes F-10d2ce969adf reads; that dependency's consumption width is unknown.

### Final acceptance

When traversal ends normally, the function returns zero if either:

- At least one inspection entry was recognized; or
- Both companion flags were set.

Otherwise it returns 2.

Every inspection entry encountered before normal loop completion must pass its checks. Finding one valid inspection entry does not permit a later invalid inspection entry to be ignored.

Likewise, observing both companion types does not bypass validation of later entries.

### Arithmetic boundaries

The count is unsigned 16-bit, so its product with 16 is at most 1,048,560 and does not itself overflow 32 bits.

The initial directory-end comparison uses unsigned 32-bit arithmetic after conversion of the derived start. Its addition can wrap. Equality with the stream extent passes.

The restoration-target arithmetic also uses unsigned 32-bit operations because it incorporates the unsigned iteration index.

Two additions are represented as signed 32-bit calculations in the supplied evidence:

- The descriptor's start field plus 12.
- The decoded offset plus 12 for interior positioning.

Portable signed-overflow behavior is not established. Inputs that overflow those additions are **documented_only**; this contract does not assign them an assumed successful or rejecting outcome.

## Side effects

The function directly maintains local decoded state, status storage, an index, and three recognition flags. It does not explicitly write descriptor fields, stream-object fields, or global marker values.

External effects occur through dependencies:

- Initial positioning.
- Entry decoding.
- Interior positioning and reading for inspection entries.
- Restoration to the next directory slot after a successful interior check.

These operations can change stream state, but their exact effects are not available. A zero status is what this function treats as success; the supplied body does not independently verify the resulting position.

There is no final positioning operation. On success, the final stream state is whatever the last completed dependency left behind. On failure, the function does not roll back prior operations.

No allocation, deallocation, explicit ownership transfer, or persistent retention of caller addresses is visible in this function. Dependencies receive borrowed-looking object addresses and pointers to local storage, but their ownership and retention rules are unknown. They must not be assumed to preserve globals, descriptor contents, stream metadata, or unrelated memory.

## Dependencies

| Opaque ID | Observed use | Result handling |
|---|---|---|
| F-e40a3ceddab2 | Receives the stream object and a requested position | Zero permits continuation; any nonzero status is returned |
| F-de70a5256fd2 | Receives the stream object, decode descriptor, and writable entry storage | Zero permits decoded-field inspection; any nonzero status is returned |
| F-10d2ce969adf | Receives the stream object and writable status storage; returns a 32-bit value | Written status is checked first; only a zero status permits value comparison |

Calling F-e40a3ceddab2 a positioning dependency and F-de70a5256fd2 a decoder describes their observed use, not verified internal implementations.

The local status storage passed to F-10d2ce969adf contains zero after successful interior positioning. If the reader leaves it unchanged, this function proceeds to compare the returned value. Whether leaving it unchanged is a valid reader behavior is unknown.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests. Unless stated otherwise, assume valid addresses, stable marker values and count, defined signed additions, populated decoder output, and successful dependencies.

1. **Zero entries:** A count of zero returns 2. No dependency is called.

2. **Directory exactly meets the extent:** A nonzero count whose computed directory end equals the extent passes the initial check. Final acceptance still depends on decoded entries.

3. **Directory exceeds the extent:** A computed directory end greater than the extent returns 2 before initial positioning.

4. **One valid inspection entry:** An entry matching an inspection marker, with length 54 or greater, an allowed extent, a matching interior value, and successful restoration yields zero if traversal ends without another failure.

5. **Inspection entry one byte too short:** A matching inspection entry with length 53 returns 2 without an interior positioning call.

6. **Both companions:** Two non-inspection entries that set different companion flags can produce zero without calling F-10d2ce969adf.

7. **Only one companion:** Entries setting only the first companion flag return 2 at normal completion, even if every decode and extent check succeeds.

8. **Unknown type:** An unrecognized type can pass its extent check and be traversed without rejection. It does not contribute to final acceptance.

9. **Exempt extent, insufficient qualification:** An entry beyond the declared extent whose type matches an exemption marker avoids that extent rejection. If it establishes no qualifying flag combination, normal completion still returns 2.

10. **Earlier success, later failure:** A valid inspection entry followed by a nonexempt out-of-extent entry returns 2, not zero.

11. **Reader status takes priority:** If the reader returns the expected value but writes a nonzero status, that status is returned and no restoration call occurs.

12. **Value mismatch:** With reader status zero but a mismatching value, the result is 2 and no restoration call occurs.

13. **Wrapped decoded end:** An unsigned offset of 4,294,967,280 and length 64 produce a wrapped end of 48. With an extent of at least 48, this particular extent comparison does not reject the entry. Subsequent behavior depends on classification and any positioning operations.

14. **Equal companion markers:** If both companion markers are equal and no inspection entry is recognized, repeated entries with that value set only the first flag. They do not satisfy the two-companion acceptance condition.

## Uncertainties

The following are **documented_only**:

- Numeric marker values, their distinctness, and their intended format meanings.
- The complete descriptor and stream-object layouts.
- Encoded-entry byte order and the decoder's transformation rules.
- Dependency cursor semantics, bounds checks, memory ownership, and retained pointers.
- The interior reader's byte consumption and error conventions beyond the observed status channel.
- Outcomes for invalid addresses, insufficient object storage, or signed-overflow inputs.
- Thread-safety and behavior under concurrent mutation.
- Whether dependencies mutate the descriptor, marker globals, or stream extent.

The entry count, stream extent, decode descriptor, and most marker values are read during processing rather than comprehensively snapshotted. The captured exemption marker and derived directory start are retained. A dependency-induced change can therefore affect later decisions without changing those retained values.

No distinctness assumption is imposed on markers. Overlap is resolved by the observed classification priority, not by presumed format design.

## Evidence

The established contract comes solely from static inspection of the supplied function body and dependency mapping.

Directly supported facts include the field offsets and widths, 16-byte stride, count rereads, unsigned extent comparisons, exemption logic, minimum length of 54, interior displacement of 12, dependency-call order, status propagation, recognition priority, and final acceptance condition.

Directory terminology, stream terminology, and semantic marker labels are inferred descriptions used to explain that behavior. They do not establish a particular file format.

Dependency implementations, runtime object contents, and actual executions were not supplied. No runtime tests have been performed.
