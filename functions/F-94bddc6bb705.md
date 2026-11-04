<!-- Trusted provenance: F-94bddc6bb705; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-94bddc6bb705

## Purpose

F-94bddc6bb705 obtains a destination address, processes an initialized four-byte candidate, and writes two two-byte portions to that destination. It then queries the destination. A zero query result ends the invocation; any nonzero result causes additional processing and a second pair of destination writes.

Each pair of writes has an established order: read the candidate’s low-order 16-bit value, write destination offset 0, read the candidate’s two-byte portion at byte offset 2, and write destination offset 2. There is no established capture of both portions before the first write. Aliasing can therefore affect the second value read.

The conditional path also evaluates an external value and assigns it through a dependency-returned address. The assignment destination is four bytes wide. The external value’s type, source-access width, and conversion behavior are not supplied.

“Candidate,” “destination,” and “context local” identify observable roles only. They do not establish application-level meanings, ownership, or a shared storage layout.

## Interface

The function accepts four arguments and has no explicit return value.

| Argument | Established representation | Visible use |
|---|---|---|
| First | Four-byte value | Passed to F-603a53835f04 to obtain the destination. |
| Second | Four-byte value | Initializes one local and is passed directly to both candidate-processing dependencies. |
| Third | Object pointer in the represented address model | Initializes a separate pointer local, supplies the conditional indirect-call receiver, and is passed directly to F-4062894eb208. |
| Fourth | Four-byte value | Initializes candidate storage. It is not subsequently passed directly to a dependency. |

Three distinct locals are initialized:

| Local role | Initialized contents | Address exposure |
|---|---|---|
| Context local | Second argument | Passed to F-67a6bf796de6 and F-4062894eb208 on the conditional path. |
| Saved pointer local | Third argument | No explicit address exposure or later read is shown. |
| Candidate local | Fourth argument | Passed to F-5d7410a76c21, F-9b718979506d, and conditionally F-4062894eb208. |

The context and candidate locals each have a four-byte declared representation. The saved pointer local has the represented pointer type. The supplied body does **not** establish that these locals form a contiguous aggregate, occupy a particular relative order in memory, or are reachable through one another’s addresses.

In particular, passing the context-local address does not establish permission or ability to access the other locals. Passing that address alongside the candidate address does not establish overlapping views.

The represented operations use four-byte values and object-table addressing, with two-byte destination writes. They do not establish a portable ABI, target-independent byte order, or general alignment rules.

Normal completion requires all executed accesses and calls to be valid. In particular:

- The acquired destination must permit two-byte writes at byte offsets 0 and 2 for each executed publication.
- Candidate storage must support the described reads and dependency interactions.
- On the conditional path, the third argument must support the object-table lookup and indirect call described below.
- The address returned by F-67a6bf796de6 must permit a four-byte assignment.
- The external expression must be valid to evaluate when reached.

No local checks enforce these prerequisites.

## Behavior

### Initialization and destination acquisition

Before the first dependency call, the function initializes the context local from the second argument, the saved pointer local from the third argument, and the candidate local from the fourth argument, in that order.

These are separate initializations, not an established initialization of a twelve-byte object.

The function calls F-603a53835f04 with the first argument followed by three zero-valued arguments. Its returned address is retained as the destination for both possible publications.

There is no explicit null check, validation, or later destination reacquisition. The same retained address is used even after intervening dependencies execute. Continued destination validity is a prerequisite rather than a guarantee supplied by this function.

### First candidate processing

The function calls F-5d7410a76c21 with:

1. The candidate-local address.
2. The second argument.
3. The value one.

No supplied contract establishes whether this dependency replaces the whole candidate, changes only part of it, or leaves it unchanged. The fourth argument’s initialization can therefore remain relevant, but it is not guaranteed to survive preceding calls unchanged.

The second argument is supplied directly. It is not obtained by rereading the context local.

### First publication

After F-5d7410a76c21 returns, the function performs these value operations in order:

1. Read the candidate’s low-order 16-bit value.
2. Write that value as two bytes at destination byte offset 0.
3. Read the candidate’s two-byte portion at candidate byte offset 2.
4. Write that portion as two bytes at destination byte offset 2.

The first destination write precedes the second candidate read. There is no established four-byte snapshot of the candidate before publication.

This distinction matters if the destination aliases candidate storage. For example, if the first destination write overlaps the candidate’s unread second portion, that write can change what the next read obtains. The second destination write then uses the later value.

The first operation is described numerically as selection of the low-order 16 bits; the second candidate operation selects the two-byte portion at byte offset 2. Their combination does not establish a target-independent byte serialization. The supplied expression structure also does not establish the exact machine-load instructions used to obtain these values.

The two destination stores are separate. No atomic four-byte replacement or synchronization guarantee is supplied.

### Post-publication processing and branch decision

After both stores, the function calls F-9b718979506d with the candidate address.

