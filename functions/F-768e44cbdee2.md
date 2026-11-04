<!-- Trusted provenance: F-768e44cbdee2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-768e44cbdee2

## Purpose

F-768e44cbdee2 selects an entry from a caller-supplied collection and, when that entry contains a nonzero target reference, conditionally invokes an operation reached through that target. Two dependencies bracket the target inspection and optional operation.

Selection has two modes:

- A zero selector chooses the entry at index zero without invoking the comparison dependency.
- A nonzero selector searches entries in increasing index order until the comparison dependency returns zero or the observed signed entry count no longer permits another comparison.

The function returns the selected operation’s 32-bit result if that operation is invoked and returns normally. Otherwise, its normal return value is the 32-bit value 1.

The source establishes this selection and dispatch behavior. It does not establish the application-level meaning of the collection, selector, target, bracketing calls, operation, or return value. In particular, returning 1 does not independently establish success.

## Interface

### Arguments

The function accepts four arguments:

| Argument | Representation visible in the source | Role |
|---|---|---|
| Collection reference | 32-bit address-bearing value | Identifies a structure containing a signed entry count and an entry-array reference. |
| Selector | 32-bit value | Zero bypasses comparison; nonzero is passed unchanged to the comparison dependency. |
| First operation argument | Uninterpreted 32-bit value | Passed unchanged to the indirect operation, if invoked. |
| Second operation argument | Uninterpreted 32-bit value | Passed unchanged to the indirect operation, if invoked. |

Although the selector may represent a reference, the source establishes only its zero test and its use as an argument to the comparison dependency. Its format, validity requirements, and ownership are unknown.

The final two arguments are not directly dereferenced by this function. Their interpretation belongs to the indirect operation.

### Return value

The result is a 32-bit value:

- Initially 1.
- Replaced by the indirect operation’s returned value only when that operation is called and returns normally.
- Not replaced by any result from the final bracketing dependency.

No separate error indicator is visible. The function does not distinguish, through its own return convention, an absent entry, a zero target reference, a zero dispatch-table reference, and an indirect operation that itself returns 1.

### Collection layout

The visible structure layout is:

| Byte offset | Width | Meaning used by this function |
|---|---:|---|
| 4 | 32 bits | Signed entry count |
| 8 | 32 bits | Entry-array base reference |

The meaning of bytes before, between, or after these fields is not established beyond these accesses.

Each array entry occupies 24 bytes:

| Offset within entry | Width | Meaning used by this function |
|---|---:|---|
| 0 | 32 bits | Value passed as the comparison dependency’s first argument |
| 4 | 32 bits | Target reference used after selection |
| 8–23 | Not accessed directly | Meaning unknown |

The stride is 24 bytes regardless of which entry fields are read.

### Target and dispatch layout

For a selected nonzero target, the function follows this reference chain after calling F-0b4658e20c99:

1. Read a 32-bit reference at target byte offset 4.
2. Dereference that reference to obtain another 32-bit reference.
3. Read a 32-bit reference at byte offset 108 from the resulting address. This identifies the dispatch receiver.
4. Read the first 32-bit word of the dispatch receiver. This is treated as a dispatch-table reference.
5. If the dispatch-table reference is nonzero, obtain the callable reference at its byte offset 40 and invoke it.

The indirect operation receives the dispatch receiver as its first argument, followed by the original third and fourth arguments. The selected target is not itself the receiver argument.

### Preconditions

For normal execution, every address actually accessed must designate suitable readable storage, and every invoked callable reference must be valid under the applicable calling convention.

The collection reference is not checked for zero. The function also does not check:

- The entry-array reference before accessing an entry.
- Intermediate references in the target’s reference chain.
- The dispatch receiver before reading its first word.
- The callable reference at dispatch-table offset 40.

A zero dispatch-table reference is explicitly handled. A zero dispatch receiver is not.

The concrete machine’s alignment, address-wrap, and fault rules are not supplied. Valid inputs must avoid depending on unspecified arithmetic or invalid memory accesses.

## Behavior

### 1. Initialize the fallback result and candidate index

The function begins with a result value of 1 and a candidate index of zero.

The index and collection count are represented as signed 32-bit integers in the supplied source. Bounds decisions use signed less-than comparisons, not unsigned comparisons.

### 2. Select a candidate

#### Zero selector

When the selector is zero, the search loop is skipped completely. The candidate remains index zero.

The function then reads the collection count for the eligibility check described below. It does not search for a special zero key and does not call the comparison dependency.

#### Nonzero selector

When the selector is nonzero, each search iteration proceeds in this order:

