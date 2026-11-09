<!-- Trusted provenance: F-ec9a8c39dd4b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ec9a8c39dd4b

## Purpose

F-ec9a8c39dd4b descends through a linked structure using a comparison dependency, then either returns a selected node or calls an insertion dependency. Its caller-visible output is a four-byte value followed by a one-byte status.

The established status meanings are:

- **0:** the selected cursor was returned without an insertion call on that path.
- **1:** an insertion dependency was called, and the contents of its supplied four-byte storage slot were copied to the result.

Status 1 does not independently establish successful insertion, allocation, or initialization of the entire returned value. The insertion dependency’s return value is not checked.

The structure and navigation suggest an ordered tree with a header and an optional uniqueness check. That is inferred intent, not a prerequisite for interpreting the accesses and branches described below.

## Interface

### Arguments and result

| Argument | Established use |
|---|---|
| Result address | Writable caller storage: four bytes at offset 0 and one byte at offset 4. |
| Container address | A 32-bit address used to access a header pointer, mode byte, and comparison-state address. |
| Query | An opaque four-byte value passed unchanged to dependencies. Its interpretation as an address is not established. |

There is no explicit return value.

The result requires at least five writable bytes. No bytes beyond offset 4 are explicitly initialized. The four-byte value is written before the status byte.

### Container and node layout

The supplied representation uses four-byte addresses and four-byte scalar slots.

| Container offset | Width | Use |
|---|---:|---|
| 0x10 | 4 bytes | Header address |
| 0x18 | 1 byte | Zero selects additional checking; nonzero selects common insertion |
| 0x19 | Address only | Comparison-state address passed to F-bcefa5957e76 |

This routine does not directly read comparison-state contents at offset 0x19.

| Node or header offset | Width | Use |
|---|---:|---|
| 0x00 | 1 byte | Marker used during cursor adjustment |
| 0x04 | 4 bytes | Root when accessed through the initial header; upward link during adjustment |
| 0x08 | 4 bytes | Link selected by a nonzero descent comparison; boundary seed when accessed through the header |
| 0x0C | 4 bytes | Link selected by a zero descent comparison; also used during adjustment |
| 0x10 | Address only | Address passed as the third argument of a descent comparison |

No complete node layout, alignment requirement, or target byte order is established.

Address calculations use the supplied 32-bit representation and fixed byte offsets. The contract does not extend these accesses to wider pointers or assume that overflow produces a valid address.

### Local storage relevant to dependencies

Two distinct four-byte scratch slots affect insertion results:

- **Immediate-insertion slot:** supplied to insertion only when the boundary dependency returns nonzero. Its low byte is later reused for status staging.
- **Common-insertion slot:** supplied to insertion on the common path. It also holds the header-derived boundary input.

Neither slot has an explicit initialization before traversal. Every executed traversal iteration writes zero to **both** slots before calling the comparison dependency. Their subsequent contents are path-dependent; neither is automatically cleared at insertion entry.

A separate four-byte cursor slot is initialized from the traversal candidate when zero mode is selected. Its address is exposed to the boundary dependency.

These slots are invocation-local storage. Passing their addresses does not establish ownership transfer or a valid lifetime after this invocation returns.

### Preconditions

There is no explicit argument validation.

Normal completion requires valid storage for every executed access, writable result storage, and normally returning dependencies. Traversal and adjustment links must permit termination.

There is no explicit non-aliasing guarantee. Result storage may overlap other reachable storage, but ordinary tree behavior is not promised under such overlap. The ordered accesses and writes remain the relevant contract.

## Behavior

### Initial state and descent

The saved comparison outcome initially has a nonzero value.

The routine first reads the header address from container offset 0x10 and reads the initial traversal node through that header’s offset 0x04. It separately rereads container offset 0x10 to establish the initial insertion candidate.

Thus, with an empty traversal, the candidate is the header obtained by the second read, not necessarily the header used for the root read. There is no null-header check before the root access.

For each nonnull traversal node:

