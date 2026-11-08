<!-- Trusted provenance: F-28ddfa58f25b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-28ddfa58f25b

## Purpose

F-28ddfa58f25b scans a selected range of eight-byte records, optionally filters each record, and conditionally invokes state-update operations on an associated owner object.

The function returns whether at least one record reached the end of the accepted-record update sequence. This is distinct from whether the input contained records, whether a filter admitted a record, or whether an individual dependency reported success.

The apparent intent is paginated processing with an optional end-of-range indicator. That interpretation is inferred from the range arithmetic and indicator writes. The established behavior is the unsigned range selection and ordered dependency calls described below; application-level meanings such as “page,” “queue,” and “commit” are not established.

## Interface

The function takes five arguments:

1. **Owner object:** a pointer to a mutable object accessed through four-byte fields.
2. **Source object:** an address or handle passed to source-related dependencies and directly used to locate record storage.
3. **Range ordinal:** a 32-bit integer used in calculating the starting record index when the range size is nonzero.
4. **Range size:** a 32-bit integer. Zero selects an unbounded scan from index zero; a nonzero value selects a calculated range.
5. **Optional range-status destination:** a pointer to one writable byte, or null.

The result is a 32-bit value:

- **0:** no record completed the accepted-record update sequence.
- **1:** at least one record completed that sequence.

These meanings assume normal return. The supplied evidence does not establish exception, fault, cancellation, or nonlocal-exit behavior.

### Object layout accessed by this function

Offsets are byte offsets from the owner object:

| Offset | Access and role established here |
|---|---|
| 0 | Four-byte filter handle or pointer. Zero bypasses filtering. |
| 4 | Four-byte value supplied to the initial dependency. |
| 28 | Address passed as an embedded-object argument during accepted-record handling. |
| 40 | Address passed as an embedded-object argument in one update branch. |
| 48 | Four-byte field used in a branch comparison; its address is also passed to several dependencies. |
| 52 | Four-byte field supplied to one update dependency. |
| 56 | Four-byte field compared with the field at offset 48. |
| 72 | Four-byte value supplied to the record-resolution dependency. |

The source object contains a four-byte pointer field at byte offset 44. For each scanned record, the function follows that field and then reads another pointer-sized value to obtain the record-storage base.

Each record occupies eight bytes and is copied into a local record as two four-byte words. No interpretation of those words is established.

The record-resolution result used by this function consists of three four-byte words. Its third word has a low-byte acceptance indicator. The remaining bits and words are not interpreted directly by this function.

### Preconditions

For ordinary completion:

- The owner must support the stated field reads and any accesses performed by dependencies.
- The source must support its direct pointer-chain reads and accesses performed by dependencies.
- Every visited index must identify an accessible eight-byte record.
- A nonnull range-status destination must permit a one-byte write whenever the range size is nonzero.
- Dependency output buffers must be populated sufficiently for the reads described below.

No local validation establishes these conditions. In particular, no null-owner check, null-source check, index validation, allocation check, or negative-argument rejection is shown.

## Behavior

### Initial operations

The accumulated result starts at zero.

Before calculating the scan range, the function:

1. Reads the owner field at offset 4 and calls F-4017a9c89a1e with that value.
2. Calls F-e703ef682ffd with the source object.
3. Treats the result of F-e703ef682ffd as a 32-bit unsigned record count.

The count is obtained once. It is not refreshed during iteration.

The first dependency therefore runs even when the eventual scan is empty. F-e703ef682ffd runs after it, so effects of the first call may affect the count or other subsequently observed state.

### Range selection and status byte

When the range size is zero:

- The starting index is zero.
- The exclusive upper bound is the returned count.
- The optional status byte is not written by this function.

When the range size is nonzero:

- The starting index is the stored 32-bit product of range size and range ordinal.
- A candidate exclusive upper bound is the stored 32-bit sum of that start and the range size.
- The candidate bound is compared with the count using an unsigned comparison.

If the candidate bound is strictly less than the count:

- The candidate becomes the exclusive upper bound.
- A nonnull status destination receives byte value 0.

Otherwise:

- The count remains the exclusive upper bound.
- A nonnull status destination receives byte value 1.

The status byte is written before any per-record processing. It describes this bound comparison, not whether any record is accepted or whether all updates finish.

Equality takes the status-1 branch. A starting index already at or above the final upper bound produces an empty scan.