It then calls F-ec61551795ee with the retained destination address and examines the returned 32-bit integer:

- Zero skips all conditional processing and ends the invocation.
- Any nonzero value enters the conditional path. Negative nonzero values are included.

The query occurs after F-9b718979506d returns. Effects of that dependency can therefore affect the state presented to the query. Neither dependency is established to preserve candidate storage, destination contents, external data, or other accessible state.

The zero branch does not undo either destination store or any earlier dependency effects.

### Conditional object dispatch

On a nonzero query result, the function accesses the object designated by the third argument. It reads the object’s first represented word, uses that value as the base for a callable-entry lookup at byte offset 8, and invokes the selected entry with the third argument as its sole visible argument.

This dispatch uses the argument directly, not an explicit reload of the saved pointer local. No additional role for that local is established.

The table offset is eight bytes; it is not eight entries. The represented address calculation supplies no bounds check. Portable behavior for an invalid base, invalid entry, or arithmetic outside the valid address representation is not established.

The indirect call produces a four-byte result. Its application-level meaning and additional effects are unknown.

### Conditional assignment and candidate processing

The function calls F-67a6bf796de6 with:

1. The context-local address.
2. The indirect call’s four-byte result.

The returned address is used as the destination of a four-byte assignment.

Only after F-67a6bf796de6 returns does the function evaluate the external value used by that assignment. It then assigns the resulting value through the returned address.

The destination type establishes the four-byte store. It does **not** independently establish:

- The external symbol’s declared type.
- A four-byte external load.
- Whether evaluation requires one load or several accesses.
- The conversion, extension, truncation, or other interpretation needed to obtain the stored value.

Those details remain unresolved. The established ordering is that evaluation of the external value follows both the indirect call and F-67a6bf796de6.

There is no explicit validation of the returned address. It is not established to refer to fresh storage or to be disjoint from the destination, candidate, context local, object, or external data. The assignment can affect later behavior if relevant aliasing exists.

The function next calls F-4062894eb208 with:

1. The candidate-local address.
2. The second argument.
3. The third argument.
4. The context-local address.

The two local addresses are separately supplied. No relative address, containment, or overlap relationship is established. The function does not explicitly reinitialize either local before this call.

The second and third arguments remain direct argument values, not values explicitly reread from the initialized locals.

### Second publication and completion

After F-4062894eb208 returns, the function publishes to the originally retained destination using the same ordering as before:

1. Read the candidate’s low-order 16-bit value.
2. Write two bytes at destination offset 0.
3. Read the candidate’s two-byte portion at candidate offset 2.
4. Write two bytes at destination offset 2.

The same aliasing considerations apply. Capturing both candidate portions before either store would impose behavior not established by the supplied body.

After these stores, the function calls F-9b718979506d again with the candidate address, then ends.

There is no second query, loop, retry, or explicit rollback. A normally completing invocation performs one publication on the zero branch and two on the nonzero branch.

## Side effects

The function explicitly performs:

- Three local initializations.
- Two unconditional two-byte destination stores.
- A conditional four-byte assignment through a returned address.
- Two additional conditional two-byte stores to the retained destination.
- Dependency calls whose further effects are not supplied.

The first publication occurs before the branch decision. Conditional processing does not defer or replace that earlier event.

The explicit destination-write totals are four bytes across two stores on the zero branch, and eight bytes across four stores on the nonzero branch. These totals describe write operations, not distinct memory extents, final contents, or all writes performed by dependencies.

Aliasing remains relevant at several points. A destination write may alter the candidate before its next read. The conditional assignment may alter state later inspected by F-4062894eb208. Dependencies may mutate state relevant to subsequent operations. No preservation or nonaliasing guarantee excludes these possibilities.

The supplied body does not explicitly allocate, deallocate, transfer ownership, adjust a reference count, or restore prior contents. F-9b718979506d cannot be classified as a release operation merely because it follows publication.

Invocation-local addresses are exposed to dependencies. Retention of those addresses is unknown; validity for access after the invocation is not established. Exposure of one local’s address supplies no established access to neighboring locals.

There is no explicit error result or recovery path. Invalid accesses, invalid call targets, and failures within dependencies are not visibly handled. Later operations depend on earlier calls returning normally.

## Dependencies

| Opaque function ID | Established interaction | Unresolved properties |
|---|---|---|
| F-603a53835f04 | Receives the first argument and three zeros; returns the retained destination address. | Allocation, initialization, ownership, lifetime, aliasing, mutation, and failure behavior. |
| F-5d7410a76c21 | Receives the candidate address, second argument, and one. | Candidate semantics, write extent, preservation, retention, and other effects. |
| F-9b718979506d | Receives the candidate address after each publication. | Mutation, release behavior, ownership, retention, global effects, and failure behavior. |
| F-ec61551795ee | Receives the destination; its 32-bit zero/nonzero result selects the branch. | Result meaning and effects on destination or other state. |
| F-67a6bf796de6 | Receives the context-local address and indirect-call result; returns the address used for a four-byte assignment. | Returned extent, lifetime, ownership, alignment, aliasing, and access beyond the supplied local. |
| F-4062894eb208 | Receives the candidate address, second argument, third argument, and context-local address. | Candidate semantics, access extent, preservation, retention, and any relationship assumed between the local addresses. |

