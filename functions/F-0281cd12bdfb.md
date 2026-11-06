<!-- Trusted provenance: F-0281cd12bdfb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0281cd12bdfb

## Purpose

F-0281cd12bdfb searches a linked collection for the first eligible entry accepted by an externally supplied predicate. If that entry is not already at the front, the function moves it to the front. If the traversal reaches a null entry without an accepted match, the function invokes a separate fallback operation.

The established behavior is a conditional search followed by either linked-list mutation or fallback dispatch. Interpreting the collection as a cache, the predicate as an equality comparison, or the fallback as an insertion operation would be inferred intent, not an established contract.

This specification describes static evidence only. No runtime tests have been performed.

## Interface

### Input and return

The function receives one address-valued argument identifying a caller-provided context object. The supplied representation uses a 32-bit integer carrier for this address. The depicted layout uses four-byte words and four-byte pointer fields.

No return value is established. The function returns after one of these outcomes:

- An accepted entry is already at the front, so this function performs no list writes.
- An accepted entry is moved to the front.
- Exhaustion causes the fallback operation to be invoked and that operation returns.

There is no explicit status result distinguishing these outcomes.

### Context layout

The following context locations are relevant:

| Offset | Width | Role |
|---|---:|---|
| `0x40` | 4 bytes | An opaque argument captured and passed to predicate and fallback operations |
| `0x44` | 4 bytes | Address of the collection container |
| `0x6C` | Address only | Beginning of a context-relative region whose address is passed to fallback |

The function does not directly read the contents at offset `0x6C`. It passes the address of that region, rather than a value loaded from it.

### Container layout

The container address obtained from the context has these relevant fields:

| Offset | Width | Role |
|---|---:|---|
| `0x10` | 4 bytes | First-entry pointer |
| `0x14` | 4 bytes | Last-entry pointer, updated when moving a non-front last entry |

A null first-entry pointer represents an exhausted traversal immediately. The function does not inspect the last-entry field merely to determine whether the collection is empty.

### Entry layout

Each visited entry has these relevant locations:

| Offset | Width | Role |
|---|---:|---|
| `0x00` | 4 bytes | Previous-entry pointer |
| `0x04` | 4 bytes | Next-entry pointer |
| `0x08` | Address only | Start of an entry-relative region passed to the predicate |
| `0xC8` | 4-byte read | Value narrowed to a character-sized value for eligibility testing |

The eligibility condition is equality to `1` after narrowing the loaded word to the depicted character width. It is not a general nonzero test.

For the depicted eight-bit character model, this tests whether the low eight bits of the loaded word equal `0x01`. It should not be restated as an unconditional one-byte read from address `entry + 0xC8`: the supplied representation shows a word load followed by narrowing.

### Dispatch object layout

A dependency supplies a dispatch-object address. Its first word identifies a dispatch table containing callable entries at byte offsets:

- `0x04`: predicate operation.
- `0x08`: fallback operation.

These are indirect targets, not independently identified functions. Their full implementations and contracts are unavailable.

### Preconditions

Normal completion requires valid addresses and callable targets for every location actually accessed. In particular:

- The context must permit the reads at offsets `0x44` and `0x40`.
- The captured container must permit reading its first-entry field.
- The supplied dispatch object and the selected dispatch-table entries must be valid.
- Every visited non-null entry must permit its eligibility-field read and any required link reads.
- A move-to-front operation requires writable link fields, a valid predecessor, and a valid current first entry.
- Fallback requires that its target accept the supplied context-relative address.

The function contains no explicit checks enforcing these conditions.

## Behavior

### Initialization and captured state

The function first invokes F-62a2ca3b7507 with no explicit arguments.

Only after that dependency returns does it read the context:

1. It captures the container address from offset `0x44`.
2. It captures the opaque four-byte argument from offset `0x40`.
3. It calls F-ab6062dd4d3b with the integer argument `1`.
4. It reads the first-entry pointer from offset `0x10` of the captured container.

This order matters. Changes made by F-62a2ca3b7507 can affect both captured context fields. Changes made by F-ab6062dd4d3b can affect the subsequently read first-entry pointer.

The container address and opaque argument are not refreshed from the context after F-ab6062dd4d3b returns. For example, if that dependency replaces the context’s container field, traversal still uses the previously captured container address.

The returned dispatch-object address is used for later indirect operations. No null-result check is shown.

### Eligibility and predicate dispatch

Traversal begins at the first-entry pointer read after dispatch-object acquisition.

For each non-null entry:

1. The function reads the word at offset `0xC8` and tests its narrowed character value against `1`.
2. If the value is not `1`, it skips predicate dispatch.
3. If the value is `1`, it invokes the predicate operation selected through dispatch-table offset `0x04`.
4. A zero predicate result rejects the entry.
5. Any nonzero predicate result accepts the entry and ends the search.

The supplied call representation shows these predicate arguments, in this order:

1. The dispatch-object address.
2. The address eight bytes into the current entry.
3. The previously captured opaque four-byte context argument.
4. A callable address read from dispatch-table offset `0x04`.
5. An additional value whose origin is not established.

The fourth argument is depicted as the predicate target itself. The fifth is not established as zero, a context field, a valid pointer, or an initialized value. Its significance and even the exact high-level call signature remain documented_only.

For either an ineligible entry or a predicate-rejected entry, traversal advances by reading the current entry’s next pointer at offset `0x04`.

When a predicate has run, this next-pointer read occurs after the predicate returns. Consequently, predicate changes to that link can affect subsequent traversal. The function does not establish a snapshot of the list before calling external code.

### Exhaustion and fallback

If the current-entry pointer is null, the function invokes the fallback operation through dispatch-table offset `0x08`.

The depicted fallback arguments are:

1. The dispatch-object address.
2. The captured container address.
3. The address at context offset `0x6C`.
4. The captured opaque four-byte argument.

The function then returns if fallback returns. It does not restart traversal or inspect a fallback result.

Fallback is reached both for an initially empty collection and after traversal rejects or skips every reachable entry. These cases are not distinguished by a separate flag.

The function performs no direct list mutation before fallback on an ordinary exhausted traversal. This does not imply that the initialization dependencies or predicates left the list unchanged.

### Accepted entry already at the front

After predicate acceptance, the function reads the accepted entry’s previous-entry pointer.

If that pointer is null, it returns without direct list writes and without fallback dispatch.

The test is specifically whether the accepted entry’s previous pointer is null. The function does not independently verify that the container’s first-entry field points to this entry.

For a valid doubly linked list, a null previous pointer identifies the front entry. On malformed or externally modified state, that interpretation is not guaranteed.

### Moving an accepted non-front entry

If the accepted entry’s previous pointer is non-null, the function detaches the entry and places it before the container’s current first entry.

The direct writes occur in this order:

1. The predecessor’s next field is set to the accepted entry’s next pointer.
2. If the accepted entry’s next pointer is null, the container’s last-entry field is set to the predecessor.
3. Otherwise, the successor’s previous field is set to the predecessor.
4. The accepted entry’s next field is set to the container’s first-entry pointer as read during this operation.
5. The accepted entry’s previous field is set to null.
6. The previous field of the entry currently referenced by the container’s first-entry field is set to the accepted entry.
7. The container’s first-entry field is set to the accepted entry.

The supplied representation rereads link fields during this sequence rather than establishing one complete immutable snapshot. For a well-formed, non-aliased list without intervening mutation, these reads produce the usual move-to-front result. Behavior for malformed aliasing arrangements is not generalized beyond the depicted access sequence.

There are no external calls within this mutation sequence. There is also no visible locking, atomic publication, rollback, or protection against concurrent observers seeing intermediate links.

After the final first-entry update, the function returns. It does not continue searching for additional matches.

### Termination and errors

There is no traversal bound, cycle detection, explicit exception translation, error return, or recovery path.

Consequences include:

- A cycle containing no accepted entry can cause unbounded traversal.
- An invalid entry, container, dispatch object, or target can prevent normal completion.
- Failure or non-return of a dependency prevents reaching later operations.
- Invalid state during relinking can leave a partially modified structure.
- No established mechanism restores earlier writes after a failure.

## Side effects

The function’s own visible state changes are confined to the move-to-front path:

- One predecessor next pointer.
- Either one successor previous pointer or the container’s last-entry pointer.
- The accepted entry’s next and previous pointers.
- The former first entry’s previous pointer.
- The container’s first-entry pointer.

It does not directly write the context fields at offsets `0x40` or `0x44`, the region at `0x6C`, or the eligibility word at entry offset `0xC8`.

External calls may have additional effects. Nothing in the supplied evidence establishes that any dependency or indirect operation preserves globals, container contents, entry contents, or context memory.

No direct allocation, deallocation, reference-count adjustment, ownership transfer, or payload copy is shown. These absences apply only to this function’s direct operations. They do not establish what callees allocate, retain, modify, or release.

All later uses require the relevant addresses to remain usable. In particular, if a predicate releases its current entry, the function can subsequently read released storage. No lifetime protection is visible.

## Dependencies

