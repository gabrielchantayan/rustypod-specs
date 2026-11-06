<!-- Trusted provenance: F-ab86ba0d462f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ab86ba0d462f

## Purpose

F-ab86ba0d462f conditionally consumes an input record or routes it through callbacks associated with a context object. Certain record values trigger a preliminary dependency call or an immediate successful return. Otherwise, a record-dependent callback is invoked. If that callback returns zero, a further callback may run when the record identifies an object currently referenced by the context.

The established behavior is routing, conditional invocation, and propagation of callback results. Interpreting the record as an event, a nonzero result as “handled,” or the context field as a selected object is inferred intent, not an established semantic guarantee.

The function itself contains no explicit memory stores. Its dependencies can nevertheless change any accessible state, including fields that this function subsequently reads.

## Interface

### Inputs

The function takes two arguments:

1. **Context address.** An address of a structure whose first field supplies a dispatch-table address. A further field at byte offset +0x50 participates in a conditional comparison.
2. **Record address.** An address of a structure containing a discriminator at byte offset +0x04 and additional fields used on particular paths.

Both arguments are forwarded, in their original order and without replacement, to every dependency invoked by this function.

The supplied representation uses integer-sized values and integer-carried addresses. The observable layout is consistent with 32-bit words and 32-bit addresses. The contracts below describe that represented layout; they do not establish a portable ABI for targets with different integer or pointer widths.

### Return value

The represented return value is a 32-bit integer:

- A nonzero result from an invoked dependency is returned unchanged.
- One special record condition returns exactly 1 without invoking a dependency.
- All paths that complete without either condition return exactly 0.

There is no normalization of arbitrary nonzero dependency results to 1. Negative dependency results, if supplied, are also nonzero and are propagated unchanged.

### Accessed layout

All offsets below are byte offsets.

| Object | Offset | Represented field | Access condition |
|---|---:|---|---|
| Context | +0x00 | Dispatch-table address | When a callback is about to be selected or invoked |
| Context | +0x50 | 32-bit comparison value | After a zero primary callback result, when the reread record discriminator is 0x0F |
| Record | +0x04 | 32-bit discriminator | At entry, during primary routing, and after a zero primary callback result |
| Record | +0x08 | Nested-object address | During primary routing when the discriminator is 0x16 |
| Record | +0x08 | 32-bit comparison value | After a zero primary callback result when the discriminator is 0x0F |
| Record | +0x0C | 32-bit special-case value | During the initial special-case processing when the entry discriminator is 0x13 |
| Nested object | +0x04 | 32-bit nested discriminator | After reading the nested-object address for discriminator 0x16 |
| Dispatch table | +0x10C | Callable address | For the distinguished nested-record route |
| Dispatch table | +0x110 | Callable address | For all other primary routes |
| Dispatch table | +0x0E0 | Callable address | For the conditional additional invocation |

The two interpretations of record offset +0x08 are selected by the discriminator at the relevant stage. The function does not validate either interpretation.

### Preconditions

For normal completion, every address actually dereferenced on the chosen path must be readable with the represented width and suitable alignment for the execution environment. Every selected callable address must support the two-argument invocation and integer return convention.

These requirements are path-dependent. For example:

- The immediate-return special case need not access the context at all.
- A nested-object address is not dereferenced unless the primary-routing discriminator is 0x16.
- The context field at +0x50 is not read unless the final-stage discriminator is 0x0F.

No null checks, bounds checks, callable validation, ownership checks, or recoverable invalid-address handling are visible.

## Behavior

### 1. Initial special-case processing

The function first reads the record discriminator at +0x04.

If that value is not 0x13, it does not read the record field at +0x0C during this stage and proceeds to primary routing.

If the entry discriminator is 0x13, it reads the 32-bit field at record offset +0x0C and tests whether subtracting 0x7F82 produces zero.

For the ordinary 32-bit interpretation, this recognizes the value 0x7F82.

If that first test succeeds:

1. Invoke F-7cdcd05ba671 with the context address and record address.
2. If the dependency returns nonzero, return that exact value immediately.
3. If the dependency returns zero, proceed to primary routing.

A zero return from F-7cdcd05ba671 does **not** cause the alternative special-case check described below to run. This remains true if the dependency changes the record field at +0x0C.

If the first test does not succeed, while the entry discriminator was 0x13:

1. Read the record field at +0x0C again.
2. Test whether subtracting 0x7F00 produces 0x81.
3. If so, return exactly 1 immediately.
4. Otherwise, proceed to primary routing.

