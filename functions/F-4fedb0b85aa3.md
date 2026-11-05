<!-- Trusted provenance: F-4fedb0b85aa3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4fedb0b85aa3

## Purpose

F-4fedb0b85aa3 coordinates delimiter-based extraction from an input object, passes the extracted representation to a classification dependency, and conditionally submits the resulting 32-bit value to a destination-associated dependency.

The visible control flow distinguishes two rejection conditions:

- Either search returns exactly −1.
- Both searches succeed, but classification returns exactly 10.

The second condition invokes a reporting dependency with a diagnostic concerning bad metadata and an unknown encoding. This supports an inferred metadata-encoding purpose. It does not establish the delimiter contents, extraction units, classification domain, or actual destination-write semantics.

This contract describes the supplied static body. Dependency names and apparent roles do not establish their implementations, ownership behavior, preservation of shared state, or normal return.

## Interface

### Inputs

The function has four visible arguments:

| Argument | Visible interpretation |
|---|---|
| Context address | Address from which a 32-bit destination-object reference is read after both searches succeed. Also forwarded to initialization and reporting. |
| Input-object address | 32-bit address-like value. The function uses an embedded object at byte offset 12 and forwards the original address to initialization and reporting. |
| First incoming word | Uninterpreted 32-bit value used to initialize the first word of temporary local storage. |
| Second incoming word | Uninterpreted 32-bit value used to initialize the adjacent second word of that storage. |

The initialized local storage has this visible layout:

| Byte offset | Initial content |
|---|---|
| 0 | First incoming word |
| 4 | Second incoming word |

The address of the first word is passed to extraction, classification, and cleanup dependencies. The visible layout does not establish that these words form a pointer-and-length pair, a complete string object, or any other particular representation.

### Output

The function has no explicit return value. It exposes outcomes through dependency calls and their effects.

There is no returned status distinguishing delimiter failure, rejection of classification result 10, or destination submission. Normal return does not independently prove that a destination field was written.

### Widths and addressing

The supplied representation uses 32-bit integer values and four-byte words. Relevant byte offsets are:

| Address relationship | Offset |
|---|---:|
| Embedded object within the input object | 12 |
| First delimiter reference within a descriptor | 4 |
| Second delimiter reference within a descriptor | 8 |
| Submission target relative to the captured destination reference | 84 |

Delimiter references and the destination-object reference are loaded as 32-bit words. Search results, the length-like result, the classification result, and the start and extent calculations are represented as 32-bit integers.

The visible comparisons recognize exactly −1 and exactly 10. There is no general negative-value check or classification-range check.

The supplied body does not establish arithmetic-overflow behavior or guarantee valid address arithmetic. This contract does not impose modular wraparound, saturation, or trapping semantics.

### Preconditions

Normal completion requires each reached dependency to return normally and every reached memory access and dependency argument to satisfy the actual runtime’s requirements.

The caller performs no visible null, alignment, object-size, or range validation. Required storage is path-dependent: for example, the context is directly dereferenced only after both searches succeed, although earlier dependencies already receive its address.

Exact dependency preconditions, object lifetimes, and supported aliasing remain documented_only.

## Behavior

### 1. Initialize temporary storage and invoke initialization

The function first copies the two incoming words into the local storage, first at offset 0 and then at offset 4.

It reads a 32-bit value from a fixed shared location and invokes F-fa3abcde8103 with:

- The context address.
- The original input-object address.
- The shared value.

No returned status is inspected. Subsequent behavior is established only if this dependency returns normally.

The descriptor-address reads occur after this call. Therefore, changes made by initialization to the shared descriptor location can affect both subsequent descriptor-address reads.

### 2. Read the shared descriptor address twice and perform the first search

The function performs two distinct reads of the same shared descriptor-address location before the first search:

1. It reads and retains a descriptor address for later field accesses.
2. It reads the shared descriptor address again and loads the first search’s delimiter reference from byte offset 4 of the address obtained by this second read.

The first search’s delimiter is therefore **not necessarily loaded from the retained descriptor address**.

The function invokes F-a21141daa905 with:

- The input-object address plus 12.
- The delimiter reference loaded through the second descriptor-address read.
- Starting value zero.

If the shared descriptor location yields different addresses on the two reads, the first search uses a delimiter from the second descriptor, while later delimiter-field accesses use the first, retained descriptor.

No dependency call is visible between these two shared reads. Whether the shared location can change during that interval, and whether such changes are supported or synchronized, remains unknown.

The first search result is compared with exactly −1:

- If equal, the function takes the delimiter-failure reporting path.
- Otherwise, processing continues without a separate sign or range check.

On first-search failure, the retained descriptor’s fields are not subsequently accessed by this function.

### 3. Read the retained descriptor’s first field and compute the start

After a non-−1 first search result, the function loads a delimiter reference from byte offset 4 of the **retained** descriptor address. It passes that reference to F-1cb585febd84.

