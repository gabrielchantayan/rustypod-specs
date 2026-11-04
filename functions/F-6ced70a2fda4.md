<!-- Trusted provenance: F-6ced70a2fda4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6ced70a2fda4

## Purpose

F-6ced70a2fda4 conditionally changes a byte-sized object state and invokes an object-specific indirect operation.

F-592bf430b661 supplies a zero-versus-nonzero decision. A nonzero result selects an active-state request; zero selects an inactive-state request. If the object’s tested state already agrees with that request, this function returns without changing the state or invoking the transition-path dependencies.

Otherwise, the function writes the requested state byte, reads a branch-selected global value, passes that value to F-9b9c1c7483f4, dispatches the resulting value through the object, and calls F-8a5aadac202f.

“Active,” “inactive,” “construction,” and “cleanup” are descriptive role labels. The supplied static evidence does not establish application-level meaning, resource ownership, or allocation behavior.

## Interface

### Arguments and result

The recovered interface has four arguments and no return value.

| Argument | Recovered representation | Visible use |
|---|---|---|
| Object | Pointer to storage represented as 32-bit integer elements | Supplies the tested state and the indirect-dispatch table address |
| Second argument | Opaque 32-bit value | Not referenced by this body |
| Third argument | Opaque 32-bit value | Copied into an invocation-local four-byte cell whose address is passed to F-9b9c1c7483f4 |
| Fourth argument | Opaque 32-bit value | Copied into a separate invocation-local four-byte cell whose address is passed to F-9b9c1c7483f4 and later F-8a5aadac202f |

The third and fourth arguments are copied as values. The addresses passed onward designate the local cells, not caller-owned argument storage. The values may encode pointers, handles, or other data; their interpretation is not established.

F-9b9c1c7483f4 produces an opaque 32-bit result. That result becomes the second explicit argument to the indirect operation. It is not returned to this function’s caller. Results from the indirect operation and F-8a5aadac202f are not used.

### Object layout

The recovered representation establishes these accesses:

| Byte offset | Access |
|---|---|
| Object + 0 | A 32-bit first word interpreted as the dispatch-table address |
| Object + 176, or 0xB0 | A 32-bit element is narrowed to a character for the state comparison; a transition explicitly writes one byte at this offset |
| Dispatch table + 284, or 0x11C | Indirect operation entry, interpreted as a function pointer |

The state comparison uses the narrowed value, not the entire 32-bit element as a Boolean. The transition store replaces only one byte. It does not explicitly replace the containing four-byte element.

Under the recovered four-byte entry model, the dispatch slot occupies four bytes. Exact machine load widths, byte order, pointer ABI, and calling convention are not independently established. In particular, the relationship between numerical narrowing of the state element and the byte at its starting address depends on representation details.

### Branch-selected global inputs

Two separate global locations supply values to F-9b9c1c7483f4:

- The **active global input** is read only on the transition from a zero tested state to an active request.
- The **inactive global input** is read only on the transition from a nonzero tested state to an inactive request.

These labels distinguish the locations by branch role. The call passes the **value read from the selected global location**, not the address of that location. The source does not establish that either value is an immutable constant, nor that the two values differ.

The declarations and stored types of these globals are not supplied. Their exact widths and semantic types therefore remain undocumented. Their read timing is established even though their representation is not.

### Preconditions

For normal execution of the selected path:

- F-592bf430b661 must return normally.
- The object must support the state access after that dependency returns.
- On a transition, the state byte must be writable.
- The selected global location must support the value read used to prepare the construction call.
- The local cells and the selected global value must satisfy F-9b9c1c7483f4’s unknown input requirements.
- After construction returns, the object’s first word and selected table entry must support a valid indirect call.
- After dispatch returns, the fourth-argument cell must satisfy F-8a5aadac202f’s unknown requirements.

The body performs no explicit null, bounds, alignment, callable-target, or construction-result validation. These preconditions concern the memory and values present when each access occurs; the body does not establish that dependencies preserve them.

## Behavior

### Decision and local copies

F-592bf430b661 is called first, without explicit arguments. Its result is compared with zero. No restriction to Boolean return values is visible.

After this call returns, the third and fourth arguments are copied into their separate four-byte local cells. These assignments precede the state comparison in the supplied representation, including on paths that subsequently return without a transition.

The state comparison occurs after F-592bf430b661 returns. Consequently, it observes the state then present, not necessarily the state that existed at function entry. The dependency is not established to preserve object storage, global inputs, or other aliased memory.

### State decision

