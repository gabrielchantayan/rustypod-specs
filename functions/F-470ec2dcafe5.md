<!-- Trusted provenance: F-470ec2dcafe5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-470ec2dcafe5

## Purpose

F-470ec2dcafe5 counts traversal occurrences whose 32-bit comparison field equals a caller-supplied value. It obtains the traversal handle from a field in a caller-supplied object and delegates traversal initiation and advancement to two dependencies.

An initially zero handle produces the distinguished result −50 without invoking either dependency. An initially nonzero handle permits traversal, but does not guarantee that any entries exist.

The established behavior is an occurrence count, not necessarily a count of distinct objects. The supplied evidence does not establish the container type, traversal order, uniqueness of returned entries, or meaning of the comparison field.

## Interface

### Inputs

The function accepts two integer-valued arguments:

| Input | Interpretation |
|---|---|
| Object address | Address used as the base for accessing a handle field at byte offset +4. |
| Comparison value | Value compared for equality with a field at byte offset +8 in every nonzero entry returned by the traversal dependencies. |

The supplied representation uses integer quantities as both values and addresses. This contract describes the accessed integer fields and comparison values as 32-bit quantities. The actual calling convention, pointer representation, alignment requirements, and signed-overflow semantics require target information not included in the evidence.

The relevant object layouts are limited to these observed fields:

| Object | Byte offset | Observed use |
|---|---:|---|
| Input object | +4 | Integer-sized traversal handle; tested against zero and, if the test succeeds, read again as the initiation argument. |
| Returned entry | +8 | Integer-sized comparison field; tested for equality with the second input. |

No other fields are accessed directly by F-470ec2dcafe5. These offsets do not establish complete structure layouts, allocation sizes, or the purpose of intervening bytes.

### Output

The function returns one integer result:

- **−50:** the first read of the handle field yields zero.
- **0:** traversal is initiated and terminates without any matching occurrences.
- **A positive count:** traversal terminates after that many matching occurrences, provided the count remains representable.

The count is an integer, not a Boolean result. Multiple matches increase it multiple times.

There is no explicit output buffer, callback argument, or ownership-transfer parameter.

### Preconditions

For ordinary defined execution:

- The input object address must permit an integer-sized read at byte offset +4.
- On the nonzero-handle path, that location must also permit the second read.
- Each nonzero entry returned by either dependency must permit an integer-sized read at byte offset +8.
- The traversal dependencies must be callable under their own contracts.
- Traversal must eventually return zero if normal termination is required.
- The match count must remain representable if a defined numerical result is required without additional overflow guarantees.

The function does not validate these conditions. In particular, it does not check whether the input object address itself is zero before accessing the field.

## Behavior

### Initial handle decision

The first externally relevant action is a read of the integer field at input-object offset +4.

If that read yields zero, the function returns −50. Neither traversal dependency is called, and no entry comparison field is read.

If that read yields a nonzero value, the function prepares a count of zero and proceeds to traversal initiation.

The handle used for initiation comes from a **second read** of the same field. The tested value is not retained as the initiation argument. Consequently, if the field changes between reads, initiation receives the later value. There is no second zero check before the dependency call.

This distinction establishes the read sequence; it does not establish that concurrent mutation is safe or supported. If both reads observe stable storage, they obtain the same handle.

### Traversal initiation

F-63a239e63e43 is called once with the handle obtained by the second field read. Its return value determines whether an entry is available:

- Zero ends traversal immediately.
- Any nonzero value is treated as an entry address.

There is no separate interpretation of negative return values or other nonzero sentinel values. Such values would also be treated as addresses for the subsequent field access.

If initiation returns zero, F-470ec2dcafe5 returns zero. F-bbc912f1c01e is not called in that case.

### Processing entries

For every nonzero entry returned by initiation or advancement:

1. The integer field at entry offset +8 is read.
2. That field is compared for equality with the comparison input.
3. A match increases the count by one; a nonmatch leaves the count unchanged.
4. F-bbc912f1c01e is called with no explicit arguments to obtain the next traversal result.

A zero result from advancement terminates traversal and causes the current count to be returned. A nonzero result causes another entry to be processed in the same way.

This ordering has several consequences:

- Every returned nonzero entry is compared before advancement is requested.
- Advancement occurs after both matching and nonmatching entries.
- There is no early return after the first match.
- The function does not deduplicate entries.
- The function does not reread the input object's handle field after traversal initiation.
- The final entry is followed by an advancement call that returns zero.
- No entry field is read from a zero traversal result.

For a normally terminating traversal with \(N\) processed occurrences, initiation is called once and advancement is called \(N\) times.

### Equality and counting boundaries

The comparison is integer equality. The function does not mask bits, apply a range test, compare only part of the field, or normalize the input.

Within the ordinary signed 32-bit range, negative comparison values are handled by the same equality operation as nonnegative values. Zero is a valid comparison value: an entry whose comparison field is zero can match even though a zero **entry result** terminates traversal.

The count starts at zero and is incremented once per match. No overflow guard or saturation behavior is present. For no more than 2,147,483,647 matches, the result fits a signed 32-bit integer. Behavior beyond that boundary is not established by the supplied evidence; wraparound must not be assumed.

## Side effects

### Direct accesses

F-470ec2dcafe5 directly reads:

- The input object's handle field once on the −50 path.
- That field twice on the traversal path.
- One comparison field for every nonzero entry occurrence returned by the dependencies.

It performs no explicit writes to the input object, returned entries, or other caller-addressed storage. Its count is internal computation returned through the ordinary result.

There is no explicit allocation, deallocation, object destruction, or release operation in the supplied function body.

### Dependency-mediated effects

Both dependencies may have effects not visible in the supplied body. Their implementations are unavailable. In particular, this contract does not assume that they:

- Preserve global state.
- Preserve traversal state belonging to other callers.
- Leave the input object or returned entries unchanged.
- Allocate or release memory.
- Retain or relinquish ownership of any handle or entry.
- Are reentrant or safe for concurrent calls.
- Return promptly or terminate successfully.

The argumentless advancement operation is consistent with traversal state held outside the explicit argument list, but the location and scope of that state are unknown.

### Errors and interruption

The only explicit error-like result is −50 for an initially zero handle. The function does not explicitly distinguish an empty traversal from a traversal containing no matching entries; both return zero.

No local recovery is shown for invalid addresses, dependency failure, nontermination, or count overflow. A dependency returning an invalid nonzero address leads to an attempted comparison-field read rather than a handled error result. The resulting fault or failure behavior is outside the established contract.

## Dependencies

| Opaque function ID | Invocation | Result consumed by this function | Established local role |
|---|---|---|---|
| F-63a239e63e43 | Once on the initially nonzero-handle path, with the second handle-field read as its sole explicit argument | Zero or a value treated as an entry address | Supplies the initial traversal result |
| F-bbc912f1c01e | Once after every processed nonzero entry, with no explicit arguments | Zero or a value treated as the next entry address | Advances traversal |

The roles above describe how F-470ec2dcafe5 uses the results. They do not establish the dependencies' complete contracts.

No explicit cleanup dependency is called when traversal ends. That absence does not prove that resources are leaked, retained, or released: resource ownership and any dependency-internal cleanup remain unknown.

## Acceptance examples

These are conditional acceptance cases derived from static evidence, not runtime observations. Unless stated otherwise, assume valid readable addresses, normal dependency completion, stable relevant storage, and a representable count.

### Initially absent handle

Given an input object's first handle-field read yields zero:

- The result is −50.
- The handle field is read once.
- Neither dependency is invoked.
- No entry comparison field is read.

The comparison input does not affect this outcome.

### Empty traversal

Given both handle reads yield a valid nonzero handle and initiation returns zero:

- Initiation is invoked once with that handle.
- Advancement is never invoked.
- The result is zero.

### Matches interspersed with nonmatches

Given a comparison input of 7 and a finite traversal whose comparison fields, in returned order, are 7, −2, 7, and 0:

- Four entry fields are read.
- Advancement is invoked four times.
- The result is 2.

The nonmatching entries do not stop traversal.

### Negative comparison value

Given a comparison input of −3 and a finite traversal containing two occurrences with comparison field −3 and one with comparison field 3:

- The result is 2.
- Positive and negative values are not treated as interchangeable.

### Zero-valued comparison field

Given a comparison input of zero and one valid nonzero entry address whose comparison field is zero, followed by a zero advancement result:

- The entry is processed and counted.
- The result is 1.

The zero comparison field is data; the zero advancement result is termination.

### Repeated entry address

Given the same valid entry address is returned twice, its comparison field matches on both reads, and traversal then terminates:

- The field is read twice.
- Both occurrences count.
- The result is 2.

There is no distinct-address filtering.

### Changed handle between reads

Conditional on an environment in which a change between the two reads is defined and observable: if the first handle read is nonzero but the second is zero, initiation is still invoked, with zero.

The result then depends on initiation and any subsequent traversal. The function does not take the −50 return path merely because the second read is zero.

### Nonterminating traversal

If initiation returns a valid entry and advancement continues returning valid nonzero entries indefinitely, the function does not normally return. No iteration limit or cycle detection is established.

## Uncertainties

The following are **documented_only** rather than inferred guarantees:

- Exact ABI, pointer width, alignment rules, and target arithmetic behavior.
- Complete layouts and types of the input object and traversal entries.
- Semantic meaning of the comparison field.
- Handle validity requirements imposed by F-63a239e63e43.
- Traversal ordering, uniqueness, and coverage.
- Location, lifetime, and exclusivity of traversal state.
- Whether dependencies mutate the input object, entries, globals, or unrelated state.
- Memory ownership and release obligations.
- Dependency-specific failures and nonlocal exits.
- Behavior after signed count overflow.
- Concurrency, signal safety, and reentrancy guarantees.

The inferred intent is to count entries associated with a supplied object whose comparison field equals a requested value. “Associated,” “entries,” and “traversal” are explanatory terms, not proof of a particular container implementation.

## Evidence

The supplied static body establishes the initial zero-handle branch, two separate handle reads on the traversal path, one initiation call, comparison-field reads at byte offset +8, equality-based counting, argumentless advancement, zero-result termination, and the explicit −50 return.

The opaque dependency mapping establishes only which dependency IDs correspond to those calls. No dependency implementation or broader program context was supplied.

No runtime tests have been performed. The acceptance examples are consequences of the supplied control and data accesses under their stated conditions, not reports of executed behavior.
