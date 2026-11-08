<!-- Trusted provenance: F-c045709aaabf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c045709aaabf

## Purpose

F-c045709aaabf conditionally invokes an action using a receiver-associated word and a selected 32-bit argument. Its decision depends on a descriptor control word, receiver-dispatched operations, and opaque dependency results.

The established control flow has three possible action sources:

- An accepted initial candidate: use a fresh read of the descriptor control word.
- A nonzero fallback query: use `0xffffffff`.
- An accepted enumerated candidate: use that candidate’s zero-based index.

If enumeration finds no accepted candidate, the function returns without invoking the action. The function itself explicitly invokes the action at most once per normal execution.

“Candidate,” “comparison,” and “action” identify roles in this contract, not established application semantics. In particular, the source does not establish that any dependency writes, initializes, constructs, copies, or releases an object merely because an address is passed to it.

## Interface

### Inputs

| Input | Established representation and accesses |
|---|---|
| Receiver | A pointer to 32-bit words. The first word is used as a dispatch-table address. The word at byte offset `0x40` is read when preparing an action. |
| Descriptor | A 32-bit address-like input. Its address plus `4` is passed to a dependency. Its word at byte offset `0x0c` is read for the initial decision and, conditionally, again for the action argument. |

The descriptor control word is initially interpreted as a signed 32-bit integer. If reread after initial-candidate acceptance, its bits are retained as an unsigned 32-bit argument.

The extent and type of the region beginning at descriptor offset `+4` are unknown. Its separation from the control word at `+0x0c` does not establish non-overlap: a dependency receiving the former address might access the latter field.

### Receiver dispatch

The receiver’s first word supplies the dispatch-table base for each reached dispatched call.

| Dispatch-table byte offset | Established invocation |
|---|---|
| `0x11c` | Receives the receiver. Its return value is retained as an unsigned 32-bit enumeration bound. |
| `0x150` | Receives a local storage address and the receiver. During enumeration, it also explicitly receives the current unsigned 32-bit index. |

The positive-control invocation at `0x150` has no explicit index argument in the supplied representation. Its signature and selection rule remain unresolved. The descriptor control value must not be treated as an argument to that invocation.

The local address passed to `0x150` serves as the apparent candidate-result location in subsequent operations. Whether the dispatched operation writes that region, and whether it fully initializes any subsequently accessed bytes, is unknown.

### Output and preconditions

There is no explicit return value.

Observable behavior includes reached memory reads, the explicit local-word assignment, dependency and dispatch calls, their argument values and order, and any effects of those calls.

Normal execution requires valid reached addresses and callable dispatch entries. The function contains no explicit null-pointer checks, backing-storage checks, or invalid-dispatch handling. The supplied representation uses 32-bit addresses and words; a different data model is not established.

## Behavior

### Entry and initial decision

The first operation passes the address of a four-byte local region to F-b02b795a8319. The source establishes the call, not that this dependency initializes or writes the region.

After that call returns, the function reads the descriptor control word at `+0x0c` as signed 32-bit data:

- A value greater than zero enters the initial-candidate path.
- Zero or a negative value enters fallback directly.

For example, `0x80000000` and `0xffffffff` take the nonpositive path under this signed interpretation.

The read occurs after F-b02b795a8319. No assumption that this dependency preserves the descriptor, receiver, or shared state is justified.

### Common candidate-processing sequence

Both the initial-candidate path and each enumeration iteration perform the following sequence after invoking dispatch `0x150`:

1. Pass an eight-byte local-region address and the address four bytes beyond the candidate-result base to F-46f0b3cb316e.
2. Read the 32-bit word twelve bytes beyond the candidate-result base and assign it to the local word immediately following that eight-byte local region.
3. Pass the candidate-result base to F-ee5919ca4be6.
4. Pass a separate eight-byte local-region address and the local-region address used as the first argument in step 1 to F-b9acea4b06ad.
5. Pass the first address from step 4 to F-44362605ab79 and retain its four-byte return value.
6. Pass another eight-byte local-region address and descriptor address plus four to F-b9acea4b06ad.
7. Pass the first address from step 6 and the retained four-byte value to F-4a902fecdea3.
8. Retain the comparison outcome, then call F-289ec4311c64 on the descriptor-associated local region.
9. Call F-289ec4311c64 on the candidate-associated local region.
10. Act on the retained comparison outcome.

The address roles explain subsequent use; they do not prove dependency output writes. In particular, neither invocation of F-b9acea4b06ad establishes that its first argument is written or fully initialized. The same limitation applies to F-46f0b3cb316e and dispatch `0x150`.

The neighboring 32-bit assignment in step 2 is different: it is an explicit read and write by F-c045709aaabf itself. The value copied is whatever the source word contains at that point; the source does not establish which earlier operation supplied it.

### Comparison result and arithmetic

The return from F-4a902fecdea3 is treated as unsigned 32-bit data. Acceptance is exactly equivalent to a zero result.