| Decision result | Narrowed state value | Direct behavior |
|---|---|---|
| Nonzero | Nonzero | Return without transition |
| Nonzero | Zero | Write byte value 1 and perform the active transition |
| Zero | Zero | Return without transition |
| Zero | Nonzero | Write byte value 0 and perform the inactive transition |

Every nonzero decision result selects the same request. Every nonzero narrowed state value counts as active for comparison purposes.

An already-active value is not normalized. For example, a nonzero decision with a tested value of seven takes the no-transition path rather than rewriting the byte to one.

On either no-transition path, this body performs no selected-global read, construction call, dispatch-table lookup, indirect dispatch, or cleanup call. The earlier decision call still occurs.

### Active transition

When the decision is nonzero and the tested state is zero, the observable sequence is:

1. Write byte value 1 at object offset 176.
2. Read the current value from the active global input while preparing the construction call.
3. Call F-9b9c1c7483f4 with the fourth-argument cell’s address, the value just read, and the third-argument cell’s address, in that argument order.
4. After construction returns, obtain the dispatch-table address from the object’s first word and the operation from table offset 284.
5. Invoke that operation with the object and the 32-bit construction result.
6. After dispatch returns normally, call F-8a5aadac202f with the fourth-argument cell’s address.
7. Return without a value.

The ordering above does not impose an additional ordering among call-argument preparation operations where none is established. It does establish that the selected global read follows the state-byte store and precedes entry into F-9b9c1c7483f4.

### Inactive transition

When the decision is zero and the tested state is nonzero, the same sequence applies with two substitutions:

- The direct state-byte store writes zero.
- The construction argument is the value read from the inactive global input.

The active global input is not read by this path.

### Global-read timing and aliasing

The selected global is read only after the decision dependency has returned and after the transition’s state-byte store. Neither entry-time caching nor preservation of the global across these earlier operations is established.

This matters in two ways:

- If F-592bf430b661 changes the subsequently selected global, construction receives the value present after that change.
- If the state-byte store aliases storage belonging to the selected global, the subsequent global read observes storage after the byte store, subject to the actual representation and valid-access requirements.

An equivalent implementation cannot simply substitute an entry-time snapshot or assume that the selected input is a fixed selector constant. It must preserve the selected read’s branch dependence and its ordering relative to the decision call and state store.

The global’s contents could themselves encode an address. That possibility does not change the established distinction: this body passes the loaded contents, not an address-of expression for the global location.

### Dispatch and cleanup boundaries

The dispatch lookup occurs after F-9b9c1c7483f4 returns. The body does not establish that construction preserves the object’s first word, the table, the selected entry, or other relevant state. Resolving the target earlier would require additional preservation guarantees.

The construction result is forwarded without a visible validity check. A zero result is not rejected by this body.

Cleanup receives the address of the same fourth-argument cell supplied to construction. It can observe that cell’s contents as they exist after both construction and dispatch. The body does not restore the original fourth argument before cleanup.

There is no subsequent state comparison, retry, rollback, or reconciliation with the decision result.

## Side effects

### Direct effects

On a transition, this function explicitly writes one byte at object offset 176. It does not explicitly overwrite the other three bytes of the containing state element.

On a no-transition path, it performs no direct object write.

The body also initializes two invocation-local four-byte cells after the decision call. Their addresses become externally available only on transition paths.

These observations describe the body’s explicit accesses. They do not limit dependency effects. If object storage aliases other storage, the state-byte store may affect that storage as well.

### Dependency effects and abnormal completion

F-592bf430b661 is invoked before any state comparison. On a normally completed transition, F-9b9c1c7483f4, the indirect operation, and F-8a5aadac202f each run once, in that order.

Their effects are not established. In particular, they may affect globals or aliased object storage; this contract does not assume preservation.

The state-byte store precedes every transition-path call. If construction does not return normally, dispatch and cleanup are not established to occur. If dispatch does not return normally, the subsequent cleanup is not established to occur. No rollback or guaranteed exceptional cleanup is visible.

The body has no explicit error return or local error-reporting branch.

### Ownership and lifetime

The two local cells belong to the current invocation. Their addresses do not acquire a lifetime beyond the invocation merely because a dependency receives them.

The supplied evidence establishes no ownership transfer for:

- The third or fourth argument.
- Resources represented by either argument.
- The construction result.
- Resources reachable through either local cell.
- The selected global value.

F-8a5aadac202f’s position suggests a cleanup role, but does not prove destruction, deallocation, or release. No separate cleanup call for the third-argument cell or construction result is visible.

If a dependency retains a local-cell address, validity after this invocation returns is not provided by this body.

### Reentrancy and concurrency

