<!-- Trusted provenance: F-b0d4afdb18cf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b0d4afdb18cf

## Purpose

F-b0d4afdb18cf routes a record through receiver-specific handlers according to its discriminator, selected receiver fields, and handler return values.

Its established behavior has three stages:

1. Give certain records an initial handling opportunity.
2. If handling has not terminated, invoke one of two general handlers.
3. If that handler returns zero, conditionally invoke a final handler.

A nonzero handler result generally terminates processing and is returned unchanged. One distinct branch invokes F-8ac0302a48ef and then returns the fixed value 1, without consulting that dependency’s result.

The apparent intent is event or message dispatch. That interpretation is inferred; the supplied material does not establish the application meaning of the discriminator values, receiver fields, or handlers.

## Interface

### Inputs

The function takes three arguments:

| Argument | Established use |
|---|---|
| Receiver | Address of an object containing a dispatch-table pointer and several comparison fields. Passed to every invoked handler or dependency. |
| Record | Address of a record containing a discriminator at byte offset +4 and, for some discriminators, a value at byte offset +8. Passed unchanged to handlers that take a record. |
| Third integer | Has no observable influence on the shown routing or return value. On the only branch that could compare against it, its working value is first replaced by a receiver field. |

The third argument remains part of the interface, but no meaningful behavioral role for its supplied value is established.

### Data layout and widths

The supplied representation uses integer reads and four-byte spacing between receiver fields. The behavioral model below treats those integer values as 32-bit quantities. The underlying ABI, pointer width, alignment requirements, and calling convention are not independently established.

Receiver-relative locations:

| Byte offset | Role |
|---|---|
| +0x00 | Pointer to a dispatch table. |
| +0x50 | Comparison value used before the optional final handler. |
| +0xB8 | Comparison value used for the first discriminator-15 route. |
| +0xC4 | Comparison value used for the second discriminator-15 route. |

Record-relative locations:

| Byte offset | Role |
|---|---|
| +0x04 | Integer discriminator. |
| +0x08 | Integer comparison value for discriminator 15; interpreted as a nested-record address for discriminator 22. |

For discriminator 22, the nested record is read at byte offset +4. Its value is compared with 42.

Dispatch-table locations:

| Byte offset within table | Role |
|---|---|
| +0x134 | Initial handler for the first discriminator-15 match. |
| +0x10C | General handler selected for discriminator 22 with nested value 42. |
| +0x110 | General handler selected in all other fallback cases. |
| +0xE0 | Optional final handler for a later discriminator-15 match. |

These are byte offsets, not ordinal method numbers. The actual target functions and any relationship among them are unknown.

### Output

The function returns an integer:

- A nonzero result from an invoked dispatch-table handler or F-c6fc31cde97f is returned unchanged.
- The branch invoking F-8ac0302a48ef returns exactly 1.
- If no invoked result terminates processing, the function returns 0.

There is no normalization of arbitrary nonzero results to 1. Negative results, if returned by an integer-valued handler, satisfy the same nonzero test and are propagated unchanged.

### Preconditions

Every memory location actually read on the selected path must be accessible and valid for the corresponding read. Every invoked dispatch-table entry must designate a callable target compatible with the invocation.

There are no visible checks for:

- A null receiver or record.
- A missing or invalid dispatch table.
- A null selected handler.
- An invalid nested-record address.
- Record length, object type, or alignment.

Only path-reached locations are required by the visible behavior. In particular, the nested-record read is not performed unless the fallback discriminator read yields 22.

## Behavior

### 1. Initial classification

The function first reads the discriminator at record offset +4.

If that value is 15, it next reads:

1. The comparison value at record offset +8.
2. The receiver field at offset +0xB8.

If those values are equal, it reads the receiver’s dispatch-table pointer, selects the entry at table offset +0x134, and invokes that target with the receiver and original record address.

- A nonzero result is returned immediately.
- A zero result proceeds directly to fallback processing.

Once this first discriminator-15 match selects the initial handler, a zero result does **not** cause the second discriminator-15 route to be tried. The first match therefore has priority over the second match.

The record and receiver may have been changed by the handler. Fallback processing reads the then-current state rather than reusing the initial discriminator.

### 2. Other initial routes

