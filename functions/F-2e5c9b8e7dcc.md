<!-- Trusted provenance: F-2e5c9b8e7dcc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2e5c9b8e7dcc

## Purpose

F-2e5c9b8e7dcc conditionally dispatches a 32-bit selection value to a downstream dependency. It uses a caller-supplied object, a request-like record, and comparisons against data obtained through the object’s virtual interface.

The established selection rules are:

- A positive signed field in the request enables an initial comparison attempt. If that comparison returns exactly zero, the function dispatches the request field’s then-current 32-bit value.
- Otherwise, a separate request query determines whether to dispatch the sentinel value `0xFFFFFFFF` or search the object’s indexed entries.
- During a search, the first entry whose comparison returns exactly zero supplies the dispatched index.
- An exhausted search produces no downstream dispatch.

“Request,” “selection,” and “entry” are descriptive roles, not established application-domain types. In particular, the evidence does not establish whether the request contains text, whether the comparison implements equality, or what the sentinel means to its consumer.

## Interface

### Inputs

The function has two explicit inputs and no explicit result value.

1. **Object reference**

   A pointer to an object represented using 32-bit fields and addresses in the supplied evidence.

   Relevant locations are:

   | Location | Observed use |
   |---|---|
   | Object offset `0x00` | Read as a virtual-dispatch-table pointer. |
   | Dispatch-table byte offset `0x110` | Indirect call producing a 32-bit unsigned search bound. |
   | Dispatch-table byte offset `0x13C` | Indirect call producing a temporary record. |
   | Object byte offset `0x40` | Read as the first argument to the final dispatch dependency. |

   The object field at `0x40` is passed by value. Its semantic type is not established.

2. **Request-record address**

   A 32-bit address-like input used to access:

   | Location | Observed use |
   |---|---|
   | Request offset `0x04` | Source address supplied to F-b9acea4b06ad. |
   | Request offset `0x0C` | Initially read as a signed 32-bit value; conditionally reread later as an unsigned 32-bit value. |

   The complete request layout is unknown. The evidence does not establish that these locations are independent fields: the representation beginning at offset `0x04` could extend into later bytes.

### Preconditions

For a defined execution of the observed operations:

- The object and request locations actually accessed must be readable.
- The object’s dispatch-table pointer and the entries used on the selected path must be valid.
- Each invoked dependency must accept the supplied arguments.
- The temporary storage passed to dependencies must meet their undisclosed representation requirements.
- Address calculations must designate valid accessible storage.

There are no visible null checks, bounds checks against a known request size, or object-type checks.

### Output

The function returns without an explicit return value.

Its principal observable result is either:

- One call to F-17445431f424 with an object-derived value and a selected 32-bit value; or
- No such call when an indexed search finishes without a qualifying comparison.

These statements assume invoked dependencies return normally. Exception, fault, and nonlocal-exit behavior are not specified by the supplied evidence.

## Behavior

### Initialization and initial decision

The function first invokes F-b02b795a8319 on a local holder. This occurs before the initial request-field read.

It then reads the 32-bit field at request offset `0x0C` and interprets it as signed:

- Values from `1` through `0x7FFFFFFF` enter the initial comparison path.
- Zero and values with the high bit set bypass that path and enter the fallback decision.

The signed interpretation matters. For example, `0xFFFFFFFF` does not enter the positive-field path.

### Initial comparison path

When the initial request field is positive, the function:

1. Invokes the object’s indirect entry at dispatch-table offset `0x13C`, supplying a temporary-result destination and the object.
2. Invokes F-46f0b3cb316e to transfer or construct an embedded portion of that result into the local holder.
3. Copies an adjacent 32-bit component of the temporary result into the holder’s corresponding component.
4. Invokes F-ee5919ca4be6 on the temporary-result record.
5. Constructs a temporary view of the holder’s embedded data through F-b9acea4b06ad.
6. Passes that view to F-44362605ab79 and retains its 32-bit result.
7. Constructs a separate request view through F-b9acea4b06ad, using request offset `0x04`.
8. Calls F-4a902fecdea3 with the request view and the retained result.
9. Invokes F-289ec4311c64 on the request view, then on the holder-data view.

The comparison outcome qualifies only when the 32-bit unsigned result of F-4a902fecdea3 is exactly zero.

If it qualifies, the function rereads the request field at offset `0x0C` as an unsigned 32-bit value. That reread supplies the selected value. It is not necessarily the value used for the earlier signed branch decision, because intervening dependencies may have changed accessible state.

The function then reads object offset `0x40` and invokes F-17445431f424 with that value and the selected value.

