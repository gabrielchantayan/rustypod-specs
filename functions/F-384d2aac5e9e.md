<!-- Trusted provenance: F-384d2aac5e9e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-384d2aac5e9e

## Purpose

F-384d2aac5e9e performs a mode-dependent sequence of object updates and dependency calls, optionally notifies a related target, and conditionally performs final processing.

The established distinction is between a zero mode and every nonzero mode. Nonzero modes must pass a shared gate before proceeding beyond the initial call and byte clear. Both branches can reach notification and final processing.

“Preparation,” “notification,” and “processing” describe roles in the observed sequence, not established application semantics. Whether this operation controls activation, visibility, attachment, destruction, or another lifecycle transition remains unknown.

## Interface

### Inputs and result

| Input | Representation | Use |
|---|---|---|
| Main object | Address in the supplied 32-bit address model | Mutable fields, related addresses, and indirect-call table |
| Mode | Unsigned 32-bit integer | Zero/nonzero branch selection and notification arguments |
| Payload A | Opaque 32-bit value | Forwarded unchanged to F-6d16652f2d03 |
| Payload B | Opaque 32-bit value | Forwarded unchanged to F-6d16652f2d03 |

There is no explicit return value or reported error status. Observable outcomes consist of direct memory writes, calls, and effects produced by those calls.

The displayed invocation of F-6d16652f2d03 additionally supplies two opaque 32-bit ambient machine-context values. They are not explicit parameters of F-384d2aac5e9e. Their meaning, validity, and acquisition are unresolved; they must not be replaced with invented constants or promoted to ordinary caller-supplied inputs without further evidence.

### Main-object layout used here

Offsets are byte offsets. A word is 32 bits in the supplied representation.

| Offset | Width | Observed role |
|---|---:|---|
| 0x000 | 32 bits | Indirect-call-table address |
| 0x034 | 32 bits | Related-object address, captured before notification and accessed again after final processing |
| 0x03C | 32 bits | Candidate notification-target address |
| 0x048 | 32 bits | Flags |
| 0x080 | Interior address | Argument to F-6d16652f2d03 |
| 0x0A0 | 32 bits in the representation | Low eight bits control an optional indirect call |
| 0x0A4 | Interior address | Argument to F-2318d06eb0b2 |
| 0x1A0 | 32 bits | First optional source address |
| 0x1A4 | 32 bits | Second optional source address |
| 0x1C5 | 8 bits | Cleared on the zero-mode branch |
| 0x1C8, 0x1CC | 32 bits each | First pair of copy destinations |
| 0x1D0, 0x1D4 | 32 bits each | Second pair of copy destinations |
| 0x1D8 | 8 bits | Cleared after the initial call |
| 0x1E0 | 32 bits | Cleared after a successful nonzero gate |
| 0x1EC | 8 bits | Cleared after nonzero preparation calls |

Each nonnull source is read at byte offsets 0x030 and 0x034. A notification target is inspected at offset 0x078. A related object may be inspected at offset 0x048 and used for its table address at offset 0x000. The final indirect-call result is used for a 32-bit flags update at offset 0x048.

Byte clears do not imply clearing adjacent bytes. Passing an interior address does not establish the size of the region a dependency may access.

### Preconditions

There is no explicit null check for the main object. Every reached read, write, and indirect call requires suitable valid storage or a valid callable entry. Requirements are path-dependent.

Dependencies may impose additional preconditions or invalidate storage. This contract does not assume that any dependency preserves object fields, globals, ownership, or allocation validity.

## Behavior

The following order is established by the supplied static representation, assuming each reached call returns normally. It is not a guarantee of atomicity or an instruction-level memory trace.

### 1. Initial call and byte clear

The first action is F-dfb0b0ad73ab with a normalized Boolean argument:

- Zero for mode zero.
- One for every nonzero mode.

After the call returns, the byte at main-object offset 0x1D8 is cleared.

Thus, a nonreturning initial call prevents that clear. A later gate failure does not undo either the call or the clear.

### 2. Zero-mode preparation

For mode zero, the function performs these actions in order:

1. Clears the byte at offset 0x1C5.
2. Calls F-ecc1f4c34dbc with the main-object address.
3. Calls F-40696ba6c03c with the main-object address.
4. Calls F-2318d06eb0b2 with the main-object interior address at offset 0x0A4.

The nonzero-mode gate is not inspected on this path.

Next, F-a1a48bd3fbde is called with the main-object address. Its result is tested for zero, not for equality to one.

For any nonzero result:

- The current main-object table address is read from offset 0x000.
- The operation at table byte offset 0x01C is invoked with the main-object address.
- F-20460d1f31d3 is called with no explicit arguments.

The indirect operation is labeled F-000000000001 in this document. A zero query result skips both calls but does not skip the common notification and eligibility stages.

### 3. Nonzero-mode gate

For every nonzero mode, a shared gate is consulted after the initial call and 0x1D8 clear.

The represented accesses are ordered as follows:

1. Read the gate’s first 32-bit word.
2. If that observation is nonzero, read and retain its second 32-bit word. Otherwise, retain zero without reading the second word.
3. Read the first word again.
4. Return if the second observation of the first word is zero or the retained second-word value is zero.