If the initial discriminator is 0x7F24, the function invokes F-c6fc31cde97f with the receiver and record.

- A nonzero result is returned unchanged.
- A zero result proceeds to fallback processing.

This route is selected using the initial discriminator read. Later changes made by the dependency do not retroactively alter that selection.

If the initial discriminator is 15 but the first comparison did not match, the function performs a second comparison. It reads the record value at +8 again, then reads the receiver field at +0xC4.

If these newly read values are equal:

1. F-8ac0302a48ef is invoked with the receiver only.
2. The function returns exactly 1.

There is no fallback processing after that invocation returns normally. The dependency’s return value, if any, is not used.

If the second comparison does not match, processing continues to fallback.

All other initial discriminator values also proceed to fallback without either named dependency being invoked.

### 3. General fallback selection

On entering fallback, the function reads the record discriminator at +4 again. This is a fresh read even if the initial stage already examined that field.

If this discriminator is 22, the function:

1. Reads the value at record offset +8 as a nested-record address.
2. Reads an integer at that address plus 4.
3. Selects the dispatch-table entry at +0x10C if the nested value equals 42.

If the discriminator is not 22, or if its nested value is not 42, the function selects the entry at +0x110.

After making this selection, the function reads the receiver’s dispatch-table pointer and the selected entry, then invokes the target with the receiver and original record address.

A nonzero result is returned unchanged. A zero result advances to the final conditional stage.

The general handler runs whenever fallback is reached. It is not conditional on a discriminator-15 match.

### 4. Final conditional handling

After the general handler returns zero, the function reads the record discriminator at +4 yet again.

If this read yields 15, it reads:

1. The comparison value at record offset +8.
2. The receiver field at offset +0x50.

If these values are equal, it reads the receiver’s current dispatch-table pointer and entry at +0xE0, then invokes that target with the receiver and original record address.

- A nonzero result is returned unchanged.
- A zero result leads to a final return of 0.

If the discriminator is not 15, or the comparison does not match, the function returns 0 without invoking the final handler.

### Read timing and precedence

The established access sequence matters when handlers mutate the receiver or record:

- The first discriminator is retained for initial routing; it is not reread between the first and second discriminator-15 comparisons.
- The record comparison value is reread for the second discriminator-15 comparison.
- Fallback always rereads the discriminator.
- The final stage rereads the discriminator after the general handler returns.
- Dispatch-table pointers are read at the point of each selected virtual invocation, rather than retained across calls.

Consequently, a handler that returns zero can affect later routing through mutations. The function does not establish any guarantee that dependencies preserve the record, receiver fields, or dispatch-table pointer.

The listed ordering describes explicit accesses in the supplied static representation. It does not establish concurrency synchronization, volatile semantics, atomicity, or machine-level memory ordering.

## Side effects

### Direct effects

No direct write to receiver or record storage is visible. The function reads fields, invokes selected targets, and produces an integer result.

Replacing local working values does not write back to the caller’s third argument.

### Dependency-mediated effects

Every invoked target may have externally observable effects. The supplied material does not constrain those effects or show whether a target:

- Mutates the receiver or record.
- Changes the receiver’s dispatch table.
- Changes global state.
- Allocates or releases memory.
- Retains any passed address.
- Transfers control without returning normally.

No ownership transfer is explicitly established by the call structure. Conversely, absence of a visible release in this function does not prove that its dependencies preserve ownership.

If a dependency returns zero and processing continues, later reads must remain valid. A dependency that releases the receiver or record without arranging valid subsequent access would violate the preconditions of the continuing path.

### Errors

The function contains no explicit validation failure result, error reporting, exception handling, or recovery branch.

A nonzero handler result is returned without interpretation; the supplied material does not establish whether particular values mean success, failure, or another status.

Invalid memory or an invalid call target has no defined recovery behavior in the supplied evidence.

## Dependencies

