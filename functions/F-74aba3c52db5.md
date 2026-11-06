<!-- Trusted provenance: F-74aba3c52db5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-74aba3c52db5

## Purpose

F-74aba3c52db5 coordinates an operation over two opaque inputs and a signed bound. It requests working storage, initializes local state, repeatedly invokes a processing dependency, obtains two final-stage storage values, publishes one through a caller-provided output location, and invokes a finalization dependency.

The established contract concerns control flow, argument relationships, state exposure, output publication, cleanup calls, and returned statuses. The supplied evidence does not establish the underlying data format or prove that the storage and cleanup dependencies implement conventional allocation and release.

State supplied by initialization or processing is not necessarily stable afterward. Several local regions are exposed by address. Dependencies may retain those addresses and make them available to other dependencies. Subsequent calls can therefore change valid, still-live state even when that state is not an explicit argument to the intervening call. In particular, the final-size slot is read separately for each final-stage storage request and again for finalization.

Publication precedes finalization. A finalization error does not cause this function to restore the previous output value or explicitly clean up the published storage value.

## Interface

### Inputs and result

| Input | Established representation | Use |
|---|---|---|
| Context | Opaque 32-bit value | Passed to initialization. |
| Bound | Signed 32-bit integer | Passed to initialization, processing, and finalization; compared with the current position before processing. |
| Source | Opaque 32-bit value | Passed to initialization and processing. |
| Output location | Address of a writable four-byte slot | Receives the second final-stage storage result if that stage is reached. |

The context and source are not directly dereferenced by this function. Their validity requirements depend on the callees that consume them.

The output location is not checked for nullness, writability, or alignment before the explicit four-byte write. It need not be accessed by this function on earlier-return paths. Dependency-mediated access through aliases or other shared state is not excluded.

The result is a 32-bit integer status:

- **1** when the initial storage request returns zero.
- **7** when a pre-processing signed comparison finds the current position greater than the bound.
- The nonzero initialization status when initialization fails.
- The nonzero processing status when processing fails.
- The finalization status when finalization returns.

These categories can overlap numerically. A result of 1 or 7 does not, by itself, uniquely identify the originating branch.

Zero means “continue” for initialization and processing. Finalization’s result is propagated without a local success test.

### Local state exposed to dependencies

The function supplies addresses of distinct local regions:

| Region | Width and interpretation | Explicit recipients |
|---|---|---|
| Shared state | Four opaque bytes | Initialization, processing, finalization |
| Current position | Signed 32-bit slot | Initialization, processing, finalization |
| Processing/finalization value | Opaque 32-bit slot | Address supplied to initialization; contents later passed to processing and finalization |
| Finalization-only value | Opaque 32-bit slot | Address supplied to initialization; contents later passed to finalization |
| Final size | Signed 32-bit slot | Address supplied to processing; contents later used for storage requests and finalization |
| Processing-only state | Four opaque bytes | Processing |
| Completion | Signed 32-bit slot | Processing |

No local initialization of these regions is visible before their first dependency call. Correct execution requires dependencies or a separate external contract to establish any contents subsequently consumed. The function performs no completeness check on successful initialization or processing.

No internal field structure, byte order, encoding, or numeric meaning is established for the opaque four-byte regions. The evidence does not independently establish pointer width, storage alignment, or dependency interface types beyond the representations visible at this caller.

## Behavior

### Working storage and initialization

The first call is to F-d33f6d597e5d with argument **512**.

A zero result causes immediate return of 1. There is no initialization, processing, final-stage storage request, explicit output write, or cleanup call on that path.

A nonzero result is retained as the working-storage value. The term “storage” describes its role; the evidence does not prove that it is a pointer to 512 usable bytes.

Initialization uses F-289a082c628b. Its ordered argument roles are:

1. Context.
2. Source.
3. Bound.
4. Address of shared state.
5. Address of current position.
6. Address of the processing/finalization value.
7. Address of the finalization-only value.

A nonzero initialization result causes a cleanup call to F-5e3309c9d083 with working storage, followed by return of that initialization result. A zero result permits processing.

Initialization exposes several local addresses. Values established through those addresses are not immutable initialization snapshots. Later dependencies may legitimately modify them through retained or shared references.

### Processing and termination

Before every processing call, the function reads the current-position slot and performs a signed comparison against the bound.

If the current position exceeds the bound, the function calls F-5e3309c9d083 with working storage and returns 7. No processing call occurs for that iteration, and final-stage storage requests, publication, and finalization are skipped.

Equality permits processing. Neither operand is separately required to be nonnegative.

Processing uses F-79487c4f97b1 with these ordered argument roles:

1. Source.
2. Bound.
3. The then-current contents of the processing/finalization value slot.
4. Address of shared state.
5. Address of current position.
6. Address of final size.
7. Address of processing-only state.
8. Working-storage value.
9. Address of completion.