The stored indexes and bounds are 32-bit unsigned values. There is no visible overflow check. Low-32-bit arithmetic can produce a bound below the start, causing an empty scan. It can also produce a small starting index from large argument bit patterns. The supplied representation expresses the initial product using signed integer operands; behavior outside the signed representable range is therefore documented_only rather than a portable signed-overflow guarantee.

### Record acquisition

For each index strictly below the chosen upper bound, using unsigned comparison:

1. Read the pointer field at source offset 44.
2. Read the record-storage base through that pointer.
3. Locate the indexed record at an eight-byte stride.
4. Copy its first four-byte word, followed by its second four-byte word, into local storage.

The storage pointer chain is evaluated again for each visited index. It is not cached across the entire scan. Consequently, dependencies that alter the source’s storage pointers can affect later record reads, while the already selected count and upper bound remain unchanged.

### Optional filtering

After copying the record, the function reads the owner field at offset 0.

- If the field is zero, the record proceeds without a filter call.
- If it is nonzero, F-1e5fe4e09258 receives that value and the address of the local eight-byte record.
- A zero filter result skips the remaining processing for this record.
- Any nonzero filter result permits further processing.

The local record is passed by address. The evidence does not establish that the filtering dependency leaves it unchanged. Later processing uses the local record as it exists after filtering.

### Resolution and acceptance

For a record that passes the optional filter:

1. Read the owner field at offset 72.
2. Call F-520579276b45 with a local output area, that owner-field value, and the local record’s address.
3. Copy the three four-byte output words into a separate local working result.
4. Inspect the low byte of the working result’s third word.

A zero low byte skips all accepted-record updates for this record. Other bits in the third word do not affect this decision.

Any nonzero low byte admits the record to the update sequence. This is a separate condition from the optional filter result.

### Accepted-record updates

For every admitted record, the function performs these operations in order:

1. Call F-d263c4119f37 with the address at owner offset 28 and the local record’s address.
2. Replace the working result’s third word with the full 32-bit value 1.
3. Compare the current four-byte owner fields at offsets 48 and 56.

The comparison occurs after F-d263c4119f37 returns. It therefore uses state available at that point, not necessarily state from before the call.

If the two fields are equal:

- Call F-70defcf64c20 with the address at owner offset 40, the current field at offset 48, the current field at offset 52, and literal value 1.

If the two fields are unequal:

1. Call F-af9bc0871b42 with the address at owner offset 48.
2. Call F-7c7f55b590bd with an eight-byte local output area, the address at owner offset 48, and literal value 1.
3. Call F-4808cb187ddd with the working result’s address and that eight-byte local area.
4. Read the low byte of the working result’s third word.
5. Call F-bcb6380f47f7 with the working result’s address and that byte value, widened to the dependency’s argument representation.

The final byte argument in the unequal branch is not guaranteed to remain 1: F-4808cb187ddd receives access to the working result before the byte is read.

After the selected branch returns normally, the accumulated result becomes 1. It is never reset during the invocation.

The function then advances the index by one and continues while it remains below the fixed upper bound.

### Final operation

After the loop, including an empty loop, the function calls F-00b2734c458c with the source object and returns the accumulated result.

No return value from this final dependency is used.

## Side effects

Directly established writes are:

- A possible one-byte write to the optional range-status destination.
- Construction and modification of local record and result buffers.
- The local accumulated result changing from 0 to 1 after an accepted update sequence.

Owner and source mutations are possible through dependency calls. Their exact contents are not established by this function alone.

There is no explicit allocation, deallocation, or direct ownership transfer in the visible body. This does not establish that dependencies avoid allocating, retaining pointers, releasing resources, or changing global state.

Local record, result, and intermediate buffers are temporary storage. Their addresses are passed to dependencies, but this evidence does not prove whether those dependencies retain them. Safe lifetime requirements for retained references remain documented_only.

There is no rollback path. If a dependency fails to return normally, earlier status writes and dependency side effects may already have occurred. Completion of F-00b2734c458c is not guaranteed on such a path.

No dependency call is assumed to preserve global state, owner state, source state, or passed buffers unless its separate contract establishes that property.

## Dependencies