With stable gate storage and contents, continuation requires both words to be nonzero. The two first-word observations are not represented as one snapshot. A second-word change after its read does not change the retained value.

Gate failure skips all remaining work, including the 0x1E0 clear, source copies, common notification, and final processing. No direct write to the gate is shown.

The synchronization, stability, and ownership of the shared gate and its reference are not established.

### 4. Nonzero-mode preparation and sequential copies

After a successful gate:

1. Write a 32-bit zero at main-object offset 0x1E0.
2. Call F-363f884d7ae3 with the main-object address.
3. Call F-a037f41af662 with the main-object address.
4. Clear the byte at offset 0x1EC.
5. Read the first source address from offset 0x1A0.
6. If nonzero, read its word at offset 0x030 and write that value to main-object offset 0x1C8; then read its word at offset 0x034 and write that value to offset 0x1CC.
7. Read the second source address from offset 0x1A4.
8. If nonzero, read its word at offset 0x030 and write that value to main-object offset 0x1D0; then read its word at offset 0x034 and write that value to offset 0x1D4.

Each source address is retained for its own pair of accesses. The second source address is not captured before the first pair completes.

These are sequential read/write pairs, not a simultaneous snapshot. Each destination receives the value observed at its immediately preceding source read. In particular:

- The first destination write of a pair can overwrite the storage used by the second source read.
- Writes in the first pair can affect data or the source-address field used by the second pair.
- A source address retained for a pair is not reread merely because an overlapping write changes its stored address field.

A null source skips its corresponding pair of writes; it does not zero the destinations. That omission does not establish preservation against other overlapping writes or dependency effects.

### 5. Common notification

After the applicable preparation branch and any zero-mode query-dependent calls, the function reads, in order:

1. The candidate target address at main-object offset 0x03C.
2. The related-object address at offset 0x034, retained for the later eligibility decision.

If the candidate target is nonnull, its address-bearing field at offset 0x078 is compared with the main-object address. Only an exact match causes F-df6da5bd408b to be called.

That call receives the target address and:

- One for mode zero.
- Zero for every nonzero mode.

The arithmetic distinguishes modes zero and one, and explicitly selects zero for unsigned modes greater than one. Modes such as two or 0xFFFFFFFF do not yield an externally passed negative or wrapped notification value.

A null target or mismatched backlink suppresses only this notification. Neither independently terminates the function.

### 6. Final-processing eligibility

After any notification returns, bit 0x10 of the main object’s 32-bit flags at offset 0x048 is tested.

- If set, final processing proceeds without reading the retained related object’s flags for this decision.
- If clear, the retained related-object address must be nonzero and its flags at offset 0x048 must contain bit 0x10.
- Otherwise, the function returns.

The related address used here was captured before notification. Changes to main-object offset 0x034 during notification do not redirect this eligibility check. However, the retained related object’s flags are read after notification, so changes to those flags can affect eligibility.

This distinction does not guarantee that the retained address remains valid.

### 7. Final processing and flag updates

On eligibility:

1. Set bit 0x20 in the main object’s 32-bit flags at offset 0x048, preserving the other bits observed for that update.
2. Call F-6d16652f2d03 with the main-object address, its interior address at offset 0x080, both payloads, and the two ambient values.
3. After that call returns, inspect the current related address at main-object offset 0x034.
4. If nonnull, test whether the low eight bits of the main object’s field at offset 0x0A0 are zero.
5. If both conditions hold, invoke an operation through the related object’s table at table byte offset 0x128.

This optional operation is F-000000000002. No explicit arguments are shown; a hidden receiver convention is not established. The related-address field is used again to resolve its table. The representation does not establish one retained address across that check and resolution.

The optional call is followed by F-000000000003, resolved through the main object’s current table at byte offset 0x05C and invoked with the main-object address.

Its result is treated as an address without a null check. Bit 0x40 is set in the 32-bit flags field at result offset 0x048, preserving the other bits observed there. This completes the represented normal path.

The final write can target the main object or another object. Neither identity nor ownership is established by the call site.

## Side effects

Direct effects comprise the listed byte clears, word clear, sequential copies, main-object bit-0x20 update, and final result-object bit-0x40 update.

Their important ordering constraints are:

- The initial dependency precedes every direct write.
- A nonzero gate failure still leaves the initial call and 0x1D8 clear already performed.
- Both nonzero preparation calls precede the 0x1EC clear and source copies.
- The main-object bit-0x20 update precedes F-6d16652f2d03.
- The final bit-0x40 update follows final processing and any optional related-object indirect call.

No direct allocation, release, ownership transfer, or retention operation is shown. Passing an object or interior address does not establish what a dependency may retain, release, or mutate.

No explicit rollback, locking, atomic update, exception handling, or recoverable error reporting is shown. Invalid storage and malformed indirect-call results are not converted into a specified error outcome. A dependency that does not return normally prevents subsequent represented actions.

## Dependencies

The direct dependencies use the supplied opaque IDs. The three numbered indirect IDs are document-local labels, not resolved implementation identities.

