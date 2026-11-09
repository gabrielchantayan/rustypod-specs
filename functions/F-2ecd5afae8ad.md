<!-- Trusted provenance: F-2ecd5afae8ad; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2ecd5afae8ad

## Purpose

F-2ecd5afae8ad performs a state-dependent operation on a mutable object. An entry selector can bypass resource processing, invoke a notification-like dependency, or permit resource retrieval and presentation of two temporary values to object callbacks. A preliminary condition can instead select a separate transition.

Every explicitly represented normal return produces the four-byte value **1**. Entry bypass, retrieval failure, callback rejection, callback acceptance, and the alternate transition therefore cannot be distinguished by the return value alone. Their observable differences are dependency calls, callback calls, and byte-sized object writes.

“Resource,” “state,” “control,” “outcome,” and “derived storage” describe observable roles, not established domain types. The contract separates represented behavior from unknown dependency semantics. No runtime tests have been performed.

## Interface

### Explicit input and object layout

The sole explicit input is an object address. The supplied representation uses four-byte integer words and a 32-bit address model. These representation facts do **not** establish the type or access width of separately referenced external data.

| Object offset | Represented access | Role |
|---|---|---|
| 0x000 | 32-bit read | Dispatch-table address used for indirect callbacks |
| 0x13C | 32-bit read | Associated-resource value passed to dependencies |
| 0x140 | 32-bit read | Entry-selection word, converted to the represented character type |
| 0x140 | 8-bit write | State location assigned 2 or 4 on particular paths |
| 0x141 | 8-bit read/write | Control flag, tested against zero and assigned 0 or 1 |
| 0x143 | 8-bit write | Outcome marker assigned 0, 1, or 2 |

The entry word spans offsets 0x140–0x143. It overlaps the state byte, control flag, byte 0x142, and outcome marker. The represented entry access is a word read, not a byte read.

Selection uses the result of converting that word to a character. Equating the selector with the byte at offset 0x140 requires additional byte-order and conversion assumptions. Character signedness and conversion behavior are not independently established.

### Indirect callbacks

No opaque function identities are supplied for the indirect targets. They are identified by dispatch-table byte offset.

| Dispatch-table offset | Explicit arguments | Result use |
|---|---|---|
| 0x11C | Object address; four-byte value obtained from F-9b9c1c7483f4 | Ignored |
| 0x15C | Object address | Four-byte value forwarded to slot 0x160 |
| 0x160 | Object address; preceding callback result; addresses of first and second derived storage | Tested for zero versus nonzero |

The object’s dispatch-table address is evaluated separately for the two paired callbacks. The callback at slot 0x15C may therefore affect which target is reached through slot 0x160.

### Preconditions and return

Each reached object field, local-storage access, external-data evaluation, and dispatch-table access requires valid storage and whatever alignment the target environment requires. Reached indirect targets must be callable, and dependency arguments must satisfy those dependencies’ contracts.

The function performs no visible null validation, bounds validation, or local exception recovery. Invalid storage, invalid callbacks, nonreturning dependencies, and exceptional exits have no established normal-return result.

On every explicitly represented normal-return path, the return value is 1.

## Behavior

### Entry selection

The initial word read and character conversion select one of three paths:

| Converted selector | Action |
|---|---|
| 2 | Write outcome 1, then return 1 without dependency calls |
| 3, 4, 5, or 6 | Call F-979606fd565e with the object and 1; then write outcome 1 and return 1 |
| Any other value, including 0 and 1 | Enter resource processing |

Selection occurs once. Later writes to the overlapping bytes do not restart dispatch during this invocation.

All subsequent ordering descriptions assume the reached calls return normally. A dependency may mutate the object, dispatch table, external data, or temporary storage unless its separate contract establishes otherwise.

### Preliminary condition

Resource processing begins by reading the resource field and calling F-779577644ecc with that value.

- A zero result bypasses the preliminary control-flag read and proceeds to ordinary processing.
- A nonzero result causes the control flag to be read.
- A nonzero control flag also proceeds directly to ordinary processing.
- Only a nonzero query result combined with a zero control flag invokes F-7e4f952e879e, followed by F-0edfbf2c458b.