The processing/finalization value is read anew for each processing call. A preceding processing call can change that slot through an address retained from initialization, so later calls need not receive the original initialization value.

If processing returns nonzero, its status takes precedence over completion. The function cleans up working storage and returns that status without inspecting completion to decide whether to finalize.

After a zero processing result, the function reads completion:

- Exactly **1** permits entry to the final stage.
- Every other value causes another pre-processing bound check.

Completion is not treated as a general truth value. There is no local iteration limit or progress requirement. Execution can continue indefinitely if processing repeatedly returns zero, completion remains other than 1 at each inspection, and the position remains within the bound at each check.

The bound check occurs before processing, not after it. A successful processing call that supplies completion 1 can also leave position above the bound; final-stage work still follows. Once the completion read selects the final stage, later mutations of completion do not cause another loop decision.

### Final-stage reads, requests, and publication

The final stage performs three distinct uses of the final-size slot:

| Use | When the slot is read | Consumer |
|---|---|---|
| First request size | Before the first final-stage storage call | F-d33f6d597e5d |
| Second request size | After that first call returns | F-d33f6d597e5d, with four added to the newly read value |
| Finalization size | After the second call returns and the output write occurs | F-00066d1b3db8 |

There is no single size snapshot shared by these three uses.

Processing has received the final-size address. Either storage-request dependency can therefore modify that slot through dependency-shared state if it has access to the retained address. Such a mutation need not invalidate the slot or disrupt normal control flow.

Consequently:

- The first request uses the value present at its own argument read.
- The second request uses the value present at its later argument read, plus four.
- Finalization receives the value present when its arguments are prepared.

A change during the first storage call does not retroactively change that call’s already-supplied argument. It can change the second request and finalization. A change during the second call can change finalization without changing either request’s already-supplied argument.

The first final-stage result is retained as temporary storage. The second is retained as published storage. Both requests occur even if the first result is zero. Neither result is checked before further use.

After the second request returns, the function writes its result to the caller’s four-byte output slot. This includes writing zero if zero was returned. The write precedes finalization and is not conditionally delayed until a successful finalization result.

Final size is a signed 32-bit slot. There is no visible rejection of negative values or range check before adding four. For values near the signed upper limit, the supplied representation does not establish whether the arithmetic wraps, traps, or has another consequence. No particular overflow behavior is an acceptance requirement.

### Finalization and cleanup

F-00066d1b3db8 receives these ordered argument roles:

1. Bound.
2. The then-current finalization-only value.
3. The then-current final size.
4. The then-current processing/finalization value.
5. Address of shared state.
6. Address of current position.
7. Retained temporary-storage result.
8. Retained published-storage result.

The three state values passed by value are read for this call; they are not guaranteed to equal values supplied by initialization or the completing processing call. The intervening storage calls may validly mutate escaped slots.

Shared state and current position are passed by address. Finalization receives access to their live regions, not snapshots of their earlier contents. There is no further bound comparison before finalization.

The supplied representation establishes call sequencing, but does not establish an ordering among individual argument evaluations within a single call.

After finalization returns normally, the function retains its status and calls F-5e3309c9d083 first with temporary storage and then with working storage. It returns the retained finalization status.

Both cleanup calls occur even when finalization returns nonzero. Temporary-storage cleanup also occurs when the temporary-storage result is zero. There is no explicit cleanup call for published storage.

## Side effects

### Direct writes and state exposure

The only explicit write to caller-supplied memory is publication of the second final-stage storage result. Earlier ordinary-return paths contain no direct output-slot write.

This is not a guarantee that dependencies leave the output slot, globals, or other caller-reachable state untouched. Their behavior through aliases, retained addresses, shared state, or external resources is unknown.

Dependencies receive addresses into this function’s local state. The function continues to use some of those regions after intervening calls. Valid mutations through retained addresses are therefore relevant to later comparisons, argument values, and completion decisions.

Those addresses refer to local storage whose lifetime ends when this invocation returns. The evidence does not establish whether dependencies retain them, whether retained references are cleared, or whether a separate contract forbids use after return.

### Resource handling

| Path | Explicit cleanup calls |
|---|---|
| Initial request returns zero | None |
| Initialization fails | Working storage |
| Bound check fails | Working storage |
| Processing fails | Working storage |
| Finalization returns normally | Temporary storage, then working storage |
| Published-storage handling | No explicit cleanup by this function |

These are call obligations, not proof of ownership transfer or release. F-5e3309c9d083 may have effects other than conventional deallocation. Finalization may also affect resource ownership; its implementation is unavailable.

The function provides no local recovery for an invalid output address, abnormal dependency termination, or memory corruption. The ordinary-return behavior assumes required accesses remain valid and dependencies return normally. That assumption does **not** require exposed state to remain unchanged.

## Dependencies