1. Both insertion-related scratch slots are explicitly set to four-byte zero.
2. F-bcefa5957e76 receives the comparison-state address, original query, and current node address plus 0x10.
3. The current node becomes the insertion candidate.
4. The returned comparison outcome selects the next link:
   - Zero selects offset 0x0C.
   - Nonzero selects offset 0x08.

Only zero versus nonzero matters. Positive and negative nonzero comparison results have identical branch significance.

The next-link read occurs after the dependency returns. Dependency changes to accessible node storage can therefore affect traversal.

Descent ends on a null link. The saved candidate is the last visited node, or the separately read header candidate if no node was visited. The saved comparison outcome is the last descent result, or its initial nonzero value for an empty traversal.

For a nonempty traversal, the last iteration has explicitly zeroed both scratch slots. No address of either slot has yet been passed to a dependency by this invocation.

### Mode selection

After descent finishes, the routine reads the byte at container offset 0x18.

- **Nonzero:** proceed directly to common insertion.
- **Zero:** initialize the cursor from the saved candidate and perform the additional checking below.

The mode is not cached at entry. Changes made during descent can affect this decision.

### Boundary decision

In zero mode, a saved comparison outcome of zero skips the boundary dependency and cursor adjustment.

A saved nonzero outcome, including the initial outcome after an empty traversal, causes the routine to:

- Reread the container’s header address.
- Read four bytes at that header’s offset 0x08.
- Store that value into the **common-insertion slot**, replacing its previous contents.
- Call F-3b480a32387a with the cursor-slot address and common-insertion-slot address.

The header-derived value is a local copy. The second argument does not directly point to the header field. The dependency can modify both supplied local slots.

If the boundary result is nonzero, the routine immediately calls F-eadd6d46fa31 with the **immediate-insertion slot**, container address, literal zero, saved insertion candidate, and original query. It copies that slot to result offset 0 and writes status 1 at offset 4.

This path does not perform cursor adjustment or extraction. Changes to the cursor do not replace the saved insertion candidate.

If the boundary result is zero, adjustment uses the cursor value remaining after that call.

### Cursor adjustment

The routine reads the cursor’s marker byte at offset 0x00.

If the marker is zero, it follows offset 0x04 and then reads offset 0x04 through the resulting address. If that two-link result equals the current cursor, it replaces the cursor with the cursor’s offset-0x0C link and finishes adjustment.

A nonzero marker skips the two-link test. A zero marker whose two-link result differs from the cursor also enters ordinary adjustment.

Ordinary adjustment first reads the cursor’s offset-0x08 link and separately rereads that field for the null test.

If the null-test read is nonzero, the first link value becomes the cursor. The routine then follows offset-0x0C links until the null test on that field succeeds. In each iteration, the field is read once for the potential next cursor and again for the continuation condition.

Under stable link contents, this selects the terminal node reached by taking offset 0x08 once and then offset 0x0C repeatedly. The repeated reads are present in the supplied representation; no snapshot or synchronization guarantee is established.

If the offset-0x08 null test succeeds, the routine instead obtains an upward candidate through offset 0x04. At each upward step:

- It compares the upward candidate’s offset-0x08 link with the previous cursor.
- It assigns the upward candidate to the cursor regardless of whether they matched.
- A match continues through the upward candidate’s offset-0x04 link.
- A mismatch stops.

Consequently, the first upward candidate whose tested link differs from the previous cursor becomes the final cursor. Returning the previous cursor on that mismatch would differ from this behavior.

### Selected-node comparison

After any required adjustment, the routine calls F-1072a4a3aba1 with the selected cursor.

Its four-byte result becomes the second argument of F-bcefa5957e76. The comparison arguments are:

- Comparison-state address at container offset 0x19.
- Extracted four-byte value.
- Original query.

This ordering differs from descent, where the query is second and a node-relative address is third.

If this comparison returns zero, the routine writes the current cursor to result offset 0 and status 0 to offset 4. No insertion call occurs on this path. There is no additional reverse comparison or independent equality test.

A nonzero result selects common insertion.

