<!-- Trusted provenance: F-71ca24cb2eb3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-71ca24cb2eb3

## Purpose

F-71ca24cb2eb3 associates a context with an input object, constructs category-indexed chains of references to the object’s records, invokes preparation, processes a selected suffix of the record list, and invokes two completion dependencies.

The established contract concerns memory accesses, allocation requests, traversal, dependency calls, and state changes. The semantic meanings of categories, records, and completion data are not established.

Two distinctions are essential:

- Descriptor enumeration covers every record reached during the construction traversal. The later processing traversal skips only a leading run of records whose discriminator byte equals `9`.
- During construction, node allocation returns **before either the descriptor address in the current scratch word or the category index through that address is read**. Descriptor selection is therefore not snapshotted before allocation.

## Interface

### Inputs and result

The function accepts two address-valued inputs in a 32-bit address model:

- **Context:** writable processing state containing, among other fields, a pointer to category chain-head slots.
- **Input object:** an object containing pointers to associated metadata and a linked record list, plus a completion word.

There is no explicit return value or error result.

A **word** below is four bytes. Address-valued fields occupy one word. Offsets are byte offsets from the relevant object’s base.

### Referenced layouts

| Object | Offset | Access and meaning established at this call site |
|---|---:|---|
| Context | `0x08` | Word read for each node-allocation call |
| Context | `0x10` | Word passed to the initial-value dependency |
| Context | `0x208` | Active-object field; receives the input address initially and zero on normal completion |
| Context | `0x21c` | Pointer to writable category chain-head slots |
| Input object | `0x04` | Pointer to the count-bearing structure |
| Input object | `0x08` | Pointer to the initial-value destination |
| Input object | `0x0c` | Record-list head pointer |
| Input object | `0x20` | Completion word, read after processing |
| Count-bearing structure | `0x20` | Four-byte category count |
| Initial-value destination | `0x08` | Four-byte destination for the initial dependency’s result |
| Record | `0x00` | Next-record pointer |
| Record | `0x08` | Single-byte discriminator |
| Descriptor | `0x10` | Four-byte category index |
| Allocated node | `0x00` | Link slot used when a subsequent node is appended |
| Allocated node | `0x04` | Associated record pointer |

The function also uses:

- Temporary allocated storage containing one four-byte append-position pointer per category.
- A local scratch area nominally containing 64 four-byte entries. Each consumed entry is interpreted as a descriptor address.
- An endpoint returned by the enumeration dependency, compared against the scratch cursor to determine when consumption ends.

### Preconditions for ordinary bounded execution

The function does not enforce these conditions:

- All directly accessed memory is valid, suitably aligned for its access width, and writable where required.
- The category count supports the requested allocation size and initialization traversal.
- The chain-head array contains the required writable slots.
- Every consumed scratch entry identifies a readable descriptor.
- Every descriptor index selects an initialized temporary-array entry and corresponding category.
- Allocation results provide the required usable storage.
- Each record traversal eventually reaches a null pointer.
- The enumeration endpoint is reachable from the scratch beginning by four-byte forward increments, within valid scratch storage.
- Dependencies return and do not invalidate storage needed by subsequent accesses.

These are safety conditions, not checked rejection cases.

## Behavior

### 1. Publish the input and initialize an associated word

The first direct write stores the input-object address at context offset `0x208`.

Next, the function reads and retains the destination pointer from input-object offset `0x08`. It reads the context word at `0x10` and passes that word to F-0a871f21bb11. After that dependency returns, its four-byte result is written at destination offset `0x08`.

The destination pointer is captured before the dependency call. A change to the input object’s destination-pointer field during that call does not redirect this write. The captured destination must nevertheless remain writable.

The function itself does not clear the active-object field between subsequent phases. Dependencies may read or change that field; no preservation guarantee is established.

### 2. Allocate append-position storage and clear category heads

After the initial result is stored, the function reads the pointer at input-object offset `0x04`, then the category count at offset `0x20` of that referenced structure.

It calls F-d33f6d597e5d with the count shifted left by two bits, representing four bytes per category in the ordinary non-overflowing case.

For each category, starting at zero and proceeding in increasing order, it performs these writes in order:

1. Store the address of that category’s context chain-head slot in the corresponding temporary-array word.
2. Store zero in that chain-head slot.

The context pointer at `0x21c` is read separately to determine the slot address for the temporary entry and to determine the slot subsequently cleared. It is not established as a single cached pointer for the whole initialization phase.

The temporary entries consequently begin as pointers to head slots. Later they become pointers to node link slots.

Existing heads are overwritten without traversing or directly releasing any prior chains. Their lifecycle is not resolved here.

A zero category count still causes a temporary-allocation request with size zero, but no category-head initialization.

### 3. Enumerate records and append reference nodes

The construction traversal obtains its initial record pointer from input-object offset `0x0c`.