| Dependency | Invocation condition | Inputs | Result handling |
|---|---|---|---|
| F-c6fc31cde97f | Initial discriminator is 0x7F24. | Receiver and record. | Return nonzero unchanged; continue on zero. |
| F-8ac0302a48ef | Initial discriminator is 15, first comparison fails, and the second comparison matches. | Receiver only. | Ignore any result; return 1 after normal completion. |
| Unidentified target at table +0x134 | First discriminator-15 comparison matches. | Receiver and record. | Return nonzero unchanged; enter fallback on zero. |
| Unidentified target at table +0x10C | Fallback discriminator is 22 and nested value is 42. | Receiver and record. | Return nonzero unchanged; advance on zero. |
| Unidentified target at table +0x110 | Any other fallback classification. | Receiver and record. | Return nonzero unchanged; advance on zero. |
| Unidentified target at table +0xE0 | Final discriminator is 15 and the value matches receiver +0x50. | Receiver and record. | Return nonzero unchanged; otherwise return 0. |

The supplied dependency map also identifies F-d5dd567991e2. The shown control flow does not establish a call to it: the corresponding reference is an internal branch destination. No external behavior or side effect is attributed to that ID here.

Distinct table offsets do not prove distinct target functions. The targets could alias, and their identities could change through receiver mutation.

## Acceptance examples

These are conditional static acceptance cases, not results of executed tests. Unless stated otherwise, all reached addresses and call targets are valid, and invoked targets return normally.

1. **First match propagates a result.**  
   The initial discriminator is 15, and the record value equals receiver +0xB8. The +0x134 target returns −7. The function returns −7. No second comparison, fallback handler, or final handler is invoked.

2. **First match has priority even when both receiver fields match.**  
   The initial discriminator is 15, and its value equals both receiver +0xB8 and receiver +0xC4. The +0x134 target returns zero. Processing enters fallback; F-8ac0302a48ef is not invoked merely because the second field also matches.

3. **Second match produces a fixed result.**  
   The initial discriminator is 15. The first comparison fails, while the newly read value matches receiver +0xC4. F-8ac0302a48ef is invoked once with the receiver. If it returns normally, the function returns 1 without further routing.

4. **Dedicated route propagates nonzero.**  
   The initial discriminator is 0x7F24. F-c6fc31cde97f returns 9. The function returns 9 without entering fallback.

5. **Dedicated route can change fallback selection.**  
   The initial discriminator is 0x7F24. F-c6fc31cde97f changes the record discriminator to 22, installs a valid nested-record address whose +4 value is 42, and returns zero. Fallback selects table +0x10C using the changed record.

6. **Nested reads are conditional.**  
   Fallback reads a discriminator other than 22. The function selects table +0x110 without reading the record’s +8 field for fallback classification or dereferencing it as a nested address.

7. **Final routing observes general-handler mutations.**  
   The general handler changes the record discriminator to 15, makes its +8 value equal receiver +0x50, and returns zero. The function selects the current table’s +0xE0 entry. If that target returns 4, the function returns 4.

8. **A zero general result need not invoke a final handler.**  
   The general handler returns zero, and the subsequent discriminator read is not 15. The function returns 0 without reading receiver +0x50 or selecting table +0xE0.

9. **The third input does not select a route.**  
   With identical reachable memory, dependency behavior, and handler results, varying only the supplied third integer does not change the established routing or return value.

## Uncertainties

The following remain **documented_only**:

- The semantic meaning of discriminator values 15, 22, and 0x7F24, and nested value 42.
- The identities, contracts, ownership effects, and global-state effects of dependency targets.
- The runtime ABI, pointer width, and exact calling convention.
- Whether nonzero results denote successful handling, errors, or other statuses.
- Whether callers impose additional validity or lifetime requirements.
- Whether shared-state access is permitted concurrently and, if so, what synchronization exists.
- Whether any target can throw, unwind, terminate, or otherwise fail to return normally.

The event-dispatch interpretation is inferred intent, not an established application-level contract. No preservation guarantee for globals or passed storage is inferred from a dependency returning zero.

## Evidence

This contract is based solely on the supplied static function representation and dependency map.

Established evidence includes explicit field offsets, comparison constants, conditional reads, call arguments, dispatch-table offsets, return-value tests, fresh reads after calls, and the absence of direct receiver or record writes in the shown function.

The 32-bit integer model follows the supplied integer representation and four-byte receiver indexing; it is not an independently verified ABI finding.

No implementation access, runtime execution, or runtime tests were performed. The acceptance examples describe behavior implied by the supplied static evidence under their stated preconditions.