### Common insertion and scratch-slot reuse

Common insertion calls F-eadd6d46fa31 with the **common-insertion slot**, container address, literal zero, saved insertion candidate, and original query.

After the call, the routine copies that slot’s four bytes to result offset 0 and writes status 1 at offset 4. Cursor adjustment does not change the candidate passed to insertion.

The insertion dependency receives existing storage contents, not an invariably fresh or uninitialized output:

| Path to insertion | Supplied slot | Explicitly established contents before relevant later calls |
|---|---|---|
| Nonempty traversal, nonzero mode | Common | Zero from the last traversal iteration |
| Nonempty traversal, zero mode, final descent outcome zero, selected comparison nonzero | Common | Zero from the last traversal iteration |
| Boundary returns zero, selected comparison nonzero | Common | Fresh header offset-0x08 value, subsequently writable by the boundary dependency |
| Nonempty traversal, boundary returns nonzero | Immediate | Zero from the last traversal iteration |
| Empty traversal, boundary returns nonzero | Immediate | No explicit initialization |
| Empty traversal, nonzero mode | Common | No explicit initialization |

Once the common slot’s address has been passed to the boundary dependency, subsequent dependencies may modify it through an escaped address. Its contents at insertion entry are therefore not necessarily the original header value or even the value present immediately after the boundary call.

Insertion does not reset either slot. If insertion leaves an initialized slot untouched, its existing value is copied. If it leaves a slot with no established contents untouched, the returned four-byte value is not established by this routine.

Partial writes likewise preserve untouched portions of the existing slot; their significance depends on prior initialization and the target representation.

## Side effects

### Direct writes

The routine directly writes invocation-local storage and the caller’s result. It contains no direct write to container or node fields.

On every normally completing path:

1. Four bytes are written at result offset 0.
2. One byte is written at result offset 4.

Status staging reuses the low byte of the immediate-insertion slot, but only after any value taken from that slot has been copied to the result. It does not replace a byte of the returned four-byte value.

If result storage aliases container or node storage, the four-byte result write can change that storage before the status write occurs.

### Dependency-mediated effects and ownership

No dependency is established as pure, allocation-free, globally preserving, or incapable of retaining an address.

Relevant exposure and ordering include:

- Descent comparisons precede the selected child-link reads.
- The mode and boundary header value are read after descent calls.
- The boundary dependency receives writable addresses for the cursor and common-insertion slot.
- Extraction and the selected comparison execute after that exposure.
- Insertion receives writable local storage plus the container, candidate, and query values.

The routine performs no explicit allocation, release, ownership transfer, reference-count operation, or rollback. Such behavior inside dependencies is documented_only.

### Errors and nontermination

There is no separate error result or checked insertion return code.

Invalid pointers or links can cause invalid accesses. Cycles can prevent termination. A dependency that faults, exits nonlocally, or does not return prevents subsequent operations from completing.

Failure to write an insertion output is **not uniformly an unestablished result**: untouched initialized storage retains its value. Only paths with unestablished storage, or unestablished subsequent modifications, lack a determined four-byte result.

## Dependencies

| Opaque ID | Contract established by this caller |
|---|---|
| F-bcefa5957e76 | Three arguments; outcome tested only for zero versus nonzero. Descent receives comparison state, query, and node-plus-0x10 address. Selected checking receives comparison state, extracted value, and query. |
| F-3b480a32387a | Called in zero mode when the saved descent outcome is nonzero. Receives addresses of the cursor and common-insertion slot. The latter is seeded with a freshly read header field. Nonzero selects immediate insertion; zero selects adjustment. |
| F-1072a4a3aba1 | Receives the selected cursor. Its four-byte result supplies the second argument of the selected comparison. Its interpretation and side effects are unknown. |
| F-eadd6d46fa31 | Receives a writable four-byte slot, container, zero, saved insertion candidate, and query. The slot has the path-dependent initial contents described above. Its post-call contents are copied without validation; status 1 follows. |

There is no direct recursive call.

## Acceptance examples