If the comparison does not qualify, execution continues to the fallback decision.

The initial indirect call has no explicit index argument in the supplied representation. It must not be specified as necessarily retrieving the entry named by the positive request field. The relationship between that field and this indirect call is **documented_only**.

### Fallback decision

The fallback decision is reached either directly from a nonpositive initial field or after an unsuccessful initial comparison.

The function:

1. Constructs a temporary request view through F-b9acea4b06ad using request offset `0x04`.
2. Queries that view with F-6d70e45e29c9.
3. Invokes F-289ec4311c64 on the view.
4. Branches using the retained query result.

If the query result is nonzero:

- The selected value is exactly `0xFFFFFFFF`.
- The function reads object offset `0x40`.
- It invokes F-17445431f424 with that object-derived value and the sentinel.

The meaning of a nonzero query result is not established. Calling it “empty,” “invalid,” or “unrestricted” would introduce unsupported semantics.

If the query result is zero, the function begins an indexed search.

### Indexed search

The function calls the object’s indirect entry at dispatch-table offset `0x110` once and retains its result as a 32-bit unsigned bound.

The search examines indices in ascending order, beginning with zero, while the current unsigned index is less than that retained bound. It does not refresh the bound between entries.

For each index:

1. The function invokes the indirect entry at dispatch-table offset `0x13C`, supplying a temporary-result destination, the object, and the current index.
2. It transfers or constructs the result’s embedded portion into the local holder through F-46f0b3cb316e.
3. It copies the adjacent 32-bit component into the holder.
4. It invokes F-ee5919ca4be6 on the temporary-result record.
5. It creates a holder-data view through F-b9acea4b06ad.
6. It obtains a 32-bit value from that view through F-44362605ab79.
7. It creates a request view from request offset `0x04`.
8. It invokes F-4a902fecdea3 with the request view and the obtained value.
9. It invokes F-289ec4311c64 on the request view and then on the holder-data view.

A zero comparison result terminates the search immediately. The current index becomes the selected value. The function reads object offset `0x40` and calls F-17445431f424.

A nonzero comparison result advances to the next index.

If no index qualifies, the function performs no final dispatch.

#### Search boundaries

- A bound of zero causes no entry-retrieval calls.
- A bound of one permits only index zero.
- Even a bound of `0xFFFFFFFF` excludes index `0xFFFFFFFF`: the last possible examined index is `0xFFFFFFFE`.
- Under this loop condition, normal loop completion does not require the index to wrap around.

The bound’s snapshot behavior does not establish that the object’s entries remain unchanged. Dependencies may mutate the object or other accessible state.

### Comparison arithmetic

The source expresses qualification using unsigned subtraction and an additional range condition. Its effective predicate is exactly:

**The 32-bit unsigned comparison result equals zero.**

For a result of zero, subtraction yields one and the path qualifies. For a result of one, subtraction yields zero. For results greater than one, the subsequent condition forces rejection. Unsigned wraparound for larger values does not make those values qualify.

Consequently, neither a conventional Boolean “true” value of one nor a negative-looking bit pattern such as `0xFFFFFFFF` qualifies.

### Normal completion

All normal paths converge on F-ee5919ca4be6 applied to the local holder initialized at entry.

When a final dispatch occurs, that holder-finalization call happens after F-17445431f424 returns. When a search finds no qualifying entry, it happens after the search finishes.

## Side effects

### Established observable ordering

The following sequencing is part of the observed behavior:

- Holder initialization precedes the first request-field read.
- Each retrieved temporary record is passed to F-ee5919ca4be6 after its payload has been transferred and before comparison-view construction.
- The request comparison view is finalized before the holder-data comparison view.
- The fallback query view is finalized before its retained result controls dispatch or search.
- Object offset `0x40` is read only when a final dispatch is about to occur.
- Final dispatch precedes finalization of the persistent local holder.

The dispatch-table pointer is accessed for indirect calls as execution reaches them. No stable snapshot of the table or object contents is established.

### Memory and ownership

The function directly uses local storage for a persistent holder, a fetched temporary record, and temporary views. It also explicitly copies a 32-bit component from the fetched record into the holder.

The fetched-record and holder layouts appear to contain a leading four-byte region, an embedded region represented as eight bytes, and a trailing four-byte component. These apparent aggregate boundaries are inferred from the supplied local-storage layout and call addresses; complete types and helper access widths are unknown.

Calls named here as construction, transfer, or finalization describe their observed roles in the sequence. They do not establish allocation, ownership transfer, reference counting, or deallocation semantics.

In particular:

