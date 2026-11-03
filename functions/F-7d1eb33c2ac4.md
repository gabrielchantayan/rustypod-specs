<!-- Trusted provenance: F-7d1eb33c2ac4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7d1eb33c2ac4

## Purpose

F-7d1eb33c2ac4 obtains an address from a preparation dependency, writes two source-derived 32-bit words into storage relative to that address, and invokes an indirect operation with a freshly calculated interior or relative address. On normal completion, it returns the address obtained from the preparation dependency.

A null source selects an additional initialization path. That path writes a word from globally designated storage into the caller’s destination and invokes a dependency at destination plus 176 bytes. The dependency’s return value determines the destination subsequently supplied for preparation.

These memory operations and calls are established static behavior. Descriptions such as construction, base-state copying, or registration are inferred intent, not established application semantics.

## Interface

### Inputs

| Input | Representation | Established use |
|---|---|---|
| Destination | Address of storage accessed as 32-bit words | Used directly on the null-source path; supplied unchanged or adjusted to F-bf74dfda5df0. |
| Source | Nullable address of storage containing two accessed 32-bit words | Supplies the result’s initial word and a payload written at a metadata-selected displacement. Null selects globally designated source storage. |
| Auxiliary value | Opaque 32-bit value | Passed unchanged to F-bf74dfda5df0. |

The represented model uses 32-bit integer words and 32-bit integer treatment of addresses. Address calculations are not established as portable to wider-address environments.

No explicit distinctness check separates destination, source, result, metadata, or global storage. Overlap can affect subsequent reads. The absence of an overlap check does not establish that every overlapping arrangement is supported by the surrounding system.

### Output

On normal completion, the function returns exactly the address returned by F-bf74dfda5df0.

That address need not equal the original destination or the destination supplied to F-bf74dfda5df0. The return value of the final indirect operation, if any, is unused.

There is no explicit status result or locally generated error code.

### Accessed layouts

The selected source supplies:

- At byte offset 0: a 32-bit word that is written at result offset 0 and used as a metadata address.
- At byte offset 4: a 32-bit payload written at a displacement-selected address relative to the result.

The metadata address supplied by the first word must permit a 32-bit displacement read 12 bytes before that address.

The object returned by F-df3f6f4fe2dc must permit:

- A 32-bit address-valued read at offset 0.
- Resolution of an indirect callable entry at byte offset 152 from the address obtained by that read.

The source-derived metadata and the dispatch object serve different purposes. The former supplies result-relative displacements; the latter supplies the indirect target and its first argument.

### Preconditions for normal completion

All actual reads, writes, address calculations, and calls must be valid when they occur. In particular:

- The selected source must permit its two word reads after F-bf74dfda5df0 returns.
- The null-source path must permit its global lookup, initial word read, and destination write.
- The null-source dependency must accept destination plus 176 bytes.
- F-bf74dfda5df0 must return an address suitable for the subsequent accesses.
- Each metadata-derived address must be valid for its particular access.
- The final dispatch object, callable entry, and argument addresses must satisfy the indirect operation’s requirements.

No local checks enforce these conditions. Required alignment, accessible extents, valid displacement ranges, and dependency-specific preconditions remain documented_only.

## Behavior

### Source selection and optional initialization

Before testing whether the source is null, the function reads and retains the global default-source pointer.

For a non-null source:

- The caller’s source remains selected.
- The original destination is supplied to F-bf74dfda5df0.
- This function performs no direct destination write before that call.

For a null source, the function performs the following ordered actions:

1. Consult the global default-source pointer for the initial word read and write that word at destination offset 0.
2. Invoke F-af167f4d7e19 with original destination plus 176 bytes.
3. Subtract 176 from that dependency’s returned 32-bit integer value and treat the result as the destination address for preparation.
4. Select the default-source pointer captured before the branch as the source for later reads.

The initial word-read expression consults the global pointer separately from the earlier capture. With no intervening external mutation, both designate the same storage. The supplied body does not establish synchronization or behavior under concurrent changes to the global pointer.

After F-af167f4d7e19, the selected source is the previously captured pointer, not a fresh global lookup. Consequently, changing the global pointer during that dependency does not itself redirect the later source reads. Changing the captured pointer’s pointee can change their values.

The initial destination word is not guaranteed to survive either dependency. No preservation guarantee for destination storage, source storage, metadata, or globals is established.