A reentrant call made after the state store can encounter the updated state, unless intervening effects change it. This ordering alone does not guarantee suppression of duplicate dispatch.

No synchronization, atomic-access protocol, or memory barrier is established. Concurrent behavior cannot be derived from the sequential ordering alone.

## Dependencies

| Dependency | Established interaction |
|---|---|
| F-592bf430b661 | Called first without explicit arguments; its zero-versus-nonzero result selects the requested state |
| F-9b9c1c7483f4 | Called only on a transition; receives pointers to the fourth- and third-argument cells around the selected global value; returns an opaque 32-bit dispatch argument |
| Object-selected indirect operation | Resolved after construction through the object’s first word and table offset 284; receives the object and construction result; return value unused |
| F-8a5aadac202f | Called after normally returning dispatch with the fourth-argument cell’s address; return value, if any, unused |

No opaque function ID is supplied for the indirect target. Its identity can vary with the object and with changes made before lookup.

The active and inactive global inputs are data dependencies, not additional functions. Only the selected transition path reads its corresponding global.

There is no visible arithmetic on the opaque argument values, selected global value, or construction result. Object and table accesses use fixed offsets; no variable indexing, range calculation, or wraparound-based behavior is shown.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Unless stated otherwise, all accesses are valid and every invoked dependency returns normally.

### Already active without normalization

Given a nonzero decision and a narrowed state value of seven:

- No direct state write occurs.
- Neither branch-selected global is read by this body.
- Construction, dispatch lookup, dispatch, and cleanup are skipped.
- Any effects of the decision dependency still apply.

### Already inactive

Given a zero decision and a narrowed state value of zero, the function returns without a transition. A valid dispatch-table entry is not required by the visible accesses on this path.

### Active transition with a changed global

Suppose F-592bf430b661 changes the active global input and returns nonzero, and the subsequently tested state is zero.

The state-byte store writes one first. Construction then receives the active global’s value as read after that store, not an entry-time snapshot and not the global’s address.

### Inactive transition with distinct global contents

Given a zero decision and a nonzero tested state, the function writes zero and reads the inactive global input. It does not read the active global input, even if that other location has a different value.

If the two locations contain equal values, the branch still determines which location is read.

### State-store aliasing

Suppose the object’s state byte overlaps part of the selected global’s storage and both accesses are valid under the applicable representation.

The selected global read follows the byte store. Construction therefore receives the value represented by the storage after that write. Its exact numerical value cannot be specified without the global’s width, layout, and byte order.

### Noncanonical active state

Given a zero decision and a narrowed state value whose byte pattern is all ones, the inactive transition occurs. Whether character interpretation makes that narrowed value negative does not change its inequality to zero.

### Construction changes the dispatch source

If construction changes the object’s table address through an alias, the subsequent lookup uses the address present after construction returns, provided the resulting lookup and call remain valid.

### Temporary mutation and cleanup

If construction changes the fourth-argument cell, and dispatch leaves that change intact, cleanup receives the same cell with the changed contents. If dispatch changes it again through an available alias, cleanup instead encounters that later state.

### Downstream nonreturn

If construction fails to return normally, the state store and selected-global read have already occurred. Dispatch and cleanup are not guaranteed. If dispatch fails to return normally, cleanup is not guaranteed.

## Uncertainties

The following remain **documented_only**:

- The application meaning of the state, decision, global inputs, and dispatched value.
- The selected globals’ declarations, widths, representations, mutability rules, and ownership implications.
- Whether either global value encodes an address.
- The exact machine load widths, byte order, pointer representation, and calling convention.
- The precise relationship between integer narrowing and the byte addressed by the store on the target representation.
- Dependency mutation, allocation, release, retention, reentrancy, and failure behavior.
- Any preservation of globals, object fields, tables, or aliased storage across calls.
- Exception handling or unwinding outside this body.
- External synchronization and concurrent-access requirements.

The second argument has no visible use here. That does not establish that it can be removed from the wider interface.

## Evidence

The supplied static representation establishes the decision-first call, subsequent local copies, zero-versus-nonzero state tests, transition-only byte stores, and branch-selected global value reads.

It specifically places each selected global read after the corresponding state store, as an argument value for F-9b9c1c7483f4. The expressions establish value reads, not addresses of globals or immutable constants.

The representation also establishes construction before indirect-target lookup, dispatch before cleanup, four-byte opaque argument cells and construction result, fixed object and table offsets, and absence of visible result validation or rollback.

Application-level role labels are inferred intent only. Unknown dependency effects and missing representation details remain explicitly unresolved.

No runtime tests have been performed.