Those last two dependencies receive no explicit arguments. The result of F-7e4f952e879e is unused. The zero/nonzero result of F-0edfbf2c458b selects the next path.

### Alternate transition

A zero result from F-0edfbf2c458b selects this sequence:

1. Write state byte 4.
2. Call F-9b9c1c7483f4 with an alternate destination address, an embedded-data address, and a separate auxiliary address. Both local areas are four bytes.
3. Pass its four-byte result, with the object, to the callback at dispatch slot 0x11C.
4. After that callback returns, pass the alternate destination address to F-8a5aadac202f.
5. After F-8a5aadac202f returns, evaluate the separately referenced external data value and pass it, with the object, to F-979606fd565e.
6. After that call returns, write outcome 2 and return 1.

The external expression’s **type, memory-read width, and argument conversion are unresolved**. Its evaluation occurs after F-8a5aadac202f returns, so it must not be treated as an entry-time snapshot. It is not established to be a constant, a four-byte load, or an argument passed without conversion.

This path does not initialize ordinary working storage, perform ordinary retrieval, derive the ordinary pair, or invoke ordinary finalization.

### Preliminary continuation

A nonzero result from F-0edfbf2c458b allows ordinary processing to continue, but first:

- Read the resource field again and call F-40aaef3d8070 with that value and 0.
- After the call returns, assign control byte 1.

The resource argument is not required to equal the value used by the preliminary query. The control assignment follows the call and therefore overwrites any control-byte change made by that call.

### Ordinary initialization and first retrieval

Ordinary processing uses three distinct four-byte local areas: working storage, first derived storage, and second derived storage.

First, F-8c34796da81c receives the working-storage address and two zero arguments. Then F-6f25363c06e5 receives a newly read resource value, selector 0, and the working-storage address.

If selector-0 retrieval returns zero:

- F-979606fd565e receives the object and decimal 150.
- Outcome 1 is written after that call.
- Working storage is passed to F-b058bb03c6c9.
- The function returns 1.

Neither derived area is produced or finalized on this path.

If selector-0 retrieval returns nonzero, F-b0d980eac3b2 receives the first derived-storage address and working-storage address.

### Conditional second retrieval

Only after the first derivation returns does the function read the control flag.

A nonzero flag suppresses selector-1 retrieval. A zero flag causes F-6f25363c06e5 to receive a freshly read resource value, selector 1, and the same working-storage address.

If this selector-1 retrieval returns zero:

- Outcome 1 is written **before** calling F-979606fd565e with the object and 150.
- First derived storage is passed to F-b058bb03c6c9.
- Working storage is passed to F-b058bb03c6c9.
- The function returns 1.

Second derived storage is neither produced nor finalized.

If selector-1 retrieval succeeds, or is suppressed, F-b0d980eac3b2 receives the second derived-storage address and working-storage address.

Suppression means that no retrieval occurs between the two derivations. It does not establish equal derived values, unchanged working storage, independent copies, or shared ownership.

### Paired callbacks

After both derived areas have been produced:

1. The callback at slot 0x15C receives the object.
2. The callback at slot 0x160 receives the object, the preceding callback’s four-byte result, and the first and second derived-storage addresses, in that argument order.

The dispatch-table address is read separately for each callback. There is no established requirement that the second callback use the table or target present before the first callback.

A zero result from slot 0x160 selects rejection:

- Write outcome 1.
- Call F-979606fd565e with the object and 150.
- Perform ordinary finalization.

This branch makes no direct state-byte or control-byte assignment.

A nonzero result selects acceptance:

- Write state byte 2.
- Read the resource field anew and call F-779577644ecc.
- If the query result is nonzero, perform no direct control read or write in this acceptance subpath.
- If the query result is zero, read the control flag. A zero flag causes F-2690d8d63608 to receive another newly read resource value and 1. Regardless of whether that call occurs, assign control byte 0.
- Write outcome 0.
- Perform ordinary finalization.

The acceptance-time query is independent of the preliminary query. Neither its argument nor its result is required to match the earlier one.

### Ordinary finalization

After either paired-callback outcome, F-b058bb03c6c9 receives:

1. Second derived storage.
2. First derived storage.
3. Working storage.

The function then returns 1.

“Finalization” names an observed call position, not a proven destruction or memory-release operation.

## Side effects

### Direct writes and visibility

Every explicit object write is eight bits wide. There is no direct write to byte 0x142 or to the resource field.

| Path | Direct-write ordering |
|---|---|
| Selector 2 | Outcome 1 |
| Selectors 3–6 | F-979606fd565e, then outcome 1 |
| Alternate transition | State 4 before transition dependencies; outcome 2 after F-979606fd565e |
| Preliminary continuation | F-40aaef3d8070, then control 1 |
| Selector-0 failure | F-979606fd565e, then outcome 1, then working finalization |
| Selector-1 failure | Outcome 1 before F-979606fd565e and finalization |
| Callback rejection | Outcome 1 before F-979606fd565e and finalization |
| Callback acceptance | State 2, conditional control processing, outcome 0, then finalization |

These are direct-write guarantees, not unconditional final-object guarantees. Ordinary finalization occurs after the outcome write, and those dependencies are not known to preserve object fields.

Conversely, the outcome writes on entry-only paths and the alternate transition occur after the last represented dependency call on their respective paths.

Resource values are read at their individual call sites. Control values are read at branch-specific points, sometimes after dependencies capable of mutation. No invocation-wide snapshot is established.

### Local storage and ownership

Five distinct four-byte local areas are represented: three for ordinary processing and two for the alternate transition. Their addresses escape to dependencies during calls.

No explicit allocation or deallocation primitive is visible in this function. Dependencies may allocate, retain, alias, release, or transform resources; none of those effects follows solely from the call names or local-storage sizes.

The alternate auxiliary area has no visible initialization before its address is supplied. That does not prove an uninitialized read: the dependency may write it or use only its address. Likewise, the source does not establish what initialization guarantees any helper provides.

Retention of local addresses beyond the invocation’s lifetime is not established as safe.

### Widths, arithmetic, and failure handling

The represented operations use fixed offsets, word-to-character conversion, and zero/nonzero decisions. There is no data-dependent arithmetic on retrieved values or callback results.

Known literal arguments include 0, 1, and decimal 150. The alternate-transition external argument is separately evaluated; its type, read width, and conversion at the call boundary remain unresolved. The address model does not resolve those properties.

The representation does not establish a general order among independent argument evaluations within a single call. The sequencing between completed calls and subsequent direct writes is established.

There is no visible retry loop or differentiated error return. Retrieval failure and callback rejection are represented normal outcomes. Exceptions, traps, nonreturn, and dependency contract violations have no specified recovery path here.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-779577644ecc | Receives the current resource value; its zero/nonzero result controls preliminary and accepted-pair branches |
| F-7e4f952e879e | Called without explicit arguments before F-0edfbf2c458b; result unused |
| F-0edfbf2c458b | Called without explicit arguments; zero selects the alternate transition |
| F-40aaef3d8070 | Receives current resource and 0 before control is directly assigned 1 |
| F-8c34796da81c | Receives working-storage address and two zeros before ordinary retrieval |
| F-6f25363c06e5 | Receives current resource, selector 0 or 1, and working-storage address; result tested against zero |
| F-b0d980eac3b2 | Receives a derived-storage destination and working-storage address |
| F-2690d8d63608 | Receives current resource and 1 on one accepted-pair subpath |
| F-b058bb03c6c9 | Receives reached ordinary local areas in the specified finalization order |
| F-9b9c1c7483f4 | Receives alternate destination, embedded-data address, and auxiliary address; supplies a four-byte callback argument |
| F-8a5aadac202f | Receives alternate destination after the slot-0x11C callback returns |
| F-979606fd565e | Receives the object and either literal 1, literal 150, or the separately evaluated external value whose type, read width, and argument conversion are unresolved |

Neither these dependencies nor the indirect callbacks are assumed to preserve globals, object fields, dispatch entries, temporary contents, or ownership.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Where an example describes a resulting field value, assume intervening dependencies do not independently modify that field unless stated otherwise.