The represented calculation subtracts that result from one using 32-bit unsigned arithmetic and overrides the acceptance flag to zero when the result is unsigned-greater-than one. Its relevant truth table is:

| Comparison result | Outcome |
|---|---|
| `0` | Accept |
| `1` | Reject |
| `2` through `0xffffffff` | Reject |

Thus a negative-looking bit pattern such as `0xffffffff` is rejected. Unsigned subtraction does not create an additional accepted case because every result greater than one is explicitly rejected.

Both F-289ec4311c64 calls occur before acceptance or rejection changes the outer control flow. Their effects are not assumed to preserve receiver or descriptor state.

### Initial-candidate path

For a positive initial control word, dispatch `0x150` receives the candidate-result base and receiver, without an explicit index. The common candidate-processing sequence follows.

If the comparison rejects the candidate, execution enters fallback.

If it accepts:

1. The descriptor control word at `+0x0c` is read again, retaining all 32 bits.
2. The receiver word at `+0x40` is read.
3. F-ee1cb06d60a6 receives the receiver word followed by the reread descriptor word.
4. Execution proceeds to common completion.

The action argument is not the saved initial signed value. Intervening dependencies may have changed the descriptor field, so the second read need not be positive or equal to the first.

The order of the two action-related reads is established: descriptor first, receiver second.

### Fallback query

Fallback is reached directly for a nonpositive initial control word, or after rejection of the initial candidate.

The function:

1. Passes an eight-byte local-region address and descriptor address plus four to F-b9acea4b06ad.
2. Passes that local-region address to F-6d70e45e29c9 and saves its signed integer return value.
3. Passes the same local-region address to F-289ec4311c64.
4. Branches on the saved query result.

This sequence does not prove that F-b9acea4b06ad wrote or initialized the queried region.

If the saved result is nonzero, the function reads the receiver word at `+0x40` and calls F-ee1cb06d60a6 with that word and `0xffffffff`. It then proceeds to common completion.

The all-ones argument has no established application-level meaning. It is not proven to mean “clear,” “none,” or an error.

If the saved result is zero, enumeration begins.

### Enumeration

Dispatch `0x11c` is called once with the receiver. Its return value is saved as an unsigned 32-bit bound.

Indices begin at zero and increase by one. Each index strictly below the saved bound is processed in ascending order:

- Dispatch `0x150` receives the candidate-result base, receiver, and current index.
- The common candidate-processing sequence runs.
- A nonzero comparison result continues enumeration.
- A zero comparison result stops enumeration and selects the current index.

On acceptance, the receiver word at `+0x40` is read and supplied to F-ee1cb06d60a6 together with the accepted index. No later indices are fetched.

The bound is not queried again. Changes to receiver state during iteration do not cause an explicit bound refresh.

If the saved bound is zero, there are no indexed candidate calls. If every visited candidate is rejected, there is no action call.

For a bound of `0xffffffff`, the last possible visited index is `0xfffffffe`. After rejection there, incrementing reaches `0xffffffff`, which fails the strict bound comparison. The loop does not require an increment from `0xffffffff` to zero.

### Common completion

Every explicit normal-return path calls F-ee5919ca4be6 with the address of the four-byte local region initially passed to F-b02b795a8319.

If F-ee1cb06d60a6 is invoked, it precedes this final call. No completion guarantee is established after a dependency faults, fails to return, or transfers control nonlocally.

## Side effects

### Explicit reads and writes

The function explicitly reads:

- The descriptor control word after the entry dependency call.
- That control word again only after initial-candidate acceptance.
- The receiver’s dispatch-table pointer when preparing reached dispatched calls.
- Reached dispatch entries.
- The receiver word at `+0x40` only when preparing an action.
- The neighboring candidate-result word during each candidate-processing sequence.

It explicitly copies the last of these words into neighboring local storage. No explicit assignment to receiver or descriptor fields appears.

The absence of explicit receiver or descriptor writes is not a preservation guarantee. Unknown dependencies and dispatched operations may mutate those objects, reachable storage, or shared state.

### Local layout and access limits

The candidate-result base is followed in the represented local layout by:

- A four-byte region at the base.
- An eight-byte region beginning four bytes later.
- A four-byte word beginning twelve bytes later.

The eight-byte middle-region address is passed to F-46f0b3cb316e. The trailing word is explicitly copied.

The apparent transformed local region consists of an eight-byte region followed immediately by the copied four-byte word. Other local regions passed to F-b9acea4b06ad are represented as eight bytes; the entry region is represented as four bytes.

These sizes describe represented local storage, not proven callee access extents. They establish neither aggregate types nor complete initialization. The source does not establish whether dependencies read, write, partially write, or leave unchanged the supplied regions.

### Ownership and lifetime

Local addresses are passed to dependencies. F-c045709aaabf itself does not retain them after return, but callee retention is unknown.

The call pattern is consistent with setup, extraction, and cleanup roles, but those are inferred roles only. In particular:

- F-ee5919ca4be6 receives each candidate-result base before comparison dependencies run.
- F-289ec4311c64 receives both comparison-associated local regions before the decision takes effect.
- No separate later call on the apparent transformed local region is visible.
- F-ee5919ca4be6 receives the entry-local address at normal completion.

None of these calls proves destruction, deallocation, reference-count changes, ownership transfer, or successful initialization beforehand.

### Errors

There is no explicit error return, diagnostic, retry, or exception handler.

Every nonzero comparison result is treated as rejection. Every nonzero fallback-query result selects the all-ones action argument. Whether either dependency encodes errors within those results is unknown.

## Dependencies

| Opaque ID or dispatch location | Established use |
|---|---|
| F-b02b795a8319 | Receives the four-byte entry-local address before descriptor inspection. Whether it writes that region is unknown. |
| F-ee5919ca4be6 | Receives each candidate-result base after extraction and the entry-local address at normal completion. |
| F-46f0b3cb316e | Receives two local addresses before the explicit neighboring-word copy. Output effects are unknown. |
| F-b9acea4b06ad | Receives a local address and either an apparent transformed-local address or descriptor address plus four. Writes and initialization are not established. |
| F-44362605ab79 | Receives a candidate-associated local address and returns a four-byte value. |
| F-4a902fecdea3 | Receives a descriptor-associated local address and the four-byte candidate-associated value. Only a zero return accepts. |
| F-289ec4311c64 | Receives local addresses after query or comparison use; ownership effects are unknown. |
| F-6d70e45e29c9 | Returns the fallback branch result after receiving a local address. |
| F-ee1cb06d60a6 | Receives the receiver word at `+0x40` and the selected 32-bit argument. |
| Dispatch `0x11c` | Returns the saved unsigned enumeration bound. |
| Dispatch `0x150` | Receives apparent candidate-result storage, the receiver, and an explicit index only during enumeration. Writes are unknown. |

No dependency is assumed to preserve globals or to own, initialize, or release supplied memory.

## Acceptance examples

These are conditional criteria, not executed tests. They assume normal returns and valid reached accesses. Where later behavior depends on local contents, usable contents are a condition of the example, not a guaranteed dependency output.

### Accepted initial candidate with a changed control word

Given an initial signed control value of `7` and an initial comparison result of zero:

- Fallback and dispatch `0x11c` are skipped.
- Both comparison-associated F-289ec4311c64 calls precede the action.
- If the later descriptor read returns `0xffffffff`, the action receives `0xffffffff`, not `7`.
- The receiver word is read after that descriptor reread.
- The entry-local completion call follows the action.

### Rejected initial candidate

Given a positive initial control value and comparison result `1`, `2`, or `0xffffffff`, execution enters fallback after both comparison-associated completion calls. None of these results directly selects the initial control value for action.

### Nonpositive control and nonzero fallback result

Given an initial control word of zero or `0x80000000`, and a nonzero fallback-query result:

- No initial candidate invocation occurs.
- No enumeration bound is requested.
- The fallback-local F-289ec4311c64 call precedes the action.
- The action receives `0xffffffff`.

### First enumerated acceptance

Given a zero fallback-query result, bound `4`, and comparison results `3` and `0` at indices zero and one:

- Only indices zero and one are fetched.
- Index zero is rejected.
- The action receives index one.
- Indices two and three are not fetched.

### Empty or exhausted enumeration

With a zero fallback-query result and bound zero, there are no indexed candidate calls and no action.

With bound two and nonzero comparisons at both visited indices, both candidate-processing sequences complete and no action occurs.

Both cases still reach the entry-local completion call.

### Unknown output writes

If an address-taking dependency returns without writing its apparent output region, the source provides no replacement initialization. Subsequent operations use the storage through the established calls and explicit copy. The resulting values and validity cannot be specified from this function alone; they remain **documented_only**.

## Uncertainties

**Inferred intent:** The structure is consistent with trying an initial candidate, consulting a fallback condition, and then searching an ordered collection.

The following remain **documented_only**:

- Semantic types and application meanings of all input and local regions.
- Whether any address-taking dependency writes or fully initializes supplied storage.
- The origin and validity of subsequently consumed local contents.
- Exact dependency access extents and aliasing.
- The initial `0x150` signature and candidate-selection rule.
- Whether the enumeration bound semantically represents a count.
- Query and comparison meanings beyond the observed branches.
- The action’s interpretation of indices and `0xffffffff`.
- Allocation, release, ownership, and retained-reference behavior.
- Effects of reentrancy, callbacks, concurrent mutation, and nonlocal control transfer.
- Behavior following invalid accesses or dependency failures.

## Evidence

Established claims derive solely from the supplied static representation: explicit accesses, local layout, argument expressions, return-value uses, call ordering, arithmetic widths, branches, and loop conditions.

Passing a local address is evidence of address exposure, not proof of an output write or initialization. The neighboring 32-bit assignment is explicit and therefore independently established.

No dependency implementations or external interface contracts were supplied. No runtime tests have been performed. The acceptance examples are conditional review criteria, not execution evidence.