1. Read the collection’s signed count.
2. Compare the current candidate index against that count.
3. If the candidate is not less than the count, end the search without comparing that entry.
4. Otherwise, read the collection’s current entry-array reference.
5. Read the 32-bit comparison value at offset zero of the candidate’s 24-byte entry.
6. Invoke the comparison dependency with that value and the original selector.
7. If the dependency returns zero, stop searching with this candidate.
8. If it returns any nonzero value, increment the candidate index by one and repeat.

Only zero versus nonzero matters for the comparison result. Positive and negative nonzero results both cause the search to continue.

The count and array reference are not captured once for the entire operation. The source contains fresh reads at the points described above. If a dependency changes the collection, later decisions can use the changed fields. No snapshot consistency is established.

### 3. Recheck candidate eligibility and load the target

After either selection mode, the function reads the collection count again and checks whether the candidate index is less than that signed count.

If this check fails, the function returns 1 without loading a target or calling either bracketing dependency.

If the check succeeds:

1. Read the collection’s current entry-array reference.
2. Read the target reference at offset 4 of the candidate’s 24-byte entry.
3. If that target is zero, return 1 without either bracketing call.

This target read is separate from any earlier comparison-value read. A comparison dependency can therefore cause the eventual target to come from changed array storage or changed entry contents.

The post-search eligibility check does not recompare the selected entry’s comparison value. If the comparison dependency returned zero and then changed the entry or array, the function still uses the existing candidate index.

### 4. Enter the target-processing phase

For a nonzero selected target, invoke F-0b4658e20c99 with that target as its sole visible argument.

Its return value is not used. Its behavior, including any changes to memory used by the subsequent reference chain, is unknown.

The target value used for this invocation is retained for the final call to F-96dd78c4ca33. The function does not reload the target from the collection after F-0b4658e20c99 returns.

### 5. Resolve and conditionally invoke the operation

After F-0b4658e20c99 returns normally, follow the target and dispatch reference chain specified in the interface.

If the dispatch receiver’s first word is zero, skip the indirect operation. The result remains 1.

If that word is nonzero:

- Read the callable reference at dispatch-table byte offset 40.
- Invoke it with the dispatch receiver and the two original operation arguments.
- Replace the result with its returned 32-bit value.

No validation of that callable reference is visible. A nonzero dispatch-table reference alone does not establish that the callable reference is nonzero or valid.

### 6. Leave the target-processing phase

After either skipping the indirect operation or receiving its normal return, invoke F-96dd78c4ca33 with the retained selected target.

Its return value is ignored. Under normal call-return execution, the enclosing function then returns the retained result: either 1 or the indirect operation’s result.

The source does not establish exception handling, nonlocal-exit handling, or guaranteed cleanup after a failing dependency or invalid memory access.

## Side effects

### Direct effects

The function directly reads collection fields, entry fields, the target reference chain, the dispatch receiver, and, conditionally, a callable reference.

No direct write to collection, entry, target, receiver, or global storage is visible. Its visible assignments maintain the current index, retained references, and result.

This does not make the function side-effect-free: all invoked dependencies may have effects not described by the supplied source.

### Dependency ordering

For a nonzero selected target, the normal call order is:

1. F-0b4658e20c99.
2. Zero or one indirect operation.
3. F-96dd78c4ca33.

Comparison calls, if any, occur before this sequence. Neither bracketing dependency is called when the final candidate is ineligible or its target is zero.

There is no unconditional cleanup path shown for failures between the two bracketing calls.

### Ownership and lifetime

No allocation, explicit release, reference-count update, or ownership transfer is directly established.

The bracketing pattern may suggest paired resource or state management, but that interpretation is inferred intent only. It must not be treated as evidence that either dependency acquires a lock, increments a reference count, releases memory, or restores prior state.

Dependencies may change globals or reachable memory. Normal completion requires the objects needed by subsequent reads and calls to remain usable; the supplied source does not show how that condition is enforced.

## Dependencies

| Dependency | Established use | Unknown behavior |
|---|---|---|
| F-0b4658e20c99 | Called once with a nonzero selected target before resolving the dispatch receiver | Side effects, ownership effects, failure behavior, return meaning, and state guarantees |
| F-96dd78c4ca33 | Called once with the retained target after normal target processing; return ignored | Side effects, ownership effects, failure behavior, and relationship to F-0b4658e20c99 |
| Comparison dependency; opaque function ID unavailable | Called for eligible entries only when the selector is nonzero; receives an entry value and selector; zero stops search | Comparison semantics, argument formats, side effects, and failure behavior |
| Indirect operation; opaque function ID unavailable | Resolved through dispatch-table offset 40; receives receiver and two 32-bit arguments; its 32-bit result becomes the enclosing result | Concrete identity, argument meanings, side effects, ownership, and failure behavior |