1. **Selector-only completion.** If the entry word converts to character value 2, write outcome 1 and return 1 without dependency calls. Merely observing byte 2 at offset 0x140 does not establish the conversion premise without representation assumptions.

2. **Entry-triggered call.** Converted selector 5 calls F-979606fd565e with 1 before writing outcome 1. The direct write overwrites any outcome change made by that call.

3. **First retrieval failure.** A processing selector, zero preliminary query, and zero selector-0 retrieval result produce ordinary initialization, retrieval, F-979606fd565e with 150, outcome 1, and working finalization. The preliminary control read, derived areas, and paired callbacks are absent.

4. **Alternate transition.** A nonzero preliminary query, zero control flag, and zero F-0edfbf2c458b result produce state 4, alternate preparation, slot 0x11C, alternate finalization, external-value evaluation and F-979606fd565e, then outcome 2. No ordinary retrieval occurs.

5. **External-value timing.** If F-8a5aadac202f changes the external data that supplies the alternate argument, the argument expression is evaluated after that change. Its exact passed value remains conditional on the unresolved data type and argument conversion; no four-byte-read requirement follows.

6. **Preliminary continuation.** Under the same preliminary conditions but a nonzero F-0edfbf2c458b result, F-40aaef3d8070 receives a fresh resource value and 0 before control becomes 1. If subsequent dependencies leave control unchanged, selector-1 retrieval is suppressed.

7. **Second retrieval suppression.** Nonzero control after the first derivation suppresses only selector-1 retrieval, not second derivation or the paired callbacks. Equality of derived values is not required.

8. **Second retrieval failure.** Zero control and failed selector-1 retrieval produce outcome 1 before F-979606fd565e with 150. Finalization covers first derived storage and working storage, but not second derived storage.

9. **Rejected pair.** A zero slot-0x160 result produces outcome 1 before F-979606fd565e with 150. Finalization follows in second-derived, first-derived, working order.

10. **Accepted pair, zero later query.** State 2 precedes the query. Zero control invokes F-2690d8d63608 with a fresh resource value and 1; nonzero control omits that call. Both alternatives directly assign control 0, then outcome 0, before finalization.

11. **Accepted pair, nonzero later query.** State 2 and outcome 0 are directly written without a control read or write in that subpath. This does not guarantee which selector a later invocation obtains.

12. **Conditional byte correspondence.** If little-endian layout and low-eight-bit character conversion are independently established, byte 2 at offset 0x140 selects 2 regardless of the other three bytes. The represented entry access nevertheless remains 32 bits.

## Uncertainties

The following remain **documented_only**:

- Target byte order, character signedness, and exact word-to-character conversion behavior.
- The external data expression’s declared type, memory-read width, resulting value, and argument conversion when passed to F-979606fd565e.
- The second-parameter type of F-979606fd565e; literal arguments do not establish that parameter’s prototype.
- Embedded-data contents and their required representation.
- Domain meanings of state, control, outcome, resource, and numeric arguments.
- Whether state writes are intended to determine a subsequent invocation’s entry path.
- Complete object type, allocation extent, alignment rules, concurrency, and reentrancy requirements.
- Whether the resource word represents an address, handle, or another identifier.
- Temporary-storage semantics, aliasing, initialization guarantees, and ownership.
- Dependency behavior involving mutation, retention, failure, exceptions, or nonreturn.
- ABI details beyond the represented access widths and invocation shapes.

No unresolved property is replaced by an assumed preservation, allocation, or release rule.

## Evidence

**Established static evidence:** The supplied representation establishes the listed object accesses, word-to-character entry selection, selector partition, conditional calls, callback argument relationships, direct byte-write ordering, reached-local finalization order, and four-byte normal return value 1. The supplied mapping establishes the opaque dependency identities.

For the alternate-transition external argument, static evidence establishes evaluation and passage to F-979606fd565e after F-8a5aadac202f returns. It does **not** establish the external data’s type, read width, or call-boundary conversion.

**Inferred intent:** Resource retrieval, paired-value presentation, notification, and transition terminology describe plausible roles. They do not establish domain state names, timing units, ownership rules, or dependency internals.

**Runtime evidence:** No runtime tests have been performed. All acceptance examples are conditional static criteria, not observed executions.