The runtime-selected indirect dependency has no supplied opaque function ID. Its selection mechanism, sole visible argument, and four-byte result are established; its semantic contract is not.

No dependency is assumed to preserve globals, preserve pointed-to data, acquire ownership, or release memory.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests.

### Zero query result

Assume valid accesses and normally returning dependencies, with F-ec61551795ee returning zero.

Acceptance requires the first publication’s low-value read, offset-0 store, second-portion read, and offset-2 store in that order. F-9b718979506d follows the stores; the query follows that call.

No object-table dispatch, F-67a6bf796de6 call, external-value assignment, F-4062894eb208 call, or second publication occurs. Earlier effects remain part of the invocation.

### Nonzero query result

Assume the query returns any nonzero 32-bit value and all conditional operations are valid.

The object-table entry at byte offset 8 is invoked with the third argument. Its result is supplied to F-67a6bf796de6 alongside the context-local address.

After that dependency returns, the external value is evaluated and assigned through the returned address with a four-byte destination width. F-4062894eb208 follows the assignment. The second publication follows that call, and a second F-9b718979506d call follows the publication.

An implementation that accepts only positive query results would not meet this branch behavior.

### Candidate initialization survives

Suppose the candidate still contains its initialized value when publication begins, and no effect between its two reads changes the relevant storage.

The publication writes the low-order 16-bit value and the portion at byte offset 2 from that initialized candidate, in order. Without an established byte order, this is not a portable four-byte serialization claim.

F-5d7410a76c21 leaving the candidate unchanged is insufficient by itself to establish this scenario: other preceding effects and interference during publication must also be excluded.

### First store changes the unread portion

Suppose destination offset 0 overlaps the candidate’s second two-byte portion, initially holding different bytes from those produced by the first store.

The first store changes that portion before the second candidate read. The second store uses the value subsequently read from the changed storage.

Capturing both portions before either store would fail this ordering requirement. The example applies to either publication.

### External value changes during a dependency

Suppose F-67a6bf796de6 changes the state determining the external value, and that state remains unchanged until the assignment evaluates it.

The assignment uses the external value evaluated after the call. Capturing the external value before the call would not preserve this ordering.

This example does not specify the source-access width or the conversion to the four-byte destination. Those depend on missing external type information.

### Separate local storage

Suppose the context and candidate locals are placed in separate valid storage regions with no adjacency, and dependencies require no undocumented layout relationship.

Nothing in the explicit body requires a shared twelve-byte region. The context address and candidate address must still be supplied at their specified call positions.

A requirement that one address expose both locals would add an unsupported layout contract.

### Observation between stores

If a valid observation mechanism can distinguish the destination between its two component stores, the contract does not promise a single atomic replacement. This conditional example does not establish a concurrency or synchronization model.

## Uncertainties

The following remain **documented_only**:

- Application-level meanings of the arguments, locals, destination, and external value.
- Semantic meaning of the query result.
- Relative placement, adjacency, padding, or containment of the three locals.
- Whether any dependency expects or accesses storage beyond the addressed local.
- The external symbol’s type, evaluation mechanics, access width, and assignment conversion.
- Target byte order, exact machine-load widths, and broader ABI details.
- Alignment requirements and behavior outside valid address arithmetic.
- Dependency mutation, preservation, retention, ownership, and failure contracts.
- Aliasing among exposed locals, destination, object, external data, and returned addresses.
- Concurrency, reentrancy, synchronization, and external observation guarantees.
- Behavior after invalid accesses or calls that do not return normally.

An initial candidate followed by a conditional alternative is a plausible interpretation of intent. It is not an acceptance requirement. The established behavior is the ordered local initialization, calls, value reads, writes, and branch decision.

## Evidence

The supplied static body establishes three separately declared locals and their initialization order. It establishes exposure of the candidate address and context-local address at particular calls. It does not establish a contiguous aggregate, relative local offsets, or guaranteed overlapping views.

The body establishes dependency arguments, retention of the acquired destination address, object-table selection at byte offset 8, the 32-bit zero/nonzero branch, and the absence of an explicit return value.

Both publications place the first destination store before the second candidate read. Under unresolved aliasing, this ordering cannot be replaced contractually by a pre-captured four-byte candidate.

The conditional assignment establishes a four-byte destination store and evaluation of the external value after F-67a6bf796de6 returns. The external declaration is absent, so its source-access width and conversion are not established.

No dependency implementations, additional layout evidence, external type declaration, or runtime observations were supplied. No runtime tests have been performed.