These are conditional static criteria, not executed tests. Unless specified otherwise, accesses are valid, links remain stable, and dependencies return normally without unrelated mutations.

### Nonempty traversal with untouched common output

Given a nonempty traversal and nonzero mode, if insertion does not modify its supplied slot, the result value is **0** and status is **1**. The final traversal iteration zeroed that slot.

The same result applies when zero mode follows a zero final descent outcome, the selected comparison is nonzero, and insertion leaves the common slot untouched.

### Empty traversal with nonzero mode

No descent comparison, boundary call, or extraction occurs. Common insertion receives the separately read header candidate.

If insertion supplies a complete four-byte value R, the result is R with status 1. If it leaves the slot untouched, no result value is established by explicit initialization in this routine.

### Boundary-triggered insertion

Given zero mode and a nonzero saved descent outcome, a nonzero boundary result selects immediate insertion with the original candidate, even if the boundary dependency changed the cursor.

If traversal was nonempty and insertion leaves its slot untouched, the result is 0 with status 1. If traversal was empty, that untouched slot has no explicit initialization.

### Header-seeded common output

Suppose the freshly read header field is H, the boundary result is zero, and the selected comparison is nonzero.

If neither the boundary dependency nor subsequent calls alter the common slot, and insertion leaves it untouched, the result is H with status 1.

If the boundary dependency changes the slot to M and no subsequent call alters it, untouched insertion returns M instead. This remains true even when H was originally zero.

### Mutation through an escaped address

Suppose the boundary dependency retains the common-slot address and a subsequent dependency changes its contents to V through that address.

If common insertion then leaves the slot untouched, the returned value is V with status 1, not necessarily the header seed or the boundary-call exit value.

### Existing-node result

Given zero mode and a zero final descent outcome, the boundary call and adjustment are skipped.

If extraction and the selected comparison complete with comparison result zero, the last visited node is returned with status 0 and no insertion call occurs.

### Adjustment boundaries

With a zero marker and a two-link offset-0x04 result equal to the cursor, the offset-0x0C replacement is used directly for extraction.

Otherwise, if ordinary adjustment enters the upward path and the first upward candidate’s offset-0x08 link differs from the previous cursor, that upward candidate becomes the selected cursor.

### Comparison and candidate distinctions

Any nonzero descent comparison follows offset 0x08; zero follows offset 0x0C.

If the selected comparison later requires insertion, insertion receives the saved traversal candidate, not the adjusted cursor.

## Uncertainties

### Inferred intent

The layout and adjustment pattern suggest ordered-tree insertion with optional duplicate suppression. Status 0 likely identifies an existing equivalent element, and status 1 likely identifies an insertion result. Only their control-flow meanings are established.

### documented_only

The supplied routine does not establish:

- Comparison ordering or equivalence rules.
- Comparison-state layout or complete node layout.
- Sentinel, marker, and upward-link invariants.
- Meaning of the literal zero insertion argument.
- Whether query or extracted values denote addresses.
- Allocation, ownership, object lifetime, or insertion success semantics.
- Dependency preservation of globals or reachable storage.
- Whether exposed local addresses escape or are later used.
- Synchronization, reentrancy, or concurrent-modification guarantees.
- Exception, allocation-failure, or fault-handling behavior.
- Target byte order or a larger ABI result layout.
- A fixed value for an untouched slot lacking explicit initialization.

These unknowns do not erase established slot initialization. Zeroed and header-seeded slots have defined prior contents unless subsequently changed; uninitialized slots do not acquire a value merely because insertion returns.

## Evidence

This contract derives solely from static inspection of the supplied routine representation.

Established evidence includes access widths and offsets, repeated header and link reads, zero/nonzero decisions, dependency argument order, candidate/cursor separation, per-iteration initialization of both scratch slots, boundary reuse of the common-insertion slot, distinct insertion output slots, and ordered result writes.

No dependency implementations were supplied. Their semantics are limited here to what this caller exposes and consumes.

No runtime tests have been performed. Acceptance examples express conditional consequences of static evidence, not observed execution.