The supplied dependency map does not identify the comparison dependency or the dynamically selected operation. Their identities remain documented_only rather than being assigned invented function IDs.

## Acceptance examples

These examples are conditional static acceptance criteria, not executed tests. Unless stated otherwise, assume valid accessible storage, normal dependency returns, and no dependency mutation of relevant state.

### Zero selector with a positive count

Given a selector of zero, a count of 2, and a nonzero target in entry zero:

- No comparison call occurs.
- Entry zero supplies the target regardless of its comparison value.
- F-0b4658e20c99 is called before dispatch resolution.
- F-96dd78c4ca33 is called after normal dispatch processing.

If the indirect operation returns 37, the enclosing function returns the 32-bit value 37.

### Zero or negative count

Given a count of zero or a negative signed 32-bit count:

- Candidate zero fails the eligibility comparison.
- No entry is read.
- No dependency is invoked.
- The return value is 1.

For a nonzero selector, the search’s initial bounds check also prevents any comparison call.

### Search stops on zero, not on a particular nonzero sign

Given three entries and comparison results of −1, 7, and 0:

- Entries zero, one, and two are compared in that order.
- Entry two is selected.
- Negative and positive nonzero comparison results both advance the search.

### Exhausted search

Given a stable count of 2 and nonzero comparison results for both entries:

- Exactly two comparison calls occur.
- The candidate reaches index 2.
- The final eligibility check fails.
- No target-processing call occurs.
- The return value is 1.

### Zero target after selection

Given an eligible selected entry whose target field is zero:

- Selection may already have invoked comparison dependencies.
- Neither F-0b4658e20c99 nor F-96dd78c4ca33 is called.
- No target reference chain is read.
- The return value is 1.

### Zero dispatch-table reference

Given a nonzero selected target and a valid reference chain ending in a receiver whose first word is zero:

- F-0b4658e20c99 is called.
- The indirect operation is skipped.
- F-96dd78c4ca33 is called.
- The return value is 1.

A zero receiver reference does not satisfy this example; it fails the prerequisite for reading the receiver’s first word.

### Collection mutation during comparison

Given that a comparison at candidate zero returns zero but changes the collection count to zero before returning:

- The search stops at candidate zero.
- The subsequent count read observes zero, assuming the mutation remains visible.
- The candidate is rejected.
- No target is loaded and no bracketing call occurs.
- The return value is 1.

Alternatively, if the comparison returns zero while replacing the array reference and leaving a positive count, the target is read from entry zero of the replacement array. The function does not retain the array reference used for the comparison.

### Dispatch state changed by the first bracketing call

Given that F-0b4658e20c99 changes the selected target’s reference chain:

- Dispatch resolution uses the fields read after that dependency returns.
- It does not use a receiver resolved before the call.

## Uncertainties

The following are documented_only:

- Application-level meanings of the collection, selector, targets, operation arguments, and return values.
- Comparison semantics beyond zero stopping the search.
- Concrete identities of the comparison and indirect-operation dependencies.
- Ownership, lifetime management, synchronization, and global-state effects of all dependencies.
- Behavior after faults, exceptions, nonlocal exits, or invalid callable references.
- Concurrency guarantees and whether other execution contexts may mutate accessed storage.
- Exact ABI, alignment requirements, and arithmetic behavior outside valid address and index ranges.

The visible scalar values and reference-bearing fields are 32 bits. Entry addressing uses a 24-byte stride. Very large indices can make the index-times-stride calculation or subsequent address additions exceed a 32-bit range. The source does not justify an unlimited mathematical address space or a portable overflow outcome. Normal acceptance examples exclude such overflow.

The paired calls plausibly surround a protected operation, but protection, locking, reference retention, and cleanup guarantees are not established.

## Evidence

This contract is based solely on static inspection of the supplied function body and the supplied opaque dependency mapping.

Established evidence includes:

- A default 32-bit result of 1.
- Signed count comparisons and an increasing candidate index.
- Zero-selector bypass of comparison.
- A 24-byte entry stride with comparison and target fields at offsets 0 and 4.
- Fresh collection reads during search and after selection.
- Conditional target processing bracketed by F-0b4658e20c99 and F-96dd78c4ca33.
- A target reference chain using offsets 4 and 108.
- Conditional indirect dispatch through table offset 40.
- Forwarding of the two trailing arguments and propagation of the indirect operation’s result.
- No visible direct writes to caller-owned structures.

No runtime tests have been performed. The acceptance examples describe consequences of the static evidence under their stated conditions; they are not execution results.