For every non-null record reached, F-93a6f85390ff receives:

- The current record address.
- The beginning of the local scratch area.
- A supplied upper-bound address for that area.

Its returned address becomes the endpoint for consuming scratch entries. Consumption begins at the scratch beginning and advances by one four-byte entry until the cursor equals that endpoint.

For each consumed scratch position, the established observable order is:

1. Read the context word at `0x08`.
2. Call F-08f20a63bf27 with that word and size `8`.
3. After allocation returns, read the descriptor address from the current scratch word.
4. Read the category index at offset `0x10` of that descriptor.
5. Read that category’s append-position pointer from temporary storage and write the new node address through it.
6. Write the current record address to new-node offset `0x04`.
7. Replace that category’s temporary append-position pointer with the new node address.

Thus **both descriptor-address selection and the category-index read occur after node allocation returns**. The allocation is associated with a scratch position before the descriptor stored there is read, not with a descriptor address captured beforehand.

This distinction matters if a dependency can reach the scratch area. F-93a6f85390ff receives its address and might retain it. If an intervening node-allocation call changes the current scratch word through such retained access, the subsequent descriptor lookup uses the changed word. The body does not prevent this, but the supplied evidence does not establish that any dependency actually performs such a mutation.

Likewise, if allocation changes the selected descriptor’s category field before the later read, that changed category is used. The scratch endpoint has already been obtained; changing scratch entries does not itself replace the endpoint held by this invocation.

For stable, valid inputs, this construction preserves record traversal order within each category. Repeated descriptors for a category produce repeated nodes; there is no deduplication.

The node address is published through the category head or preceding node before its record-reference word is written. Updating the temporary append position afterward makes the new node’s first word the destination of a later append.

**Final link initialization is not established.** The function does not explicitly write a new node’s first word when allocating it. A subsequent append overwrites that word with the next node address, but the last node in a nonempty category retains whatever first-word contents allocation or other external behavior supplied. Null termination therefore requires an additional guarantee not established here.

After finishing a record’s scratch entries, the function reads that record’s next pointer. Mutations made by dependencies before this read can change the construction traversal.

### 4. Pass temporary storage to a release dependency and prepare

After the construction traversal reaches null, the function passes the temporary allocation address to F-5e3309c9d083.

It then calls F-cf8e9318ec25 with the context.

The temporary allocation is not directly accessed afterward. The invocation of F-5e3309c9d083 establishes a release-related call site, not independently verified deallocation semantics.

No allocated eight-byte node is directly released by this function. Whether any dependency consumes, retains, changes, or releases those nodes is unknown.

### 5. Skip the leading discriminator run and process the suffix

After preparation returns, the function reads the list head again from input-object offset `0x0c`. The processing traversal therefore need not see the same list that construction saw if a dependency changed the head or links.

Initially, it examines the byte at record offset `0x08`:

- A value of `9` causes it to follow that record’s next pointer and continue examining records.
- Any other byte value begins the processing suffix.
- Reaching null first produces no record-processing calls.

Once the suffix begins, F-dd58a80227f9 receives the context and each current record until the traversal reaches null. The discriminator is not checked again in this suffix. A later record with discriminator `9` is processed.

After each processing call returns, the function reads the current record’s next pointer. The processing dependency can therefore affect subsequent traversal if it changes that pointer. It must leave the current record readable for this access for ordinary valid execution.

### 6. Complete and clear the active object

After processing, even if no record was processed, the function:

1. Reads the input-object word at `0x20`.
2. Calls F-be353302b92c with the context and that word.
3. Calls F-6e21455df470 with the context.
4. Writes zero to context offset `0x208`.
5. Returns without a value.

The completion word is read at this late stage, not captured at entry. Earlier dependency mutations can affect the argument received by F-be353302b92c.

The final zero write follows both completion calls and overwrites whatever value is then in the active-object field. It is established only on the visible normal-return path, not as cleanup after a fault or a dependency that does not return.

## Side effects

Direct writes comprise:

- Initial publication and final clearing of the active-object field.
- Replacement of one word in the captured initial-value destination.
- Clearing category heads.
- Initializing and updating temporary append-position entries.
- Publishing node addresses through category heads or preceding links.
- Writing record addresses into nodes.

Resource-related calls comprise one temporary allocation, one later release-dependency invocation for that address, and one eight-byte node allocation per consumed scratch entry. An empty enumeration produces no node allocations.

No direct allocation-failure check, index check, endpoint validation, rollback, old-head restoration, or error result is shown.

Publishing a node before writing its record-reference word permits an observer to encounter a partially initialized node if concurrency or reentrant observation is otherwise possible. No synchronization or atomic publication protocol is established.

Dependency effects are additional to these direct effects. No dependency is assumed to preserve global state, object fields, scratch contents, or ownership.

## Dependencies

