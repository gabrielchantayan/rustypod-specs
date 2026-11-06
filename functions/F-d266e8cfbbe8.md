<!-- Trusted provenance: F-d266e8cfbbe8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d266e8cfbbe8

## Purpose

F-d266e8cfbbe8 obtains an object through a dependency, queries that object, and optionally publishes two one-byte results:

- A value obtained from an initial indirect call.
- A classification selected by ordered, short-circuiting queries.

When acquisition returns a non-null object and all required indirect calls return normally, the function returns the 32-bit value zero. Either output destination may be null without preventing the object queries.

If acquisition returns null, the function calls a separate dependency. The supplied evidence does not establish a usable return value or reliable continuation behavior for that path.

The meanings of the queried value and classification are not established. The classification must be understood through its selection rules, not through inferred domain names.

## Interface

### Inputs

The interface has three arguments, in this order:

| Argument | Established representation | Use |
|---|---|---|
| First argument | 32-bit value | Not explicitly read or forwarded by the visible function body. |
| First output destination | Nullable pointer to a one-byte writable location | Receives the value from the initial indirect query when non-null. |
| Second output destination | Nullable pointer to a one-byte writable location | Receives the selected classification when non-null. |

No buffer lengths, object handles, or ownership indicators are supplied by the caller.

For the visible body, changing the first argument does not change an explicit branch, dependency argument, or output expression. This is not a guarantee about behavior outside the visible body, including dependencies with access to ambient state.

### Return value

On the normal non-null acquisition path, the function returns exactly zero as a 32-bit result.

There is no established nonzero error code. In particular, the null acquisition path must not be specified as returning zero, returning an error value, or returning at all.

### Output layout

Each explicit output store affects one byte at the supplied address. There is no visible multi-byte output structure, padding, length field, or byte-order conversion.

A non-null output destination must be writable when its store occurs. The function does not visibly validate accessibility or alignment. Its one-byte stores do not establish any stronger alignment requirement.

The two output destinations are not visibly required to be distinct. If they identify the same byte and execution reaches both stores, the classification store occurs last and determines the final byte value, absent intervening external modification.

### Object layout used by indirect calls

The acquired object is accessed through an indirect-dispatch layout:

1. The beginning of the object supplies a dispatch-table address.
2. A callable target is loaded from that table at a specified byte offset.
3. The target is invoked with the acquired object pointer as its explicit argument.

The visible representation uses 32-bit pointer-bearing accesses. The table offsets below are byte offsets, not method ordinals.

A non-null acquisition result is therefore insufficient by itself for defined execution: the object, its dispatch-table pointer, and every selected callable entry must be valid when accessed. No validity check beyond the initial null comparison is visible.

## Behavior

### Acquisition

The function first calls F-b7af80e829ec with no explicit arguments.

If that dependency returns null, the function takes the failure path described below. Otherwise, it retains the returned object pointer for all subsequent indirect calls.

The object is obtained internally; neither output pointer is used as an acquisition input.

### Initial value query

Before evaluating classification queries, the function invokes the acquired object's dispatch entry at byte offset `0x13C`.

The result is captured as a one-byte value. Only that captured byte is subsequently published to the first output destination. The evidence does not establish whether the underlying method's declared result is itself one byte or whether a wider machine result is narrowed at this boundary.

The initial query occurs even when both output destinations are null.

Its captured value is retained across the later classification queries. The function does not visibly repeat this query immediately before writing the first output.

### Ordered classification

Classification uses query results represented as 32-bit integers. Each result is tested only for equality or inequality with zero. No magnitude comparison, arithmetic combination, or bit-mask interpretation is visible.

The query order and selection rules are:

| Priority | Dispatch-table byte offset | Action on nonzero result | Action on zero result |
|---|---:|---|---|
| 1 | `0x120` | Select classification `0`; stop querying | Continue to priority 2 |
| 2 | `0x114` | Select classification `3`; stop querying | Continue to priority 3 |
| 3 | `0x118` | Select classification `4`; stop querying | Continue to priority 4 |
| 4 | `0x10C` | Select classification `1`; stop querying | Continue to priority 5 |
| 5 | `0x110` | Select classification `2` | Select classification `0` |