### Preparation

The function invokes F-bf74dfda5df0 with:

- The original destination when the caller supplied a non-null source; otherwise, the adjusted address derived from F-af167f4d7e19.
- The unchanged auxiliary 32-bit value.

The returned address becomes the result address used for the remaining operations and the final return.

There is no null-result check. The function does not require or verify that the result equals the preparation destination.

F-bf74dfda5df0 runs before either source-word read used to populate the result. Values present at function entry therefore do not, by themselves, determine those writes: the dependency may alter the selected source or its metadata before the reads occur.

### Source-derived writes

After preparation returns, the function reads the selected source’s first word and then writes that word at result offset 0.

The first word is retained for the metadata lookup associated with the next write. That lookup uses the retained value, rather than independently reloading result offset 0.

The next write depends on two reads:

- The 32-bit displacement located 12 bytes before the address represented by the retained first word.
- The selected source’s second 32-bit word, at source offset 4.

**Their relative evaluation order is not established.** They occur in different operands of the same assignment expression. Both follow the first result write, and both precede the second result write.

Once those values are obtained, the function writes the second source word at result plus the displacement.

This partial ordering matters for overlap:

- If the first result write changes source offset 4, the subsequent payload read observes that location after the first write.
- If the first result write changes the displacement’s storage, the displacement read likewise occurs after that write.
- The contract does not impose a displacement-read-before-payload-read order, or the reverse.

The displacement is a 32-bit integer. There is no local rejection of a negative value, no bounds check, and no checked or saturating address arithmetic. Exact behavior for signed overflow, unrepresentable addresses, or invalid resulting addresses is not established by the supplied representation.

### Obtaining the dispatch object

After both result writes, the function invokes F-df3f6f4fe2dc with no explicit arguments.

Its returned address supplies:

- The object from which the final callable entry is resolved.
- The first argument to the indirect operation.

This call is a mutation boundary. It is not established to preserve result contents, source contents, either metadata structure, the global pointer, or the lifetime of any previously accessed storage.

### Indirect operation

The function resolves the callable entry through the object returned by F-df3f6f4fe2dc: it reads that object’s first word as a dispatch address and obtains the entry at dispatch offset 152 bytes.

The indirect operation receives:

1. The object returned by F-df3f6f4fe2dc.
2. A freshly adjusted result address.

The second argument uses a new metadata lookup:

- Read the word currently at result offset 0.
- Read a 32-bit displacement 12 bytes before the address represented by that word.
- Add that displacement to the retained result address.

This calculation does not reuse either the retained source-first-word value or the earlier displacement. The final argument can differ from the location of the second result write.

Such a difference may result from overlap during the second write or from mutations performed by F-df3f6f4fe2dc.

Callable-target resolution and calculation of the second argument must both complete before invocation. Their relative evaluation order is not established by the supplied expression.

### Return

After the indirect operation returns normally, the function returns the retained result address from F-bf74dfda5df0.

Mutating the contents at that address does not change which address this function returns. Whether it still denotes live, usable storage depends on dependency behavior that is not supplied.

## Side effects

The function directly performs:

- A destination-offset-0 word write on the null-source path.
- A result-offset-0 word write on every normally progressing path.
- A displacement-selected word write relative to the result.
- The dependency calls and indirect invocation described above.

It reads the global default-source pointer even for a non-null source. It contains no explicit assignment to that global pointer.

Source storage is not explicitly written through the selected source address. Nevertheless, aliasing can cause the direct writes to alter source storage, metadata, or other reachable state.

The body contains no explicit allocation, deallocation, reference-count adjustment, locking, rollback, or cleanup. Dependencies may perform any of these. Their absence from the local body is not evidence that the complete invocation lacks them.

There is no established transactional guarantee. If an access or call does not complete normally, earlier writes are not explicitly undone. No local recovery path is shown.

Ownership and retention of the destination, selected source, result, global storage, dispatch object, and indirect arguments remain documented_only. Passing or returning an address does not establish ownership transfer, preservation, or release.

## Dependencies

| Dependency | Invocation | Established role |
|---|---|---|
| F-af167f4d7e19 | Null-source path only; original destination plus 176 bytes | Its returned 32-bit integer, minus 176, determines the preparation destination. |
| F-bf74dfda5df0 | Every normally progressing path; preparation destination and auxiliary value | Returns the result address used for subsequent operations and final return. |
| F-df3f6f4fe2dc | After both result writes; no explicit arguments | Returns the dispatch object and first indirect argument. |
| Unidentified indirect target | Entry at byte offset 152 in the resolved dispatch structure | Receives the dispatch object and freshly adjusted result address. Its result is unused. |