For the ordinary 32-bit interpretation, the second test recognizes 0x7F81.

The second field read is a distinct access. The function does not reuse the first +0x0C value for this test. The entry discriminator is not reread between these two special-case checks.

Consequently, where external mutation or access-sensitive memory is meaningful in the surrounding environment, the second test can observe a different +0x0C value from the first. Such an environment is not itself established by the supplied evidence.

### 2. Primary callback selection

On reaching primary routing, the function reads the record discriminator at +0x04 again. This read occurs after any zero-returning invocation of F-7cdcd05ba671.

If this newly read discriminator is 0x16:

1. Read the nested-object address from record offset +0x08.
2. Read the nested discriminator from that object's offset +0x04.
3. Select dispatch-table slot +0x10C if the nested discriminator is 0x2A.
4. Otherwise, select dispatch-table slot +0x110.

If the newly read record discriminator is anything other than 0x16, select dispatch-table slot +0x110 without reading the nested object.

After determining the route, the function reads the current dispatch-table address from context offset +0x00 and loads the callable address from the selected slot. It invokes that callback with the original context and record addresses.

If the callback returns nonzero, the function returns that exact result immediately. No final-stage discriminator or comparison-field reads occur on this path.

If the callback returns zero, processing continues.

Selection of slot +0x10C does not bypass the later conditional stage. Both primary routes have the same continuation rule.

### 3. Conditional additional invocation

After a zero result from the primary callback, the function reads the record discriminator at +0x04 again.

If the discriminator is not 0x0F, return 0.

If it is 0x0F:

1. Read the 32-bit value at record offset +0x08.
2. Read the 32-bit value at context offset +0x50.
3. Compare those values for exact equality.
4. If they differ, return 0.
5. If they match, read the current dispatch-table address from context offset +0x00.
6. Load and invoke the callable at dispatch-table offset +0x0E0, passing the original context and record addresses.
7. Return its result if nonzero; otherwise return 0.

Thus, whenever this additional callback is invoked and returns normally, its result is also the final result, including when it is zero.

The equality check is a raw represented-value comparison. No dereference of either comparison value occurs here, and no pointer-identity or domain-specific meaning is established.

### 4. Mutation-sensitive sequencing

Reads after a dependency call use the then-current memory contents:

- F-7cdcd05ba671 can affect primary routing by changing the record, a nested object, or the context's dispatch-table address.
- The primary callback can affect final-stage eligibility by changing the record discriminator, record offset +0x08, or context offset +0x50.
- The primary callback can change which dispatch table supplies the callback at +0x0E0.

The function retains the original argument values, not immutable snapshots of the objects they address.

Normal completion therefore requires dependencies to leave later-used addresses valid, but this is a precondition rather than a guarantee made by this function. The supplied evidence does not establish that any dependency preserves globals, object lifetimes, dispatch-table contents, or ownership.

## Side effects

### Direct effects

The function directly performs:

- Reads from the record.
- Conditional reads from the context and a nested object.
- Reads of callback addresses from a dispatch table.
- Up to three dependency invocations.
- A return to its caller.

No explicit allocation, deallocation, reference-count operation, lock operation, global assignment, or structure-field write appears in the supplied body.

### Dependency effects

The possible invocation order is:

1. F-7cdcd05ba671, only for the first special-case match.
2. Exactly one primary callback, unless an earlier return occurs.
3. The additional callback, only after a zero primary result and a successful final-stage comparison.

Three calls are possible in one execution if the preliminary dependency returns zero, the primary callback returns zero, and the final-stage condition matches.

All effects of these calls are otherwise unknown. They may modify memory or external state. Nothing in the function proves that they neither allocate nor release memory.

### Ownership and lifetime

The function does not visibly transfer ownership through an explicit ownership operation. It passes both supplied addresses to each invoked dependency.

Whether those invocations borrow, retain, consume, or release any referenced object is undocumented. In particular, a dependency that invalidates an address needed by subsequent processing can prevent normal completion.

### Errors and abnormal completion

There is no distinct error-result convention beyond propagation of nonzero dependency results. Whether a particular nonzero value denotes success, consumption, failure, or another condition depends on contracts not provided here.

Invalid data addresses or callable addresses are not converted into a defined return value. Faults, exceptions, nonlocal exits, and nonreturning dependencies are outside the established normal-return behavior.

## Dependencies

### F-7cdcd05ba671

Established contract at this call site:

- Called only when the entry discriminator is 0x13 and the first special-value test succeeds.
- Receives the original context and record addresses, in that order.
- Produces an integer result.
- A nonzero result terminates processing and is propagated unchanged.
- A zero result permits primary routing using subsequent memory reads.

Its broader input requirements, side effects, ownership behavior, and meaning of return values are unknown.

### Indirect dependencies

Three callback locations are established: dispatch-table offsets +0x10C, +0x110, and +0x0E0.

No opaque function IDs for their runtime targets were supplied. Assigning identities or treating these slots as known functions would invent evidence. Their contracts here are limited to the invocation conditions, arguments, result handling, and access order described above.

The slots are not proven to contain distinct callable addresses.

## Acceptance examples

These examples are conditional contracts, not reports of executed tests. Unless stated otherwise, assume valid accessible memory, returning dependencies, and no relevant mutation during a described sequence.

1. **Preliminary dependency terminates processing.**  
   Entry discriminator 0x13 and special field 0x7F82 cause F-7cdcd05ba671 to run. If it returns −7, the function returns −7. Neither primary slot is invoked.

2. **Preliminary dependency permits routing.**  
   With the same initial values, suppose F-7cdcd05ba671 returns zero after changing the discriminator to 0x16 and establishing a nested discriminator of 0x2A. Primary routing uses slot +0x10C, not a route derived from the entry discriminator.

3. **Immediate special-case result.**  
   Entry discriminator 0x13 and both special-field reads yielding 0x7F81 cause an immediate return of 1. No dependency is invoked, and the context is not dereferenced.

4. **Distinct special-field reads.**  
   If the first special-field read yields a value other than 0x7F82 and the second yields 0x7F81, the immediate return is still 1. This example applies only where such differing observations are permitted by the surrounding execution environment.

5. **No alternative special check after the preliminary call.**  
   If the first special-field read matches 0x7F82, F-7cdcd05ba671 changes that field to 0x7F81, and the dependency returns zero, processing proceeds to primary routing. The changed field does not trigger the immediate-return branch.

6. **Distinguished nested route.**  
   A primary-routing discriminator of 0x16 and nested discriminator 0x2A select slot +0x10C. Any other nested discriminator selects +0x110.

7. **No nested access on other routes.**  
   A primary-routing discriminator other than 0x16 selects +0x110 without dereferencing the record's +0x08 value as a nested-object address.

8. **Primary result takes precedence.**  
   If the selected primary callback returns 23, the result is 23 even if the record would otherwise qualify for the additional callback. The final-stage reads do not occur.

9. **Post-callback state controls the additional call.**  
   If the primary callback returns zero and leaves the record discriminator as 0x0F with record +0x08 equal to context +0x50, slot +0x0E0 is invoked. If that callback returns −2, the final result is −2.

10. **Comparison failure ends with zero.**  
    If the primary callback returns zero and the final discriminator is 0x0F but the comparison values differ, the additional callback is not invoked and the result is 0.

11. **Dispatch-table replacement is observed.**  
    If the primary callback returns zero, replaces the context's dispatch-table address, and leaves the final comparison eligible, the additional callback is loaded from the replacement table.

## Uncertainties

- **documented_only — Domain meaning:** The meanings of discriminator values, special values, callback slots, and the context comparison field are not established.
- **documented_only — Concrete ABI:** The supplied representation supports a 32-bit layout interpretation but does not independently specify compiler, alignment rules, endianness, or a cross-platform calling convention.
- **documented_only — Arithmetic semantics:** The displayed special tests use subtraction. Under 32-bit modular arithmetic, they uniquely recognize 0x7F82 and 0x7F81 respectively. The supplied material does not independently establish source-language overflow rules; no additional overflow error behavior is claimed.
- **documented_only — Dependency contracts:** Preservation of memory, globals, ownership, and object lifetimes is unknown.
- **documented_only — Concurrent or access-sensitive memory:** Distinct reads are visible in the supplied representation, but concurrency guarantees and volatile or device-memory semantics are not provided.
- **documented_only — Invalid inputs:** No defined recovery behavior for invalid addresses, malformed objects, or invalid callback entries is established.

## Evidence

Established static evidence consists solely of the supplied function body and the supplied opaque mapping for F-7cdcd05ba671. The access offsets, branch conditions, invocation order, rereads, and return propagation above follow from that material.

“Event,” “handled,” and similar domain interpretations are inferred intent only. Callback identities beyond the supplied mapping remain unknown.

No runtime tests have been performed. The acceptance examples specify expected behavior under their stated conditions; they are not execution evidence.
