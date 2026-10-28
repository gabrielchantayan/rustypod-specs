<!-- Trusted provenance: F-26d1da26cf7b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-26d1da26cf7b

## Purpose

F-26d1da26cf7b searches an indirectly referenced collection for the first element whose associated record contains two specified 32-bit values. It returns the element’s zero-based index, or the 32-bit value `0xFFFFFFFF` when the search reaches the collection’s current bound without finding a match.

Before inspecting the collection, the function invokes F-30529d2a9890. This invocation occurs even when the collection is empty. The dependency’s purpose and effects are unknown; the search must not be described as operating on the state that existed before that call.

The established behavior is an ordered, first-match search. Descriptions such as “lookup by composite key” are inferred intent, not an established domain-level interpretation of the two fields.

## Interface

### Inputs

The function has four supplied parameters:

| Position | Representation in the supplied evidence | Established role |
|---|---|---|
| First | 32-bit integer used as an address base | Identifies the object containing the collection pointer and element count. |
| Second | 32-bit opaque value | Not explicitly read, compared, stored, or forwarded by the visible function body. |
| Third | 32-bit integer | Required value for the associated record’s field at byte offset `0x110`. |
| Fourth | 32-bit integer | Required value for the associated record’s field at byte offset `0x114`. |

The first parameter is an address carrier, not an integer that the function validates or converts into a separately checked handle.

The second parameter has no established direct effect on the search. This does not establish that a called dependency cannot observe ambient machine state or otherwise behave differently between invocations.

### Return value

The return value is an unsigned 32-bit quantity:

- A matching element produces its zero-based index.
- Failure to find a match produces `0xFFFFFFFF`, numerically 4,294,967,295.

The failure value is not a successful index under the visible control flow. Before an element could be inspected at that index, the unsigned bound check would necessarily terminate the search: no unsigned 32-bit count exceeds `0xFFFFFFFF`.

Although the sentinel has the same bit pattern as signed 32-bit negative one, the visible return representation is unsigned.

### Required memory layout

All offsets below are byte offsets. Pointer-bearing storage is represented by 32-bit quantities in the supplied evidence.

| Location | Contents used by this function |
|---|---|
| Base object plus `0xEEC` | Address of a table of element addresses. |
| Base object plus `0xEF4` | Unsigned 32-bit element count. |
| Table plus four times the current index | Address of an element object. |
| Element object plus `0x20` | Address of an associated record. |
| Associated record plus `0x114` | 32-bit field compared with the fourth parameter. |
| Associated record plus `0x110` | 32-bit field compared with the third parameter, conditionally read. |

Each table slot occupies four bytes. The table does not directly contain the records whose comparison fields are examined: two levels of address retrieval separate a table slot from those fields.

### Preconditions

For defined, normally completing execution, F-30529d2a9890 must return, and every memory access actually reached by the search must be valid in the execution environment.

In particular:

- The count storage must be readable whenever a bound check occurs.
- When the count admits an index, the table pointer, selected table slot, selected element’s record-pointer field, and record’s `0x114` field must be readable.
- The record’s `0x110` field must be readable if its `0x114` field matches the fourth parameter.
- Address calculations and resulting accesses must be meaningful under the target’s address model.

No null checks, accessibility checks, table-size checks, or record validation appear in the visible body. The function does not establish these preconditions itself.

## Behavior

### Dependency invocation precedes the search

F-30529d2a9890 is invoked once before the visible collection reads. No explicit arguments are shown, and no return value from that invocation is used.

Only after this invocation returns does the search begin with index zero. Consequently, any changes the dependency makes to the base object, its table, its count, or reachable records may affect the search.

An initially empty collection does not bypass this dependency.

### Bounds are checked before each element

At each candidate index, the function reads the unsigned 32-bit count at base offset `0xEF4`.

If that count is less than or equal to the current unsigned index, the function immediately returns `0xFFFFFFFF`. No table slot or record is inspected for that iteration.

The count is read again on later iterations rather than captured once for the entire search. This distinction matters if memory changes during execution. The function does not establish a stable snapshot.

### Record selection follows the current table

For an index admitted by the bound check, the visible access sequence is:

1. Read the table address from base offset `0xEEC`.
2. Read the element address from the table slot selected by the current index, using a four-byte stride.
3. Read the associated-record address from element offset `0x20`.
4. Read the associated record’s field at offset `0x114`.

The table address is obtained for each visited candidate. It is not visibly cached across the search.

Within a candidate, the associated-record address obtained through the element is the address used for both potential field comparisons. The function does not visibly retrieve that address again between comparisons.

### Comparison order is significant

The field at record offset `0x114` is compared with the fourth parameter first.

- If it differs, the candidate fails without a visible read of the field at `0x110`.
- If it matches, the field at `0x110` is read and compared with the third parameter.
- If both comparisons succeed, the current index is returned immediately.

Although the supplied representation treats the comparison fields and the third and fourth parameters as signed integers, only equality is used. There is no signed ordering, range restriction, coercion between field widths, or arithmetic involving the requested values. Equal 32-bit representations match, including representations commonly interpreted as negative integers.

### Search progression and first-match behavior

After a failed candidate, the index increases by one and the next iteration begins with another count read.

For a stable collection:

- Candidates are considered in ascending index order.
- The first matching candidate is returned.
- Later candidates are not inspected after a match.
- A complete unsuccessful search examines all indices from zero through count minus one, then returns the sentinel.

The index is unsigned and 32 bits wide. It cannot wrap during the visible search: once it reaches `0xFFFFFFFF`, the next bound check must terminate before an element access or another increment. Even a count of `0xFFFFFFFF` admits indices only through `0xFFFFFFFE`.

### Failure behavior