Allocation, release, ownership transfer, preservation of memory or globals, failure signaling, and retention behavior are unknown for all dependencies. No opaque identity for the indirect target is supplied; none is invented here.

## Acceptance examples

These are conditional behavioral criteria, not executed tests. Unless stated otherwise, all accesses are valid, address arithmetic is representable, and every call returns normally.

### Non-null source with stable post-preparation values

Immediately after F-bf74dfda5df0 returns, suppose:

- Source offset 0 contains a metadata address.
- The displacement 12 bytes before that address is 24.
- Source offset 4 contains 9.
- The first result write does not overlap the payload or displacement storage.
- The second write does not change result offset 0 or the metadata needed for final adjustment.
- F-df3f6f4fe2dc preserves that result word and metadata until the final argument is calculated.

The function must write the metadata address at result offset 0, write 9 at result plus 24, and pass result plus 24 as the indirect operation’s second argument. It must not call F-af167f4d7e19. Its returned address must equal F-bf74dfda5df0’s result.

These fixture values are established after preparation; no preservation assumption about F-bf74dfda5df0 is implied.

### Preparation changes the source

Suppose a non-null source initially has payload 9, but F-bf74dfda5df0 changes that payload to 11. If the payload remains 11 until read, the second result write must use 11, not the entry-time value 9.

### Null source with a changed global pointer

Suppose the initially captured global pointer designates source A, and F-af167f4d7e19 changes the global pointer to designate source B.

The later source reads must still use source A. Changes to A’s contents by either preparation dependency can affect those reads. The function does not require the later first-word value to match the initial destination write.

### Adjusted preparation destination

Suppose the null-source dependency returns a valid 32-bit value representing an address 176 bytes beyond storage suitable for preparation. F-bf74dfda5df0 must receive that value minus 176, rather than necessarily receiving the original destination.

### Result differs from preparation destination

If F-bf74dfda5df0 returns a valid address distinct from its destination argument, subsequent result writes, the final adjusted argument, and the return value must use that returned address.

### Metadata changes before dispatch

Suppose the displacement used for the second write is 16. F-df3f6f4fe2dc then changes the metadata used by the fresh lookup so that its displacement is 40.

The second write occurs at result plus 16. The indirect operation receives result plus 40 as its second argument.

### Overlapping source and result

If result offset 0 aliases source offset 4, the first result write changes the word later read as the payload. Subject to valid storage and metadata, the second write must use that changed word, not a pre-write snapshot.

This criterion does not depend on which of the payload and displacement reads occurs first.

### Negative displacement

If the displacement is −8 and all relevant arithmetic and accesses are valid, the second write targets eight bytes before the result. A fresh displacement of −8 likewise selects result minus 8 for the indirect argument. No local rule rejects the value solely for being negative.

## Uncertainties

The following remain documented_only:

- Application-level object types and intended lifecycle operation.
- Meaning of the auxiliary value.
- Global default-source initialization, lifetime, and synchronization.
- Dependency effects on storage, globals, ownership, and liveness.
- Valid object extents, alignment, metadata formats beyond accessed fields, and displacement ranges.
- Error signaling, exceptions, nonlocal exits, and invalid-access behavior.
- Exact overflow behavior and portability outside the represented 32-bit model.
- Relative order of the payload and displacement reads for the second write.
- Relative order of final callable-target resolution and adjusted-argument calculation.
- Calling-convention details and the indirect target’s identity.
- Supported overlapping layouts.
- Whether the indirect operation retains either argument.
- Atomicity, concurrent mutation, and memory-order guarantees.

## Evidence

Established static evidence consists solely of the supplied function body and opaque dependency mapping. It supports the branch condition, accessed word widths, byte offsets, explicit calls, retained source pointer, result-address identity, and fresh metadata lookup before indirect invocation.

The body establishes only the partial access ordering described above. In particular, it does not establish an order between the second source-word read and the displacement read used for the same write, nor between final target resolution and second-argument calculation.

Construction and dispatch terminology describes structural patterns, not verified application intent. Dependency implementations and runtime observations were not supplied. No runtime tests have been performed.