- No direct heap allocation or release operation is established.
- F-46f0b3cb316e may perform behavior beyond copying bytes.
- F-ee5919ca4be6 and F-289ec4311c64 must not be assumed to release memory.
- Replacing the holder’s embedded payload on successive search iterations has whatever ownership effects the dependencies implement; no separate prior-payload cleanup is visible at that point.
- No dependency is assumed to preserve globals, object fields, or request contents.

No direct write to caller-owned object or request storage is visible in the function itself. Such writes may occur through dependencies or indirect calls.

### Errors

There is no visible error return, diagnostic emission, recovery branch, or retry.

A nonzero comparison result is a selection rejection, not an established operational error. A nonzero fallback query result selects the sentinel path, not an established failure path.

## Dependencies

| Opaque dependency | Established role in this function |
|---|---|
| F-b02b795a8319 | Receives the persistent local holder before any request decision. |
| F-b9acea4b06ad | Receives destination storage and a source address to prepare temporary views. |
| F-6d70e45e29c9 | Queries a request view; zero enables search, nonzero selects the sentinel. |
| F-46f0b3cb316e | Receives destination and source embedded-record addresses during payload transfer. |
| F-ee5919ca4be6 | Receives fetched temporary records and, at normal completion, the persistent holder. |
| F-44362605ab79 | Produces the 32-bit argument subsequently supplied to the comparison dependency. |
| F-4a902fecdea3 | Produces the 32-bit unsigned comparison result; exactly zero qualifies. |
| F-289ec4311c64 | Receives temporary views after their last visible use. |
| F-17445431f424 | Receives the value read at object offset `0x40` and the selected 32-bit value. |

Two additional dependencies are indirect virtual targets identified only by dispatch-table offsets `0x110` and `0x13C`. No opaque target IDs are supplied for them. Their implementation identities and full contracts remain **documented_only**.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests. Each assumes dependencies return normally and provide valid representations.

1. **Successful positive-field attempt**

   The initial signed request field is `7`, and the initial comparison returns zero. No dependency changes that field before its reread.

   Expected: one final dispatch with selected value `7`; no fallback query and no search-bound query.

2. **Request mutation before dispatch**

   The initial request field is positive. An intervening dependency changes it to `0x80000000`, and the initial comparison returns zero.

   Expected: final dispatch receives `0x80000000`. The function does not repeat the signed positivity test.

3. **Nonpositive field and nonzero fallback query**

   The initial request field is zero or has its high bit set. The fallback query returns any nonzero 32-bit value.

   Expected: no initial comparison attempt; one dispatch with selected value `0xFFFFFFFF`; no indexed search.

4. **Initial rejection followed by first search match**

   The initial field is positive, but its comparison returns one. The fallback query returns zero. The search bound is three, and comparisons for indices zero and one return nonzero and zero respectively.

   Expected: index one is dispatched. Index two is not retrieved.

5. **Zero search bound**

   The fallback query returns zero and the search-bound call returns zero.

   Expected: no indexed entry retrieval and no final dispatch. The persistent holder is still finalized.

6. **Nonzero comparison bit patterns**

   A search comparison returns `0xFFFFFFFF`.

   Expected: that entry is rejected. Unsigned arithmetic does not convert this result into acceptance.

7. **Search exhaustion**

   The retained bound is two, and both comparisons return nonzero.

   Expected: indices zero and one are examined in order, no selection is dispatched, and normal holder finalization occurs.

8. **Late object-field observation**

   A dependency changes object offset `0x40` before a successful selection reaches dispatch.

   Expected: dispatch receives the value read after selection, not an earlier snapshot.

## Uncertainties

### Inferred intent

The function appears to resolve a request into either an existing indexed entry or a sentinel selection. The temporary operations resemble managed-record and view handling. Neither interpretation establishes concrete domain types or ownership rules.

### Documented-only unknowns

- Meaning and complete layout of the request and fetched records.
- Meaning of the fallback query and comparison operation.
- Meaning of the selected value to F-17445431f424.
- Full signatures and identities of the indirect targets.
- Why the initial entry-retrieval call lacks an explicit index while indexed retrieval supplies one.
- Allocation, aliasing, retention, and release behavior of all helpers.
- Mutation, reentrancy, synchronization, and exceptional-exit behavior of dependencies.
- Cleanup guarantees if a dependency does not return normally.

## Evidence

The supplied static function body establishes the branch conditions, 32-bit interpretations, field offsets, explicit copy, call order, search boundaries, sentinel value, and normal completion paths described above. The supplied dependency mapping establishes the opaque IDs used here.

Aggregate meanings, ownership semantics, and application intent are not established by that evidence.

No runtime tests have been performed.
