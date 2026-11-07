<!-- Trusted provenance: F-bfbc686b3cef; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bfbc686b3cef

## Purpose

F-bfbc686b3cef conditionally issues refresh-related or start-related requests through a service object and notification dependencies. One branch also writes a byte in the caller-supplied receiver and invokes a receiver callback.

The established outcomes are:

- An initial service query returns zero: no temporary request is prepared and no further action is initiated by this function.
- Request evaluation returns 1 or 2: the request temporary is finalized without refresh or start processing.
- Request evaluation returns zero and an availability query returns zero: the function writes the receiver flag, issues a refresh notification, and invokes a loading-screen-related receiver callback.
- Request evaluation returns zero and availability is nonzero: an additional predicate determines whether the start-related sequence runs.
- Request evaluation returns any other 32-bit value: a dependency annotated as non-returning is invoked before request finalization.

Every normally returning path returns the 32-bit value 1. That value does not distinguish skipped processing from an issued request.

“Refresh,” “loading screen,” “availability,” and “start” are descriptive labels. The message names support the inferred intent of requesting refresh, navigation, and start behavior; they do not establish successful completion of those operations.

## Interface

### Input and object layout

The function accepts one receiver pointer. The supplied representation uses 32-bit integer and address-bearing values.

The function directly accesses the receiver only on the refresh branch:

| Location | Established access |
|---|---|
| Receiver byte offset 0 | Reads the virtual-table pointer for receiver dispatch. |
| Receiver byte offset 0xD0, decimal 208 | Writes the single byte value 1. |
| Receiver virtual-table byte offset 0x11C, decimal 284 | Reads the callback entry invoked with the receiver and a constructed loading-screen argument. |

The receiver write is one byte, not a four-byte integer store. Its offset follows from advancing 0x34 elements in the supplied four-byte integer view: 52 × 4 = 208 bytes.

No explicit null, bounds, alignment, or object-validity check precedes the receiver write or dispatch. No surrounding receiver-field meanings or byte order are established.

The function also obtains a service-object pointer from F-6da76fc58c80. The service is used through these virtual-table byte offsets:

| Offset | Established use |
|---|---|
| 0xA0, decimal 160 | Initial query returning a 32-bit value. |
| 0x9C, decimal 156 | Query returning a 32-bit value later supplied to the start predicate. |
| 0xA8, decimal 168 | Entry invoked with the service receiver and the 32-bit argument 1 when the start predicate accepts. |

The service pointer is retained. Each virtual dispatch obtains its entry through the service object at that point; the function does not establish that intervening dependencies preserve the service’s virtual-table pointer.

### Local storage and output

The visible local storage includes:

- A 24-byte request region.
- A four-byte argument-storage region reused for each named argument.
- A separate four-byte auxiliary region passed to every argument-construction call.
- A four-byte refresh payload assigned the value 1 before the refresh notification.

The auxiliary region has no visible initialization before its first use. Its address is supplied to the constructor; its role and whether the constructor reads or writes it are unknown.

On normal completion, the output is the 32-bit value 1. There is no separate returned status identifying which branch ran.

### Preconditions

Normal execution of a selected path requires:

- A usable service pointer from F-6da76fc58c80.
- Valid service dispatch entries for every virtual call reached on that path.
- Dependencies compatible with the supplied local storage and arguments.
- On the refresh branch, a receiver permitting the byte write and receiver dispatch.

There is no visible recovery for invalid pointers, incompatible temporary-storage requirements, failed construction, or abnormal dependency termination. A path that does not directly access the receiver does not establish that an arbitrary receiver pointer is universally safe: aliasing and dependency-mediated accesses remain unspecified.

## Behavior

### Initial service query

The function first calls F-6da76fc58c80 without explicit arguments. It retains the returned service pointer, reads the service dispatch entry at offset 0xA0, and invokes that entry with the service as receiver.

If the returned 32-bit value is zero, the function returns 1 immediately. On this path:

- No request temporary is initialized or finalized.
- No request evaluation occurs.
- No availability query or additional predicate occurs.
- No named argument is constructed.
- This function performs no direct receiver access.

Any nonzero initial value continues to request preparation. No positive-only restriction is present.

### Request preparation and evaluation

The function calls F-ba5a4c1e0746 with the 24-byte request region and two zero arguments.

After that call returns, it calls F-d73bd4e369fe with the nonzero initial-query value and the address of the same request region. It then calls F-504b664a4c49 with the retained service pointer and that region’s address.

The storage identity and ordering are established. The request’s internal schema and the meanings of the two zero arguments are not.

The 32-bit evaluation result selects the next step:

| Result | Behavior |
|---|---|
| 0 | Continue to the service-value and availability queries. |
| 1 | Skip further processing and finalize the request. |
| 2 | Skip further processing and finalize the request. |
| Any other value | Invoke F-a817d8d58c10 without explicit arguments. |

Results 1 and 2 have identical visible follow-on behavior.

F-a817d8d58c10 is annotated as non-returning in the supplied static evidence. The request is not finalized before this call. A normal return or cleanup sequence after an unexpected return from this dependency is not a supported contract.

### Service-value capture, availability, and notification-target read

For evaluation result zero, the function:

1. Invokes the service virtual entry at offset 0x9C with the retained service pointer.
2. Retains its 32-bit result.
3. Calls F-40819ce9a421 without explicit arguments.
4. Reads a shared notification-target value and retains that value locally.

The target read occurs after F-40819ce9a421 returns, on both availability outcomes. It also occurs when the later start predicate will reject processing.

Consequently, a change to the shared target made by F-40819ce9a421 can affect this invocation’s retained target. A later change to that global by another dependency does not replace the locally retained argument value. Retaining the value does not establish that any object it denotes remains unchanged or alive.

The captured result from service entry 0x9C is passed to the additional predicate only on the nonzero-availability branch. Its producing call still occurs on the refresh branch, even though that result is not subsequently consumed there.

### Refresh branch

If F-40819ce9a421 returns zero, the function performs these operations in order:

1. Writes the byte value 1 at receiver offset 0xD0.
2. Calls F-9b9c1c7483f4 with the four-byte argument-storage region, the name `"RefreshingGenius"`, and the auxiliary region’s address.
3. Assigns the 32-bit value 1 to the local refresh payload.
4. Calls F-7cbd4a877291 with the retained notification target, the value returned by argument construction, and the refresh payload’s address.
5. Calls F-8a5aadac202f with the argument-storage address.
6. Calls F-9b9c1c7483f4 again, reusing the argument-storage and auxiliary regions, with the name `"GotoGeniusLoadingScreen"`.
7. Reads the receiver’s virtual dispatch entry at offset 0x11C and invokes it with the receiver and the second constructor’s returned value.
8. Calls F-8a5aadac202f again with the argument-storage address.

The notification and callback receive the respective constructor return values. The evidence does not require those values to equal the supplied storage address.

The receiver byte write precedes both argument constructions. It occurs even if that byte already contains 1. This function does not explicitly clear or restore it.

The receiver dispatch is resolved after the refresh notification, first argument finalization, and second construction. No assumption is made that those intervening dependencies preserve the receiver’s virtual-table pointer or other state.

The local payload is set to 1 immediately before the notification call. Because its address is passed, the dependency may access that storage according to its unknown contract; the function does not read the payload afterward.

### Nonzero-availability branch

If F-40819ce9a421 returns nonzero, the function calls F-4e6b58fde38f with the 32-bit value captured from service entry 0x9C.

If this predicate returns zero, no start argument is constructed, service entry 0xA8 is not invoked, and F-10655c9db871 is not called. The function proceeds to request finalization.

If the predicate returns nonzero, the sequence is:

1. Invoke the service virtual entry at offset 0xA8 with the retained service pointer and the 32-bit argument 1.
2. Call F-9b9c1c7483f4 with the argument-storage region, the name `"StartGenius"`, and the auxiliary region’s address.
3. Call F-4b9875325552 with the retained notification target, the constructor’s returned value, and the address of separate global payload storage.
4. Call F-8a5aadac202f with the argument-storage address.
5. Call F-10655c9db871 without explicit arguments.

The dispatch at offset 0xA8 and its argument are established. Whether that entry changes service state, changes other state, or has no state-changing effect is unknown.

The third notification argument is the address of global storage, not a local copy of its contents. The payload’s size, representation, value, and mutability are not established.

This branch contains no direct receiver access.

### Finalization and return

Every normally completed path that initialized the request region calls F-4e292a3dd745 with its address before returning 1.

On the refresh branch, both argument finalizations precede request finalization. On the accepted-start branch, argument finalization precedes F-10655c9db871, which precedes request finalization.

The initial-query-zero path bypasses request finalization because it also bypasses request initialization. The annotated non-returning path reaches F-a817d8d58c10 before request finalization.

## Side effects

### Direct writes and reads

The only direct write to the caller’s receiver is the one-byte store of 1 at offset 0xD0.

Other explicit writes concern local storage, including the local refresh payload. This function performs no visible direct assignment to shared global storage.

It reads the shared notification-target value only after the availability query. It supplies the address of a separate global payload to the start notification without visibly reading or copying that payload’s contents.

These observations do not imply that globals, the service, or the receiver remain unchanged across dependency calls.

### Dependency-mediated effects

The function may:

- Obtain and query a service object.
- Prepare, evaluate, and finalize a request temporary.
- Construct and finalize named arguments.
- Invoke notification dependencies.
- Invoke the receiver’s loading-screen-related virtual entry.
- Invoke the service virtual entry at offset 0xA8 with argument 1.
- Invoke F-10655c9db871.
- Invoke the dependency annotated as non-returning.

The actual effects of the service entry at offset 0xA8, and of the other unknown dependencies, remain unspecified.

Return values from notification calls, finalization calls, the receiver callback, service entry 0xA8, and F-10655c9db871 do not control subsequent visible branching.

### Ownership and lifetime

All local regions exist only for this invocation. Their addresses are passed to dependencies, but no evidence establishes that callees copy them, refrain from retaining them, or take ownership of resources associated with them.

Completed named-argument constructions are paired with F-8a5aadac202f on normally proceeding paths. Request preparation is paired with F-4e292a3dd745 on normally returning paths that reach preparation. These pairings support a lifecycle-management interpretation, not a specific allocation, release, or reference-counting contract.

The auxiliary region is reused without an explicit reset. In particular, no separate initialization appears before its second use on the refresh branch.

There is no explicit service-object release. Whether any dependency acquires or releases resources is **documented_only**.

## Dependencies

| Opaque ID or dispatch point | Established relationship |
|---|---|
| F-6da76fc58c80 | Returns the retained service pointer. |
| Service entry 0xA0 | Produces the initial 32-bit zero/nonzero value. |
| F-ba5a4c1e0746 | Receives request storage and two zero arguments. |
| F-d73bd4e369fe | Receives the initial nonzero value and request storage. |
| F-504b664a4c49 | Receives service and request storage; returns the branch-selecting evaluation result. |
| F-a817d8d58c10 | Called for evaluation results outside 0, 1, and 2; annotated as non-returning. |
| Service entry 0x9C | Produces the value later passed to the start predicate. |
| F-40819ce9a421 | Selects refresh handling on zero and predicate handling on nonzero. |
| F-4e6b58fde38f | Gates the start sequence using the captured service value. |
| Service entry 0xA8 | Invoked with service and argument 1 before start-argument construction; actual effects unknown. |
| F-9b9c1c7483f4 | Receives argument storage, a name, and auxiliary storage; returns the value passed onward. |
| F-7cbd4a877291 | Receives the refresh target, constructed value, and local payload address. |
| F-4b9875325552 | Receives the start target, constructed value, and global payload address. |
| F-8a5aadac202f | Receives argument storage after each completed use. |
| Receiver entry 0x11C | Receives the receiver and constructed loading-screen value. |
| F-10655c9db871 | Runs after start-argument finalization. |
| F-4e292a3dd745 | Receives request storage on normally completed prepared-request paths. |

