<!-- Trusted provenance: F-a56eafe76a58; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a56eafe76a58

## Purpose

F-a56eafe76a58 transfers up to a requested number of single-byte values from a receiver-mediated source into caller-provided storage. Before obtaining each byte, it invokes a receiver-selected status operation. A zero status permits one byte acquisition; any nonzero status stops the transfer.

The result is the number of bytes directly written by this invocation. Reaching the requested count returns that count without a further status check.

The established behavior is a bounded, status-gated byte transfer. Interpreting the status as end-of-input, availability, or an error indication is inferred intent, not an established contract. The routine does not distinguish those meanings.

## Interface

### Inputs

The routine has three arguments:

1. **Receiver reference.** A reference to an object whose leading storage contains a dispatch-table address. Two indirect operations are selected through that table:
   - A status operation at byte offset 24.
   - A byte-producing operation at byte offset 32.

   Each operation receives the original receiver reference as its sole explicit argument.

2. **Destination address.** The starting address for consecutive single-byte stores. The destination is passed by value; the routine advances its own position rather than returning or directly updating a caller-owned destination pointer.

3. **Requested count.** An unsigned integer bounding the number of bytes to transfer.

The supplied representation uses 32-bit unsigned quantities for the requested count, progress count, and result, a 32-bit integer status result, and an 8-bit transfer value. The excerpt does not independently establish the target ABI or pointer width; these integer widths are the working interpretation of its represented types. Pointer width must not be inferred from the dispatch-table offsets.

### Output

The routine returns an unsigned transferred-byte count:

- The requested count when that many byte stores have completed.
- The number of completed byte stores when a status operation returns nonzero before the request is satisfied.

No distinct error result, status enumeration, output terminator, or end-of-input flag is produced.

### Preconditions

For a zero requested count, this routine performs no receiver dereference, dependency invocation, or destination access. It therefore requires no usable receiver or destination storage merely to execute that path.

For a positive requested count, defined execution requires the following whenever the corresponding operation is reached:

- The receiver’s dispatch-table address can be read.
- The selected table entry can be read and invoked using the expected calling convention.
- Each invoked dependency returns normally if normal completion is expected.
- Every destination byte actually written is writable.
- The address progression used for those stores is valid for the target execution environment.

Because the status dependency runs before the byte dependency, validity at entry alone is insufficient if a dependency mutates or invalidates receiver-related storage. The byte-producing target is selected using a fresh dispatch-table lookup after the successful status call.

The routine performs no explicit null, capacity, dispatch-table, or callable-target validation.

## Behavior

### Initial state and request limit

The completed-transfer count starts at zero. The destination position starts at the supplied destination address.

Before any status check, the routine compares the completed count against the requested count. If the request is already satisfied, it returns the requested count immediately.

This ordering establishes two boundaries:

- A zero-length request makes no indirect calls.
- After the final requested byte has been stored, the routine returns without probing for another byte.

The request is a by-value limit. There is no explicit reread of a caller-owned count field.

### Status selection and stopping

When more bytes are requested, the routine reads the receiver’s current dispatch-table address and selects the operation at byte offset 24. It invokes that operation with the receiver reference.

Only whether the returned status equals zero matters:

- **Zero:** proceed to obtain one byte.
- **Nonzero:** stop immediately and return the completed-transfer count.

Positive and negative nonzero status values have the same stopping effect under the represented signed status interpretation. No nonzero value is propagated to the caller.

A stopping status causes no byte-producing call and no direct destination store for that iteration. Previously completed stores are not rolled back.

### Byte selection and storage

After a zero status, the routine again reads the receiver’s dispatch-table address. It selects the operation at byte offset 32 and invokes it with the same receiver reference.

The value used from this invocation is one byte. The routine then:

- Advances its completed-transfer count by one.
- Stores the acquired byte at the current destination position.
- Advances the destination position by one byte.

It subsequently reevaluates the request limit.

The status operation and byte-producing operation are separate calls. The routine provides no atomicity between them and performs no second status check before storing the returned byte.

All byte values are ordinary data. In particular, a zero byte does not terminate the transfer. There is no content-based filtering, text conversion, terminator insertion, or padding.

### Dispatch lookup ordering

The routine does not retain one dispatch-table address for the entire transfer. It also does not reuse the status lookup’s table address to select the byte-producing operation.

The observable selection order for each successful iteration is:

1. Read the receiver’s dispatch-table address for the status operation.
2. Read the entry at offset 24 and invoke it.
3. After a zero result, read the receiver’s dispatch-table address again.
4. Read the entry at offset 32 and invoke it.
5. Store the acquired byte.

Consequently, a status operation that changes the receiver’s dispatch table can change which byte-producing operation is called immediately afterward. A byte-producing operation or an earlier destination store may likewise affect a later lookup if the relevant memory is aliased.

This does not establish that such mutations are intended or safe. It establishes that the routine contains no cached-table guarantee.

### Count arithmetic

Under the represented 32-bit unsigned model, the requested count ranges from zero through 4,294,967,295.

The count increments only when its current value is below the requested count. Thus, the routine’s count arithmetic does not wrap:

- The greatest possible increment changes 4,294,967,294 to 4,294,967,295.
- The next limit check returns before another increment.

This arithmetic property does not establish that a destination of that size is addressable or valid.

The destination position advances after every successful store, including the final requested store. Address validity and address-space limits remain separate from the bounded count arithmetic.

## Side effects

### Direct memory effects

The routine’s explicit writes are one-byte stores at consecutive destination positions, in increasing address order.

If it returns normally with result \(k\), it has performed exactly \(k\) such stores. Their addresses are the supplied destination address through that address plus \(k-1\), when \(k\) is positive.