This is a fresh field read. It need not yield the delimiter used by the first search:

- The two shared descriptor-address reads may have yielded different descriptors.
- The first search may have modified the retained descriptor’s contents.

The function adds the result of F-1cb585febd84 to the first search result. The resulting 32-bit integer becomes both the extraction start and the starting value for the second search.

The apparent intent is to advance past the first delimiter. That interpretation is inferred, not established: the supplied body does not prove that F-1cb585febd84 returns a byte length, character count, or any particular unit.

There is no visible check for overflow, a negative start, or a start beyond the embedded object.

### 4. Read the retained descriptor’s second field and perform the second search

After F-1cb585febd84 returns, the function loads the second delimiter reference from byte offset 8 of the retained descriptor address.

It invokes F-a21141daa905 with:

- The input-object address plus 12.
- This newly loaded delimiter reference.
- The computed extraction start.

The shared descriptor-address location is not read again for this search. Replacing that shared address during either preceding dependency does not redirect the retained descriptor accesses. Mutating the retained descriptor itself can affect its later field loads.

If the second search returns exactly −1, the function takes the delimiter-failure reporting path.

Any other result continues. The caller does not verify that the result is at least the extraction start or within the input object’s extent.

### 5. Capture the destination reference and extract

After the second search succeeds, the function reads the 32-bit destination-object reference through the context address.

This read occurs after both searches and the length-like call, but before extraction and classification. Changes those earlier dependencies make to the context can affect the captured reference.

The function then invokes F-ed33cfb647fe with:

- The local-storage address.
- The input-object address plus 12.
- The computed extraction start.
- The second search result minus that start.

The last argument is a 32-bit subtraction result. No visible check rejects a zero or negative extent, an excessive extent, or arithmetic overflow.

The arguments are consistent with extracting a half-open interval between delimiters. Actual range units, bounds enforcement, copying, aliasing, and representation construction remain dependency-defined.

Extraction receives storage already containing both incoming words. The caller does not establish whether extraction reads, overwrites, consumes, or otherwise interprets those initial values.

### 6. Classify and select reporting or submission

After extraction returns, the function invokes F-db6fdb08aa63 with the local-storage address.

Its 32-bit result selects one of two branches:

| Classification result | Next dependency call |
|---|---|
| Exactly 10 | F-bd120056260a with the context address, original input-object address, a diagnostic concerning bad metadata and unknown encoding, and a final argument of zero. |
| Any other value | F-1bfc23b6b704 with the captured destination reference plus 84, the classification result, and a final argument of one. |

No additional validation restricts the non-10 branch. Zero, negative results, and positive values other than 10 all select destination submission.

The destination reference is not reread after extraction or classification. If either dependency replaces the reference stored through the context, submission still uses the previously captured reference. Those dependencies may nevertheless modify the object to which that reference points.

### 7. Invoke cleanup after either classification branch returns

After either reporting or destination submission returns normally, the function invokes F-289ec4311c64 with the local-storage address. It then returns if cleanup returns.

Classification result 10 does not itself bypass cleanup.

The supplied body shows no exception handler or other mechanism ensuring cleanup after abnormal termination or nonlocal exit. If an earlier dependency does not return normally, subsequent calls are not established.

### Delimiter-failure reporting

If either search returns exactly −1, the function invokes F-bd120056260a with only the context address and original input-object address explicitly supplied at that call site.

If reporting returns, the function returns. This path does not invoke extraction, classification, destination submission, or local-storage cleanup.

The shorter reporting call must not be expanded into the diagnostic-bearing form by assuming default arguments. Its actual calling convention and argument interpretation remain unresolved.

## Side effects

### Directly visible operations

The function directly performs the following local writes and nonlocal reads, in order along the continuing path:

1. Initializes both local words.
2. Reads the initialization-associated shared value.
3. Calls initialization.
4. Reads and retains the shared descriptor address.
5. Reads the same shared descriptor-address location again.
6. Loads the first search’s delimiter at offset 4 through the second address.
7. Calls the first search.
8. Loads offset 4 through the retained address.
9. Calls the length-like dependency.
10. Loads offset 8 through the retained address.
11. Calls the second search.
12. Reads the destination reference through the context.

Failure branches stop this sequence at their respective search results.

The body contains no direct store into the context, input object, descriptor, or destination object. Any such mutation is dependency-mediated.

### Dependency-mediated state changes

None of the dependencies is established to be pure or to preserve shared locations, descriptor contents, input contents, context contents, or destination contents.

The ordering constrains how mutations affect later behavior:

- Initialization precedes both shared descriptor-address reads.
- The first search precedes the retained descriptor’s offset-4 reload.
- The length-like call precedes the retained descriptor’s offset-8 load.
- Both searches precede destination-reference capture.
- Extraction and classification occur after that capture and cannot change which reference has already been selected for submission.