These are call relationships, not independent behavioral specifications for the dependencies.

## Acceptance examples

The following are conditional acceptance scenarios derived from static evidence. None has been executed.

### Initial query returns zero

Given a usable service whose entry 0xA0 returns zero, the function returns 1 without request preparation, request evaluation, availability querying, named-argument construction, or direct receiver access.

### Evaluation returns 1 or 2

Given a nonzero initial value and evaluation result 1 or 2, request preparation and evaluation occur in order. Service entry 0x9C and F-40819ce9a421 are not called. F-4e292a3dd745 receives the prepared request region, and normal return is 1.

### Refresh selected

Given evaluation result zero and availability result zero:

- Service entry 0x9C precedes the availability query.
- The shared target is read after that query.
- Receiver byte 0xD0 is written before `"RefreshingGenius"` construction.
- The refresh notification receives the constructor return value and a pointer to a local 32-bit payload assigned 1.
- First argument finalization precedes loading-screen argument construction.
- Receiver dispatch precedes second argument finalization.
- Request finalization precedes normal return 1.

An initial receiver-byte value of 1 does not suppress the store or subsequent sequence.

### Start accepted

Given evaluation result zero, nonzero availability, and a nonzero predicate result, the predicate receives the previously captured service value. Service entry 0xA8 is invoked with argument 1 before `"StartGenius"` construction.

The notification receives the retained target and global payload address. Argument finalization precedes F-10655c9db871; request finalization follows it. Normal return is 1. Acceptance does not require a particular state change from service entry 0xA8.

### Start rejected

Given nonzero availability and predicate result zero, the shared target has already been read, but no start construction, start notification, service entry 0xA8 invocation, or F-10655c9db871 call occurs. The request is finalized and normal return is 1.

### Target changes during dependencies

If F-40819ce9a421 changes the shared target before returning, the subsequent target read observes that value, absent an intervening external change. If a later constructor changes the global target, notification still receives the earlier retained value.

### Unexpected evaluation result

Given evaluation result 3, or any other 32-bit value except 0, 1, and 2, F-a817d8d58c10 is invoked before request finalization. This includes negative values under a signed interpretation. Normal return is not promised.

## Uncertainties

The following remain **documented_only**:

- Concrete service and receiver types and their broader layouts.
- Semantic meanings of query values and evaluation results 1 and 2.
- Request contents and storage-alignment requirements.
- Constructor return representation and auxiliary-storage requirements.
- Shared target identity, ownership, and validity duration.
- Global payload size, contents, and mutability.
- Actual effects of service entry 0xA8 and all other unknown callees.
- Allocation, ownership transfer, retention, copying, and release behavior.
- Notification delivery timing and reentrancy.
- Exception handling and cleanup after abnormal termination.
- Thread safety and concurrent global updates.
- Successful completion of any requested UI or Genius operation.

The visible function contains no loop, retry, timeout, rollback, or completion check. None is added to this contract.

## Evidence

Established static evidence consists of the supplied function body’s call relationships, branch comparisons, 32-bit values, local-region sizes, dispatch offsets, receiver byte write, shared-target read placement, argument forwarding, finalization order, and constant normal return.

Inferred intent is limited to interpreting the three message names as refresh-, navigation-, and start-related requests, and interpreting paired temporary operations as lifecycle management.

The non-returning classification of F-a817d8d58c10 comes from the supplied static annotation. It is not independently verified.

No dependency implementations or runtime traces were available. No runtime tests have been performed.