This is a statement about the routine’s direct stores, not necessarily the final contents of that region. Dependencies may modify destination memory through aliases or other state. Likewise, the absence of direct stores beyond the transferred prefix does not prove that dependencies leave the remaining destination storage unchanged.

The routine does not explicitly clear unused destination space.

### Receiver and dependency effects

Both indirect calls receive the receiver reference. Their state changes are unknown. They may affect receiver contents, dispatch selection, source position, shared state, or other accessible memory.

The routine contains no save-and-restore mechanism for dependency-visible state. It does not undo effects when a later status stops the transfer.

The byte-producing operation is not passed the destination address or requested count explicitly. Whether it can access either through aliases or shared state is unknown.

### Ownership and lifetime

There is no explicit allocation, deallocation, ownership transfer, reference-count update, or resource release in the shown routine.

That observation does not constrain the dependencies: they may allocate, release, retain references, or invalidate objects. Their ownership contracts are unavailable.

The routine does not explicitly retain its receiver or destination for later use. Any retention performed by a dependency remains unknown.

### Failure and concurrency

No local error translation or recovery mechanism is shown. Invalid addresses, invalid indirect targets, nonreturning dependencies, and nonlocal control transfers have no specified normal-result behavior here.

No synchronization is shown. Thread safety, reentrancy, and consistency under concurrent receiver or destination mutation are documented_only unknowns.

## Dependencies

The following opaque identifiers label unresolved indirect dependency roles. They are not resolved function identities, and a role may select different concrete targets across calls.

| Opaque dependency ID | Selection | Explicit input | Result used | Established role |
|---|---|---|---|---|
| F-unresolved-001 | Current receiver dispatch table, byte offset 24 | Receiver reference | Integer tested against zero | Gates each attempted byte transfer |
| F-unresolved-002 | Fresh receiver dispatch-table lookup, byte offset 32 | Receiver reference | One byte | Supplies the value for one destination store |

For a normally returning invocation with result \(k\):

- F-unresolved-002 is called exactly \(k\) times.
- If \(k\) equals the requested count, F-unresolved-001 is called exactly \(k\) times.
- If \(k\) is less than the requested count, F-unresolved-001 is called exactly \(k+1\) times, with its last result nonzero.

These counts describe calls directly issued by this invocation. They exclude additional calls made internally or recursively by dependencies.

No dependency is known to preserve global state, receiver state, destination contents, ownership, or dispatch-table identity.

## Acceptance examples

These are conditional behavioral examples derived from the supplied static evidence. They are not executed tests.

### Zero-length request

Given a requested count of zero:

- The result is zero.
- Neither dependency is invoked.
- The receiver is not dereferenced.
- No destination store occurs.

This path does not require valid receiver or destination storage.

### Immediate stop

Given a positive request and an initial status result that is nonzero:

- The status dependency is invoked once.
- The byte dependency is not invoked.
- The result is zero.
- This routine performs no destination store.

Any effects of the status dependency itself remain possible.

### Full transfer

Given a request for three bytes, three zero status results, and byte results of decimal 17, 0, and 255:

- The routine directly stores those bytes at destination offsets zero, one, and two, respectively.
- Each status call precedes its corresponding byte call.
- The result is three.
- There is no fourth status call.
- The zero-valued second byte does not stop the transfer.

Final memory equals those values only if dependencies and other actors do not subsequently modify them.

### Partial transfer

Given a request for five bytes and status results of zero, zero, then nonzero:

- Exactly two byte-producing calls occur.
- Their results are stored at destination offsets zero and one.
- The third status call stops the transfer.
- The result is two.
- The routine performs no direct store at destination offset two or later.

### Changed dispatch table

Given a zero-returning status operation that replaces the receiver’s dispatch-table address:

- The subsequent byte-producing target is selected from the replacement table at offset 32.
- The prior table is not reused for that selection.

This example requires the replacement table and selected target to remain valid.

### Largest represented request

Given a request of 4,294,967,295, continuously zero status results, normally returning byte operations, and valid destination progression:

- The count reaches the requested value without unsigned wrap.
- The result equals the request.
- No extra status call occurs after the last store.

This is an arithmetic boundary example, not evidence that such storage is available.

## Uncertainties

The following remain documented_only:

- The concrete identities and complete contracts of both indirect dependencies.
- Whether nonzero status means exhaustion, failure, temporary unavailability, or another condition.
- Whether status checks consume or otherwise change source state.
- Whether byte-producing calls consume exactly one source byte.
- Dependency effects on globals, ownership, lifetime, receiver state, and destination aliases.
- The target ABI, pointer width, alignment requirements, and independent confirmation of the represented integer widths.
- Permitted overlap between destination storage, receiver storage, and dispatch tables.
- Exception behavior, nonlocal exits, blocking, cancellation, concurrency, and reentrancy.
- Any external requirement that the destination hold the entire requested count rather than only the bytes actually reached.

The routine itself neither establishes nor enforces answers to these questions.

## Evidence

**Established static evidence:** The supplied function representation contains a zero-initialized progress count, a pre-call request-limit check, an indirect status call through offset 24, a nonzero stopping branch, a separately selected byte call through offset 32, a one-byte destination store, and bytewise destination advancement.

**Derived properties:** The call-count relationships, absence of a final extra status check, treatment of zero bytes as data, and nonwrapping progress arithmetic follow from that control flow under normal return and the stated width model.

**Inferred intent:** The arrangement resembles a bounded source-to-buffer transfer. Names such as end-of-input, read, stream, or availability would impose meanings not established by the evidence.

**Verification status:** No runtime tests have been performed. Acceptance examples are static contract consequences, not observed execution results.