These constraints do not establish safe concurrent access, synchronization, or protection against invalidated storage.

### Ownership and lifetime

The local-storage address is passed to extraction, classification, and cleanup on the successful-search path. Its storage duration is local to this invocation; validity after return is not established.

The cleanup call suggests resource management but does not prove that it frees memory. Likewise:

- The incoming words are not established as owned resources.
- Extraction is not established to allocate or copy.
- Classification is not established to leave the representation unchanged.
- Reporting and submission are not established to consume or retain their arguments.
- Delimiter failure performs no cleanup call for the initialized local words.

Dependency-specific ownership requirements remain documented_only.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-fa3abcde8103 | Receives context, original input object, and a shared 32-bit value before descriptor reads. No result is tested. |
| F-a21141daa905 | Called up to twice with embedded-object address, delimiter reference, and start. Exactly −1 selects delimiter failure. |
| F-1cb585febd84 | Receives the retained descriptor’s freshly loaded offset-4 reference. Its result contributes to the extraction start. |
| F-ed33cfb647fe | Receives local storage, embedded-object address, start, and computed extent. |
| F-db6fdb08aa63 | Receives local storage and returns the value compared with 10 or forwarded to submission. |
| F-1bfc23b6b704 | Receives captured destination reference plus 84, classification result, and one. Actual write semantics are unknown. |
| F-bd120056260a | Receives either two explicit arguments on delimiter failure or four on result-10 rejection. Reporting and return behavior are unknown. |
| F-289ec4311c64 | Receives local storage after either classification branch returns normally. Resource-release semantics are unknown. |

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Each assumes valid reached accesses, representable arithmetic, and normal return from preceding dependencies.

1. **First-search failure.** If the first search returns −1, delimiter-failure reporting is next. No retained-descriptor field reload, length-like call, second search, extraction, classification, submission, or cleanup follows.

2. **Second-search failure.** If the first search returns 7 and the length-like dependency returns 3, the second search receives start 10. If it returns −1, reporting follows without extraction or cleanup.

3. **Submission path.** With first result 7, length-like result 3, second result 15, and classification result 4, extraction receives start 10 and extent 5. Submission receives the captured destination reference plus 84, value 4, and one. Cleanup follows normal submission return.

4. **Result-10 rejection.** Successful searches followed by classification result 10 select diagnostic-bearing reporting, not submission. Cleanup follows if reporting returns.

5. **Empty or reversed interval.** If the computed start is 10 and the second result is 10, extraction receives extent zero. If the second result is 8, extraction receives −2. Neither condition is rejected by this caller; dependency behavior remains unresolved.

6. **Non-sentinel negative result.** A search result of −2 passes that search’s sentinel check. Likewise, classification result −2 selects submission rather than result-10 reporting.

7. **Different descriptor reads.** Suppose the first shared descriptor-address read yields descriptor A and the second yields descriptor B, with both valid. The first search receives B’s offset-4 field. The length-like dependency receives A’s offset-4 field as read after the first search, and the second search receives A’s offset-8 field as read after the length-like call. This criterion does not assert that changing the shared location between reads is supported.

8. **Descriptor replacement versus mutation.** If the first search replaces the shared descriptor address, later delimiter reads still use the retained address. If it instead modifies the retained descriptor’s offset-4 field, the length-like dependency receives the modified field value.

9. **Destination replacement after capture.** If classification changes the destination reference stored through the context and returns a non-10 result, submission still uses the reference captured before extraction.

## Uncertainties

The following remain documented_only:

- Complete object layouts and meanings of the context, input object, descriptors, and destination-associated field.
- Delimiter contents and the initialization-associated shared value.
- Whether positions and extents use bytes, characters, or another unit.
- Search guarantees beyond the caller’s exact −1 comparisons.
- Integer-overflow behavior and treatment of invalid addresses or ranges.
- Whether and how the shared descriptor location can change between its two reads.
- Synchronization, reentrancy, and concurrent-use requirements.
- Dependency side effects, ownership, retention, allocation, and release behavior.
- The classification domain and why exactly 10 is rejected.
- Whether reporting returns normally and how its shorter call form is interpreted.
- Effects of abnormal termination on resources and shared state.

The inferred intent is delimiter-bounded metadata extraction followed by encoding interpretation. That intent does not strengthen the established bounds, ownership, or error guarantees.

## Evidence

The supplied static body establishes the two local-word initializations, dependency ordering, two separate shared descriptor-address reads before the first search, use of the second address for that search’s delimiter, and use of the retained first address for later delimiter-field reads.

It also establishes the exact −1 and 10 comparisons, byte offsets, start and extent arithmetic, destination-reference capture before extraction, differing reporting argument lists, and cleanup placement after either normally returning classification branch.

The diagnostic supports the metadata-and-encoding interpretation. Dependency implementations and runtime observations were not supplied.

No runtime tests have been performed. All acceptance examples are conditional predictions from static evidence.