Thus the classification belongs to the set `{0, 1, 2, 3, 4}` on normal completion.

Classification zero has two distinct selection conditions:

- The first classification query returns nonzero.
- All five classification queries return zero.

These conditions are observably different if indirect calls have side effects: the first condition executes only one classification query, whereas the second executes all five.

Every nonzero 32-bit query result satisfies the relevant condition. A negative signed result, if supplied by a dependency, is not treated as a separate error by this function.

### Dispatch reads and mutable state

For each indirect invocation, the visible operation obtains the dispatch-table address from the retained object and then obtains the target at the applicable offset.

The contract must not replace these repeated dispatch accesses with an assumed immutable table snapshot. An earlier dependency may change the object or its table pointer. A later invocation uses the dispatch state available at its own access.

The same acquired object pointer is passed to every selected indirect target. There is no visible reacquisition between queries.

This yields a maximum of six indirect calls after successful acquisition: the initial value query and five classification queries. A normal non-null path always makes at least two indirect calls: the initial value query and the first classification query.

### Publication and return

Only after classification is complete does the function publish its explicit outputs:

1. If the first output destination is non-null, store the captured initial-query byte there.
2. If the second output destination is non-null, store the selected classification byte there.
3. Return 32-bit zero.

Null output destinations suppress only their respective stores. They do not suppress acquisition, the initial query, or classification.

All selected indirect calls therefore precede both explicit output writes. If a selected call does not return normally, neither explicit output write has yet occurred. This statement does not exclude writes performed by dependencies themselves.

### Null acquisition path

When F-b7af80e829ec returns null, the function calls F-a817d8d58c10 with no explicit arguments.

On this path, the visible body performs:

- No indirect query through an acquired object.
- No explicit store to either output destination.
- No established normal return with a specified result.

The supplied material includes a non-return warning, but it does not provide enough reliable behavioral evidence to define whether the failure dependency terminates, throws, transfers control elsewhere, or returns unexpectedly. If it returns, subsequent behavior and the return value remain undocumented.

## Side effects

### Explicit memory effects

The function's visible writes are limited to the two optional one-byte output stores on the normal non-null path.

The first store precedes the second. They are not established as atomic as a pair, transactional, or synchronized with other threads.

The visible body does not initialize outputs on entry, clear them on acquisition failure, or provide a separate validity marker.

If both output pointers address the same byte, the first store is still part of the operation sequence; it is not omitted merely because the second store will overwrite it.

### Dependency effects

F-b7af80e829ec, F-a817d8d58c10, and all indirect targets may have effects not visible in this function. No preservation guarantee for global state, object fields, dispatch tables, or caller-accessible memory can be inferred.

Consequently:

- A dependency may alter memory before either explicit output store.
- The initial query and classification queries need not observe an immutable object snapshot.
- A dependency may invalidate the acquired object or an output destination.
- Whether such behavior is permitted by the larger system is unknown.

These are limitations on what can be promised, not claims that the dependencies actually perform those actions.

### Ownership and lifetime

There is no visible allocation, explicit release, or ownership transfer performed by this function.

The acquisition dependency's ownership contract is unknown. The absence of a visible release does not establish that the object is borrowed, permanent, reference-counted, or leaked.

Likewise, the function neither visibly retains nor releases the output destinations. Caller-provided storage must remain usable until the relevant store occurs, subject to any stronger requirements imposed by dependency behavior.

### Failure and invalid access

Beyond checking the acquired pointer and each output pointer for null, the function performs no visible validation.

There is no established recovery for an invalid object layout, missing indirect target, invalid output address, or abnormal dependency termination. Such cases must not be converted into invented status codes or no-op behavior.

## Dependencies