| Dependency | Established use | Not established |
|---|---|---|
| F-62a2ca3b7507 | Called once, before context reads, with no explicit arguments | Purpose, global effects, lifetime effects, error behavior |
| F-ab6062dd4d3b | Called once with argument `1`; its result supplies the dispatch object | Ownership, nullability guarantees, allocation behavior, side effects |
| Indirect target at dispatch-table offset `0x04` | Called for eligible entries; zero rejects and nonzero accepts | Comparison semantics, full signature, preservation guarantees, meaning of the extra argument |
| Indirect target at dispatch-table offset `0x08` | Called once on exhaustion with four depicted arguments | Whether it inserts, allocates, reports an error, or changes ownership |

The dispatch table is consulted for indirect calls. The evidence does not justify assuming that its contents remain constant across earlier external calls.

## Acceptance examples

These are conditional behavioral expectations derived from static evidence, not executed tests. Unless stated otherwise, they assume a valid 32-bit layout, valid indirect targets, a well-formed list, and no dependency mutation of relevant state.

### Empty collection

Given a null first-entry pointer after F-ab6062dd4d3b returns:

- No eligibility field is read.
- No predicate is called.
- Fallback receives the captured container, the address at context offset `0x6C`, and the captured opaque argument.
- The function performs no direct list writes.

### Ineligible entry followed by an accepted entry

Given entries A then B:

- A’s narrowed eligibility value is not `1`.
- B’s narrowed eligibility value is `1`.
- B’s predicate result is nonzero.

Then A receives no predicate call. B is accepted and moved ahead of A. Fallback is not called.

### Nonzero eligibility is insufficient

Given an entry whose eligibility word narrows to `2`, that entry is skipped without predicate dispatch.

Under the depicted eight-bit narrowing model, a word value of `0x00000101` instead narrows to `1` and qualifies for predicate dispatch. This demonstrates that eligibility is neither whole-word equality to `1` nor whole-word nonzero testing.

### First entry accepted

Given a valid first entry with a null previous pointer and an accepting predicate:

- Traversal stops at that entry.
- No direct list writes occur.
- Fallback is not called.

### Interior entry accepted

Given the list A → B → C, with B accepted after A is skipped or rejected:

- The final forward order is B → A → C.
- B has no predecessor.
- A’s predecessor is B.
- C’s predecessor is A.
- The last-entry field remains C.

### Last entry accepted

Given A → B with B accepted:

- A’s next pointer becomes null.
- The last-entry field becomes A.
- B becomes first, with no predecessor and A as its successor.
- Fallback is not called.

### All entries rejected

Given a finite list in which every entry is ineligible or returns zero from its predicate:

- Traversal reaches null.
- Fallback is called once.
- No move-to-front writes are performed by this function.

### Predicate changes the next pointer

Given an eligible entry whose predicate returns zero but changes that entry’s next pointer, the subsequent traversal step uses the next pointer read after the predicate returns. The function does not necessarily visit the entry that was next before predicate dispatch.

### Dependency replaces the context container

If F-ab6062dd4d3b changes the context’s container field, traversal still reads the head from the previously captured container. A change to that captured container’s own first-entry field before the read can affect traversal.

## Uncertainties

- **documented_only — Predicate ABI:** The additional predicate argument has no established origin or meaning. The exact callable signature is unresolved.
- **documented_only — Fallback semantics:** Its invocation is established; insertion, allocation, copying, ownership transfer, and success reporting are not.
- **documented_only — Dependency preservation:** No dependency is known to preserve globals, memory, links, or object lifetimes.
- **documented_only — Platform details:** The representation depicts four-byte fields and 32-bit address arithmetic. Endianness, alignment requirements, and overflow behavior are not independently established.
- **documented_only — Address arithmetic limits:** Offsets are applied to supplied addresses without explicit bounds checks. Behavior at overflow or outside valid objects is not specified as portable wrapping behavior.
- **documented_only — Concurrency:** No synchronization is visible. Safe concurrent traversal or mutation is not established.
- **Inferred intent:** Moving the first accepted entry to the front suggests preferential ordering of recently selected entries. No particular caching or scheduling policy is proven.

## Evidence

The available evidence is a single static function representation and the provided opaque dependency mapping.

It establishes call order, field offsets, depicted word widths, eligibility gating, zero-versus-nonzero predicate handling, traversal progression, direct relinking writes, and fallback arguments.

It does not include dependency implementations, authoritative type declarations, an independently verified calling convention, execution traces, or runtime observations. The acceptance examples are therefore static consequences under their stated preconditions, not reports of successful execution.