| Opaque ID | Use within this function |
|---|---|
| F-4017a9c89a1e | Initial call using the owner field at offset 4; result ignored. |
| F-e703ef682ffd | Obtains the single 32-bit unsigned count used to select the range. |
| F-1e5fe4e09258 | Optional per-record filter; zero rejects, nonzero permits further processing. |
| F-520579276b45 | Produces the three-word result whose third word’s low byte controls acceptance. |
| F-d263c4119f37 | First update for an accepted record, using owner offset 28 and the local record. |
| F-70defcf64c20 | Equal-fields update branch. |
| F-af9bc0871b42 | First operation in the unequal-fields branch. |
| F-7c7f55b590bd | Produces an intermediate eight-byte object in the unequal-fields branch. |
| F-4808cb187ddd | Receives the working result and intermediate object before finalization. |
| F-bcb6380f47f7 | Final unequal-branch operation; receives the then-current low byte of the working result’s third word. |
| F-00b2734c458c | Final source-related call after all selected records have been considered. |

The initial and final calls may be paired operations, but locking, lifetime management, and release semantics are not established.

## Acceptance examples

These are conditional static examples, not executed tests. They assume valid accessible objects and normally returning dependencies.

### Unbounded scan

Given a count of 3 and range size 0:

- The visited indexes are 0, 1, and 2.
- A nonnull status destination retains its prior value unless a dependency changes it.
- The final source-related dependency is called even if every record is rejected.

### Interior range

Given count 10, range ordinal 1, and range size 3:

- The start is 3 and candidate upper bound is 6.
- Indexes 3, 4, and 5 are visited.
- A nonnull status destination receives 0 before the first record read.

### Range ending exactly at the count

Given count 6, range ordinal 1, and range size 3:

- Indexes 3, 4, and 5 are visited.
- The candidate upper bound equals the count.
- A nonnull status destination receives 1.

### Start beyond the count

Given count 5, range ordinal 2, and range size 3:

- The start is 6.
- No records are read.
- A nonnull status destination receives 1.
- The result is 0 after the final source-related call returns.

### Filter rejection

If the owner’s filter field is nonzero and F-1e5fe4e09258 returns zero:

- F-520579276b45 is not called for that record.
- No accepted-record update runs for that record.
- A prior accumulated result of 1 remains 1.

### Low-byte acceptance boundary

If the resolution result’s third word is 0x00000100, its low byte is zero and the record is not admitted.

If that word is 0x00000102, its low byte is nonzero and the record is admitted. Before the owner-field comparison, the entire third word is replaced with 1.

### Equal and unequal update branches

For an admitted record, equality of owner fields at offsets 48 and 56 after F-d263c4119f37 selects only F-70defcf64c20.

Inequality selects F-af9bc0871b42, F-7c7f55b590bd, F-4808cb187ddd, and F-bcb6380f47f7 in that order.

If F-4808cb187ddd changes the working result’s third word to 0x0000007F, the final dependency receives byte-derived value 127 rather than 1.

### Wrapped bound without overflowing the initial product

With count 0xFFFFFFFF, range ordinal 1, and range-size bit pattern 0x80000001:

- The stored start is 0x80000001.
- The stored candidate bound is 2.
- Unsigned comparison makes 2 the upper bound and writes status 0 when a destination exists.
- The scan is empty because the start is not below 2.

This demonstrates why status 0 does not imply that any record is visited.

## Uncertainties

The following are documented_only:

- Application-level meanings of records, owner fields, and dependency operations.
- Dependency error behavior, synchronization, reentrancy, ownership, allocation, and global-state effects.
- Whether dependencies fully initialize their output buffers and whether they mutate input records.
- Valid ranges for the range ordinal and range size.
- Target-specific treatment of signed multiplication outside the representable signed range.
- Address-overflow behavior and record-store capacity beyond valid caller-provided inputs.
- Aliasing restrictions among the owner, source, status destination, and storage reachable through them.

Aliasing can matter observably: the status byte is written before record reads, and later owner and storage fields are read after dependency calls. No independent non-aliasing guarantee is supplied.

## Evidence

The established behavior comes exclusively from the supplied static function representation: explicit field offsets, four-byte word accesses, eight-byte record stride, unsigned comparisons, low-byte tests, local-buffer copies, call order, and return assignments.

The pagination interpretation and possible pairing of the initial and final calls are inferred intent, not established dependency semantics.

No runtime tests have been performed. No external implementation, dependency body, or prior behavioral document was used.