| Dependency | Visible call-site contract |
|---|---|
| F-dfb0b0ad73ab | First call; receives normalized nonzero-mode Boolean |
| F-ecc1f4c34dbc | Receives main object on zero-mode path |
| F-40696ba6c03c | Receives main object after F-ecc1f4c34dbc |
| F-2318d06eb0b2 | Receives main-object interior address 0x0A4 |
| F-a1a48bd3fbde | Receives main object; any nonzero result enables two calls |
| F-000000000001 | Main-object table offset 0x01C; receives main object |
| F-20460d1f31d3 | No explicit arguments; follows F-000000000001 |
| F-363f884d7ae3 | Receives main object after successful nonzero gate |
| F-a037f41af662 | Receives main object after F-363f884d7ae3 |
| F-df6da5bd408b | Receives backlink-matching target and zero-mode indicator |
| F-6d16652f2d03 | Receives main object, interior address 0x080, payloads, and ambient values |
| F-000000000002 | Related-object table offset 0x128; no explicit arguments shown |
| F-000000000003 | Main-object table offset 0x05C; receives main object and returns the final write address |

Only the query result and final indirect result are explicitly consumed as results. Other return values do not control the displayed flow. No dependency implementation or preservation guarantee is supplied.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. All reached storage must remain valid, and reached dependencies must return normally. Any stability assumptions below apply only to the stated example.

1. **Zero mode with a zero query result.**  
   The initial call receives zero, followed by clears at 0x1D8 and 0x1C5. The three zero-mode preparation calls and F-a1a48bd3fbde occur in order. Its zero result skips F-000000000001 and F-20460d1f31d3; common notification and eligibility remain reachable.

2. **Stable failed nonzero gate.**  
   With nonzero mode and either gate word stably zero, the function returns after the initial call and 0x1D8 clear. It does not perform nonzero preparation, copies, common notification, or final processing.

3. **Absent first source, nonoverlapping second copy.**  
   After a successful gate, suppose the first source is null. At the second source’s first read, offset 0x030 contains P; at its second read, offset 0x034 contains Q. The second pair writes P to 0x1D0 and Q to 0x1D4. If Q is instead described as an initial value, this outcome additionally requires that the first destination write and any other intervening effects do not change that unread source word. The null first source itself performs no writes to 0x1C8 or 0x1CC.

4. **Overlapping second source changes the second value.**  
   Suppose the first source is null and the second source equals the main-object address plus 0x19C. Before copying, main-object offsets 0x1CC and 0x1D0 contain distinct values P and Q. The second source’s offset 0x030 therefore supplies P. Writing P to destination 0x1D0 overwrites the storage used by the second source’s offset 0x034. The next read observes P, and destination 0x1D4 receives P, not the original Q. This assumes no other intervening mutation.

5. **Notification values at unsigned boundaries.**  
   For a nonnull target with a matching backlink, notification receives one for mode zero and zero for modes one, two, and 0xFFFFFFFF, provided the selected branch reaches notification. A mismatched backlink suppresses the call.

6. **Notification redirects the stored related address.**  
   Suppose the captured related address is A, notification changes offset 0x034 to B, and the main-object eligibility bit remains clear. Eligibility consults A’s flags, not B’s. If eligible, the post-processing related-address accesses occur after F-6d16652f2d03 and need not observe A or B.

7. **Low-eight-bit condition.**  
   With a valid nonnull related address after final processing, a main-object value of 0x00000100 at 0x0A0 permits F-000000000002; 0x00000101 suppresses it. Higher bits alone do not determine this condition.

8. **Final result identity.**  
   If F-000000000003 returns the main-object address, the final update sets that object’s bit 0x40. If it returns another valid address, that other object receives the update. A null result has no specified recoverable handling.

## Uncertainties

The following remain **documented_only**:

- Application meanings of modes, gate words, flags, related objects, and copied values.
- Complete layouts, alignment requirements, allocation extents, and lifecycle constraints.
- Full dependency signatures, internal effects, ownership rules, and preservation guarantees.
- Identities and calling conventions of indirect targets.
- Meaning and provenance of the two ambient arguments.
- Synchronization and stability of shared gate storage, its reference, and object fields.
- Outcomes when dependencies invalidate addresses subsequently used.
- Whether the represented word access followed by eight-bit conversion at 0x0A0 corresponds to a narrower machine access.
- Instruction-level access counts, fault behavior, and exceptional exits.

A lifecycle-transition interpretation is inferred intent only. Branch conditions, represented access order, widths, offsets, and explicit calls are established static evidence. The overlapping-copy example is a conditional consequence of that access order, not a runtime observation.

## Evidence

This contract derives from the supplied function representation and opaque dependency map, rather than treating the prior draft or feedback as authoritative.

Offsets were checked using four-byte word indexing. Byte stores are distinguished from word stores, and the unsigned 32-bit mode arithmetic is reflected in the notification outcomes. The copy examples distinguish values observed sequentially from values present before copying; overlap can invalidate preservation of an initially unread value.

No dependency implementations, independent type definitions, instruction-level listing, or runtime observations were supplied. No runtime tests have been performed.