| Opaque ID | Established role and interaction |
|---|---|
| F-d33f6d597e5d | Receives 512 initially, then two independently formed final-stage size arguments. Only its initial result is checked for zero. Its calls are not established to preserve escaped state or globals. |
| F-289a082c628b | Receives opaque inputs, bound, and local-state addresses. Zero permits processing; nonzero is returned after working-storage cleanup. |
| F-79487c4f97b1 | Performs repeated processing with live state and completion output. Nonzero terminates processing; zero permits completion inspection. |
| F-00066d1b3db8 | Runs after publication, receives current state and both retained final-stage storage values, and supplies the eventual returned status. |
| F-5e3309c9d083 | Receives storage values on the paths specified above. Its result does not select the function’s returned status. |

Dependency names describe roles observed here, not verified internal semantics. No independent dependency implementation or interface contract is supplied.

## Acceptance examples

These are conditional criteria derived from static evidence, not reported executions. Dependencies are assumed to return normally, with valid accesses and usable state for each described step. Any required state stability is stated explicitly.

1. **Initial request returns zero.**  
   The result is 1. No later dependency is called, and this function does not explicitly write the output slot.

2. **Initialization fails.**  
   If working storage is nonzero and initialization returns 12, working-storage cleanup precedes return 12. Processing and finalization are absent.

3. **Initial bound failure.**  
   If the first comparison reads position 21 with bound 20, working-storage cleanup precedes return 7. Processing is not called.

4. **Equality and negative values.**  
   Position 20 with bound 20 permits processing. With bound −2, position −1 fails the comparison, while position −3 permits processing. Dependency acceptance of negative values remains unresolved.

5. **Processing failure overrides completion.**  
   If processing returns 9 while writing completion 1, the function cleans up working storage and returns 9. No final-stage request or publication follows.

6. **Completion requires exactly one.**  
   If processing returns zero and the subsequent completion read yields 2, another bound check follows. If that check reads position above the bound, the result is 7 without another processing call.

7. **No post-completion bound check.**  
   If processing returns zero and the completion read yields 1, final-stage requests follow even if position now exceeds the bound.

8. **Live processing input.**  
   If initialization supplies processing/finalization value 30, but the first processing call changes its escaped slot to 40 and requests continuation, the next processing call receives 40 if the slot remains 40 until that call’s argument read.

9. **Stable final size.**  
   If completed processing supplies final size 100, and the slot remains 100 through the relevant later reads, final-stage request arguments are 100 and 104, and finalization receives size 100. Publication precedes finalization; temporary cleanup then precedes working-storage cleanup.

10. **Final size changes between requests.**  
    If the first request reads 100 and changes the slot to 200, the second request uses 204 provided its argument read sees 200. If the second request then changes the slot to 300, finalization receives 300 provided its argument read sees 300. Neither request argument is retroactively changed.

11. **Other escaped state changes.**  
    If a storage call changes the finalization-only slot through a retained address, finalization receives the changed value when that value persists until its argument read. It is not required to receive the original initialization value.

12. **Unchecked results and finalization error.**  
    A zero final-stage storage result does not itself stop execution. A zero second result is published. If finalization then returns 6, temporary cleanup and working-storage cleanup still occur, and the function returns 6 without explicitly restoring the output slot or cleaning up published storage. Whether finalization safely accepts zero storage values is documented_only.

## Uncertainties

### Established static evidence

The evidence establishes the signed pre-processing comparison, exact completion value 1, literal values 512 and 4, locally selected statuses 1 and 7, dependency call order, separate final-size reads, explicit publication, cleanup ordering, and status propagation.

It also establishes address exposure for the local regions described above. It does not establish that those regions remain unchanged across intervening calls.

### Inferred intent

The operation appears intended to process bounded input until completion and then construct a published result using temporary storage. This interpretation is not required for the control-flow contract and does not establish a data format or ownership model.

### documented_only

Unresolved properties include:

- Meaning and units of the opaque inputs, bound, position, and final size.
- Dependency argument types beyond caller-visible representations.
- Storage capacity, alignment, address interpretation, and zero-size behavior.
- Arithmetic behavior when adding four exceeds the signed 32-bit range.
- Which successful calls initialize all subsequently consumed state.
- Actual retained-address sharing and mutations performed by dependencies.
- Progress, termination, concurrency, and reentrancy guarantees.
- Ownership and release obligations for every storage value.
- Whether finalization consumes or releases supplied storage.
- Global-state effects, external I/O, and aliasing restrictions.
- Dependency status meanings and abnormal-exit behavior.

Unknown behavior is not replaced by assumptions of state preservation, ownership transfer, checked arithmetic, or safe null handling.

## Evidence

This contract uses only the supplied static function body and opaque dependency mapping. The earlier draft and reviewer feedback are not treated as independent behavioral evidence.

The final-stage state distinction follows from separate uses of the address-exposed final-size slot before each storage request and again for finalization. The same live-state treatment applies to other exposed regions wherever their contents are subsequently consumed.

No runtime tests have been performed. All acceptance examples describe conditional consequences of the supplied static evidence.