| Opaque identity | Contract visible at this call site |
|---|---|
| F-0a871f21bb11 | Receives the context word at `0x10`; returns a word stored through the previously captured destination |
| F-d33f6d597e5d | Receives the shifted category-count size; returns the temporary-storage address |
| F-93a6f85390ff | Receives a record and scratch boundaries; returns the scratch-consumption endpoint |
| F-08f20a63bf27 | Receives the current context word at `0x08` and size `8`; returns a node address before the current descriptor address is loaded |
| F-5e3309c9d083 | Receives the temporary allocation after construction |
| F-cf8e9318ec25 | Receives the context before the list head is reread for processing |
| F-dd58a80227f9 | Receives the context and each record in the selected processing suffix |
| F-be353302b92c | Receives the context and the late-read completion word |
| F-6e21455df470 | Receives the context immediately before the final active-object clear |

These observations do not establish internal algorithms, node initialization, memory-release guarantees, ownership transfer, failure conventions, or preservation of accessible state.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Unless a mutation is explicitly stated, they assume stable inputs, valid storage, valid enumeration results, successful allocations, and returning dependencies.

1. **Empty list.** With a null list head throughout the relevant phases, category heads are initialized, no enumeration or node-allocation calls occur, temporary storage is passed to F-5e3309c9d083, preparation runs, no record-processing calls occur, both completion dependencies run, and the active-object field ends as zero.

2. **Zero categories with empty enumerations.** The temporary allocation receives size zero. No head slots are cleared. Records may still be enumerated, but empty enumerations produce no nodes. The temporary allocation result is still passed to F-5e3309c9d083.

3. **Repeated category.** If one record supplies two scratch entries whose descriptors both select category `2`, two nodes are allocated. The category head receives the first node, the first node links to the second, and both record-reference words contain that record’s address. The second node’s outgoing link is not explicitly initialized here.

4. **Order across records.** If earlier and later records each contribute one descriptor for the same category, the earlier record’s node precedes the later record’s node.

5. **Descriptor substitution during allocation.** Suppose the current scratch word initially identifies descriptor A. If F-08f20a63bf27 changes that word to identify valid descriptor B before returning, the function reads B afterward and uses B’s category. It does not use a pre-allocation snapshot of A. If A remains selected but its category changes during allocation, the subsequent category read observes the changed field.

6. **Leading-only skip.** For discriminator bytes `[9, 9, 4, 9, 2]`, all five records participate in construction. Processing receives the third, fourth, and fifth records, in order.

7. **Entire processing list skipped.** For `[9, 9]`, no F-dd58a80227f9 calls occur. Preparation and both completion calls still occur.

8. **Late completion value.** If an earlier returning dependency changes input-object offset `0x20`, F-be353302b92c receives the value present at the eventual read.

9. **Invalid category or endpoint.** An out-of-range category or malformed scratch endpoint has no established rejection result. The function performs unchecked accesses or traversal; an error report or safe early return must not be expected.

## Uncertainties

### Established static evidence

The supplied body establishes the access widths and offsets, dependency-call order, allocation-before-descriptor-load ordering, node-publication order, separate list traversals, leading-only skip, and final normal-path clear.

### Inferred intent

The temporary array appears intended to maintain per-category append positions. The eight-byte objects appear intended as linked record-reference nodes. These descriptions explain the structure without establishing domain semantics or ownership.

### Documented-only unknowns

- Whether allocation initializes node link words.
- Whether any dependency retains the scratch address or mutates scratch entries.
- Ownership and release obligations for records, descriptors, nodes, and associated structures.
- Allocation-failure and zero-size-allocation behavior.
- Enumeration capacity guarantees.
- Dependency non-returning behavior.
- Support for aliasing, concurrency, or reentrancy.
- Semantic meanings of category indices and discriminator values.

The count and indices occupy 32 bits. Allocation sizing shifts the count by two; slot addressing scales indices by four. No overflow checks are shown. A nonnegative signed 32-bit count through `0x1fffffff` has a four-byte product representable as a signed 32-bit value, but that arithmetic condition alone does not establish allocation success or valid address ranges.

No portable recovery or wraparound contract is assigned to negative counts, overflowing shifts, or invalid address arithmetic. Initialization stops on equality with the count, so an invalid count is not safely interpreted as an empty range.

Scratch consumption likewise stops on address equality, not a validated bounds comparison. An endpoint behind the beginning, outside valid storage, or unreachable by four-byte increments has no established safe outcome.

## Evidence

This contract is based solely on static inspection of the supplied function body and opaque dependency mapping. The prior draft and feedback are not treated as independent proof.

No dependency implementations, independent layout definitions, allocator guarantees, or runtime observations were supplied. No runtime tests have been performed. The acceptance examples express conditional consequences of the visible behavior and do not resolve documented-only unknowns.