The sentinel represents ordinary search exhaustion, including an empty collection. There is no separate visible return value for a malformed table, an invalid record address, or a failure inside F-30529d2a9890.

Invalid memory or a dependency that fails to return can prevent a normal return. The supplied evidence does not define exception handling, fault recovery, diagnostics, or translation of such conditions into `0xFFFFFFFF`.

## Side effects

The visible search body performs memory reads but no explicit writes to the base object, table, elements, or associated records. Its changing index is internal search state.

This does not make the function as a whole side-effect-free. F-30529d2a9890 runs before the search, and its effects are unknown. In particular, the available evidence does not establish whether that dependency:

- Writes shared or global state.
- Changes the collection or any referenced object.
- Acquires or releases synchronization resources.
- Allocates, retains, releases, or invalidates memory.
- Produces externally observable output.

The search itself does not visibly allocate, free, retain, or transfer ownership of any object. Its successful result is an integer index, not an owned reference or pointer. No lifetime extension for the selected element or associated record is established.

There are no visible synchronization operations in the search body. Repeated count and table reads do not provide a consistency guarantee. If the collection can change concurrently, validity, lifetime, and synchronization requirements remain external to this function; the evidence does not define a safe concurrent-mutation protocol.

The access order described above is the order expressed by the supplied static representation. It is not a separately established hardware memory-ordering guarantee.

## Dependencies

### F-30529d2a9890

Established contract at this call site:

- Invoked exactly once on entry to the visible body.
- Invoked before the search’s visible collection reads.
- Receives no explicit arguments in the supplied representation.
- Has no return value consumed by this function.
- Must return for the visible search to proceed.

Everything else about the dependency is unknown. It must not be labeled an initializer, lock operation, validator, allocator, or cleanup operation without additional evidence.

The dependency’s ownership behavior and preservation of globals are not established. Any claim that the search leaves shared state unchanged would therefore exceed the available evidence.

No other function calls appear in the supplied body.

## Acceptance examples

These examples are conditional static expectations, not reports of executed tests. Unless otherwise stated, assume that F-30529d2a9890 returns, all reached accesses are valid, and the collection remains stable afterward.

### Empty collection

Given a count of zero after the dependency returns:

- The dependency has still been invoked.
- The first count check terminates the search.
- No table slot or record field is inspected.
- The return value is `0xFFFFFFFF`.

For this path, the search does not require a readable table target because it never follows the table pointer.

### Match at index zero

Given a positive count and an associated record for index zero whose `0x114` field equals the fourth parameter and whose `0x110` field equals the third parameter:

- The `0x114` comparison occurs first.
- The `0x110` comparison occurs second.
- The return value is zero.
- No later candidate is inspected.

### First comparison rejects a candidate

Given a valid candidate whose `0x114` field differs from the fourth parameter:

- That candidate does not match, regardless of its `0x110` value.
- The visible body does not read its `0x110` field.
- Search continues at the next index after another count check.

This conditionally permits the current candidate’s `0x110` storage to remain unread; it does not make other invalid reachable storage acceptable.

### Only one of the two fields matches

Given a matching `0x114` field but a differing `0x110` field:

- Both fields are read, in that order.
- The candidate is rejected.
- A later candidate can still supply the result.

### Duplicate matches

Given matching candidates at indices two and five, with no earlier match:

- The return value is two.
- Index five is not inspected by this invocation.

The contract is first-match selection, not uniqueness checking.

### Exhaustion

Given three candidates, none matching both requested values:

- Candidates zero, one, and two are considered in order.
- A subsequent count check at index three ends the search.
- The return value is `0xFFFFFFFF`.

### Dependency changes the collection

Given that the dependency changes the count from a positive value to zero before returning:

- The search observes the post-call count.
- It returns the sentinel without inspecting a candidate.

The pre-call count does not determine the result.

### Count changes between iterations

If an external mechanism validly changes the count from three to one after index zero fails but before the next count read:

- The next iteration checks index one against the newly observed count.
- That check terminates the search.
- Index one is not inspected.

This illustrates repeated bound reads; it does not establish that unsynchronized mutation is supported.

### High-bit comparison values

Given requested values and record fields with equal 32-bit representations, including representations with the high bit set:

- Those field comparisons succeed.
- The values are not rejected merely because a signed interpretation would be negative.

## Uncertainties

The following are **documented_only** unknowns:

- The domain meaning of the base object, elements, associated records, and comparison fields.
- The purpose and complete behavior of F-30529d2a9890.
- The exact calling convention and whether the dependency can observe values beyond its explicit argument list.
- Allocation sizes, actual table capacity, alignment requirements, and pointer provenance.
- The exact target rules for address arithmetic near the address-space boundary.
- Object ownership, lifetime guarantees, and invalidation rules.
- Any required caller-held lock or other synchronization discipline.
- Fault behavior for inaccessible memory or a non-returning dependency.
- Whether any broader interface assigns meaning to the otherwise unused second parameter.

The evidence supports a 32-bit address-bearing layout and four-byte table slots. It does not independently establish a complete platform ABI.

“Associated record” and “collection” are descriptive terms for the observed layout. They do not imply a particular application-level type or ownership relationship.

## Evidence

Established static evidence consists of the supplied body for F-26d1da26cf7b and the opaque mapping identifying its sole explicit dependency as F-30529d2a9890.

That body establishes the pre-search call, unsigned count checks, four-byte table stride, pointer-indirection layout, ordered conditional field reads, ascending search, first-match return, and `0xFFFFFFFF` exhaustion result. It also establishes the absence of explicit search-body writes and the lack of direct use of the second parameter.

Inferred intent is limited to describing the operation as a first-match search over a two-field criterion. Domain semantics, dependency effects, ownership, concurrency guarantees, and platform-specific fault behavior remain unknown.

No runtime tests have been performed.