| Dependency | Role | Established interaction | Unestablished properties |
|---|---|---|---|
| F-b7af80e829ec | Acquire the queried object | Called once, first, with no explicit arguments; result is tested for null | Ownership, lifetime, global effects, failure mechanism, thread safety |
| F-a817d8d58c10 | Handle null acquisition | Called with no explicit arguments only after a null acquisition result | Whether it returns; termination or transfer mechanism; memory and global effects |
| Unresolved indirect dependency at `0x13C` | Obtain initial output byte | Called once on the normal non-null path, before classification, with the acquired object | Semantic meaning, full result signature, side effects |
| Unresolved indirect dependency at `0x120` | Highest-priority classification condition | Always called after the initial query on a continuing non-null path | Semantic meaning and side effects |
| Unresolved indirect dependency at `0x114` | Select classification `3` | Called only if `0x120` returned zero | Semantic meaning and side effects |
| Unresolved indirect dependency at `0x118` | Select classification `4` | Called only if the preceding two conditions returned zero | Semantic meaning and side effects |
| Unresolved indirect dependency at `0x10C` | Select classification `1` | Called only if the preceding three conditions returned zero | Semantic meaning and side effects |
| Unresolved indirect dependency at `0x110` | Select classification `2` or final zero | Called only if the preceding four conditions returned zero | Semantic meaning and side effects |

The offsets identify dispatch positions, not stable identities of their runtime targets. No opaque function IDs for those indirect targets are supplied.

## Acceptance examples

These examples are conditional static acceptance criteria, not executed tests. Except where stated otherwise, assume acquisition returns a valid object, selected dependencies return normally, dispatch accesses remain valid, output storage is writable, and dependencies do not independently modify that storage.

### Classification cases

| Query outcomes in execution order | Required classification | Required skipped classification queries |
|---|---:|---|
| `0x120` returns nonzero | `0` | `0x114`, `0x118`, `0x10C`, `0x110` |
| `0x120` returns zero; `0x114` returns nonzero | `3` | `0x118`, `0x10C`, `0x110` |
| First two return zero; `0x118` returns nonzero | `4` | `0x10C`, `0x110` |
| First three return zero; `0x10C` returns nonzero | `1` | `0x110` |
| First four return zero; `0x110` returns nonzero | `2` | None |
| All five return zero | `0` | None |

In every row, the `0x13C` query precedes the listed classification calls.

### Separate output bytes

If the initial query supplies byte `0xA7`, the first classification condition returns zero, and the second returns nonzero, distinct non-null output destinations receive `0xA7` and `0x03`, respectively. The function returns zero.

The first byte is not stored until after the second classification condition has returned.

### Null destinations

If both output destinations are null, acquisition and all classification calls required by the observed results still occur. On normal completion, the function returns zero without either explicit output store.

If only one destination is null, only that destination's store is omitted.

### Aliased destinations

If both output destinations address the same writable byte, the initial query supplies `0xA7`, and classification selects `2`, the explicit store sequence is `0xA7` followed by `0x02` at that address. The final byte is `0x02`.

### Nonzero values are not limited to one

If the first classification query produces the 32-bit all-ones value, it satisfies the nonzero condition. Classification is zero, and no later classification query is made. The function does not visibly translate that value into an error.

### Acquisition failure

If acquisition returns null, F-a817d8d58c10 is invoked. No indirect query or explicit output store is required or permitted by the visible path. No specified normal return value follows from this evidence.

## Uncertainties

### Documented only

The following remain unresolved:

- The domain meanings of the initial byte and classifications.
- The concrete identities and complete signatures of indirect targets.
- Whether the initial result is intrinsically eight bits or narrowed from a wider result.
- Ownership and lifetime of the acquired object.
- Dependency preservation of globals, object state, and caller memory.
- Concurrency guarantees and validity under concurrent mutation.
- The actual control-transfer behavior after null acquisition.
- Behavior if an indirect target or acquisition dependency fails to return normally.
- Whether the original source imposes constraints not represented in the supplied body.

### Inferred intent

The ordered conditions appear to implement priority-based state classification, and the null path appears intended to delegate failure handling. These are interpretations, not additional guarantees.

The distinct classifications must not be assigned semantic names without further evidence.

## Evidence

This contract is based solely on static inspection of the supplied function representation and the provided opaque dependency mapping.

Established observations include:

- A single acquisition call followed by a null test.
- An initial byte-valued query before all classification queries.
- Five ordered zero/nonzero tests with short-circuit selection.
- Explicit classification values from zero through four.
- Two independently optional one-byte stores in a fixed order.
- A 32-bit zero return on the completed non-null path.
- A separate dependency call on the null acquisition path.

The non-return annotation is not treated as proof of a specific failure mechanism. No runtime tests have been performed, and no acceptance example is presented as an observed execution.
