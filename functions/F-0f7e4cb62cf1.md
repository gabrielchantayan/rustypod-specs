<!-- Trusted provenance: F-0f7e4cb62cf1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0f7e4cb62cf1

## Purpose

F-0f7e4cb62cf1 conditionally dispatches an event through an object’s indirect-call table. If that dispatch does not return a nonzero result, the function applies a separate fallback for events whose current type is `0x13`.

The fallback recognizes two conditions, in priority order:

1. A pair of event fields matches an object-associated value and either another object-associated value or a global alternate value.
2. An enabled secondary condition matches an object-associated value and a separate global value.

A successful fallback invokes an indirect operation and returns `1`. An unmatched event returns `0`.

These are structural descriptions of the established behavior. The meanings of the event types, classification values, object fields, and global values are not established. Terms such as “event,” “object,” and “classification” identify their roles in this contract, not a verified application-level interpretation.

## Interface

### Inputs

The function receives two arguments:

| Argument | Role | Required accessible storage |
|---|---|---|
| Object reference | Receiver for indirect operations and source of comparison state | A table reference at byte offset `0x00`; additional fields as required by the executed path |
| Event reference | Input to classification and indirect dispatch; source of fallback fields | A 32-bit field at byte offset `0x04`; fields at `0x08` and `0x0C` when the fallback reaches them |

The event reference is represented as an integer in the supplied function representation but is used as a memory address. This contract treats it as an address-bearing argument, not as an event identifier.

All offsets below are byte offsets.

### Object layout used by this function

| Offset | Accessed content |
|---|---|
| `0x00` | Reference to an indirect-call table |
| `0x40` | 32-bit primary comparison value |
| `0x44` | 32-bit primary comparison value |
| `0x98` | 32-bit secondary comparison value |
| `0x9C` | 32-bit secondary enable field; any nonzero value enables consideration |

Only these fields are established as directly accessed. Their names, larger enclosing layout, and ownership are unknown.

### Event layout used by this function

| Offset | Accessed content |
|---|---|
| `0x04` | 32-bit event type |
| `0x08` | 32-bit comparison field |
| `0x0C` | 32-bit comparison field |

No direct access to event offset `0x00` is established.

### Indirect-call table layout

The object’s table provides callable entries at these offsets:

- Event-dispatch entries: `0x114`, `0x118`, `0x11C`, `0x120`, `0x124`, `0x128`, `0x12C`, `0x130`, `0x134`, `0x138`, and `0x13C`.
- Primary fallback operation: `0xD4`.
- Secondary fallback operation: `0xA0`.

Event-dispatch entries receive the original object and event arguments and supply a 32-bit result.

The primary fallback entry receives the original object argument. Its result, if any, is ignored.

The secondary fallback entry receives the original object argument and a one-byte value returned by F-5710a8628cba. Its result, if any, is ignored. The machine-level extension and argument-passing convention for that byte are not established by the supplied evidence.

### Output

The function returns a 32-bit integer:

- A selected event-dispatch operation’s exact nonzero result, without normalization.
- `1` after either successful fallback operation returns.
- `0` if no operation reports or establishes handling.

Negative nonzero dispatch results are propagated just like positive nonzero results.

### Preconditions and failure boundary

Every read and indirect call reached during execution requires valid, suitably accessible storage and a callable target with the appropriate calling convention.

There is no explicit validation for null references, invalid table entries, inaccessible event storage, or invalid object storage. No defined error result is established for such cases. Faults or nonlocal control transfers from dependencies are not converted into `0` or `1`.

## Behavior

### 1. Ordered event dispatch

The function evaluates the following candidates in order. A classification comparison invokes F-468ee00d626a with the event reference only when that candidate’s event-type comparison succeeds.

| Priority | Required event type | Required classification result | Selected table offset |
|---:|---:|---:|---:|
| 1 | `0x06` | `0x6F` | `0x114` |
| 2 | `0x0D` | `0x6F` | `0x118` |
| 3 | `0x08` | `0x6F` | `0x11C` |
| 4 | `0x0D` | `0x59` | `0x128` |
| 5 | `0x06` | `0x59` | `0x12C` |
| 6 | `0x0D` | `0x5A` | `0x120` |
| 7 | `0x06` | `0x5A` | `0x124` |
| 8 | `0x08` | `0x59` | `0x130` |
| 9 | `0x15` | Not consulted | `0x134` |
| 10 | `0x0D`, otherwise `0x07` | `0x57` | `0x138` |
| 11 | `0x0D`, otherwise `0x07` | `0x58` | `0x13C` |

The first successful candidate selects the dispatch operation. No later candidate is evaluated after selection.

For each of the final two candidates, the `0x0D` alternative is evaluated first. If that complete alternative fails, the `0x07` alternative is evaluated. Failure includes either a type mismatch or a classification mismatch.

The table is not a license to replace the evaluation with a single lookup over cached inputs:

- Each candidate reads the event type afresh.
- Each applicable classification comparison makes its own call to F-468ee00d626a.
- A classification result is not reused by a later candidate.
- The `0x15` candidate does not itself call the classifier, but earlier candidates may already have done so if event state changed during classification.

For a stable event type and a classifier that always returns a nonmatching value, type `0x0D` causes five classification calls, type `0x06` causes three, and types `0x08` and `0x07` each cause two. These counts are conditional on stability; they are not guarantees for dependencies that mutate event state.

Once a candidate succeeds, the function reads the object’s table reference, reads the selected callable entry, and invokes it with the original arguments. Table selection therefore occurs after the classification work that led to the match.

If this call returns nonzero, that value is returned immediately. If it returns zero, processing continues to the fallback. If no candidate succeeds, processing also continues to the fallback.

At most one event-dispatch operation is called per invocation.

### 2. Fallback eligibility

On reaching the fallback, the function reads the event type again.

If its current value is not `0x13`, the function returns `0`.

This read is independent of the earlier dispatch comparisons. A dispatch operation that returns zero may have changed the event type, and its post-call value determines fallback eligibility. The function does not require the event’s original type to have been `0x13`.

### 3. Primary fallback condition

For a current type of `0x13`, the function reads the event field at `0x0C` and retains that value for fallback comparisons.

It first compares that retained value with the object field at `0x44`.

If they differ, the primary condition fails without reading the event field at `0x08`, the object field at `0x40`, or the primary alternate global for this condition.

If they match, the function:

1. Reads the event field at `0x08`.
2. Reads the object field at `0x40`.
3. Accepts equality between those two values.
4. Only when they differ, reads a 32-bit primary alternate global and accepts equality between the event value and that global value.

Thus, the global alternate is a fallback comparison value, not an unconditional read. It is not read when the object-associated value already matches.

If the primary condition succeeds, the function obtains the callable entry at table offset `0xD4` from the object’s table and invokes it with the object argument. After that operation returns, this function returns `1`.

The secondary condition is not evaluated after a successful primary condition.

### 4. Secondary fallback condition

If the primary condition fails, the function reads the object field at `0x9C`.

- If that field is zero, it returns `0`.
- If it is nonzero, it compares the retained event value from offset `0x0C` with the object field at `0x98`.

The event field at `0x0C` is not reread for this comparison.

If those values differ, the function returns `0` without reading the secondary comparison global or making the secondary fallback call.

If they match, the function reads the event field at `0x08`, then reads a separate 32-bit secondary comparison global. These values must be equal.

On equality:

1. F-5710a8628cba is called without explicit arguments.
2. Its one-byte result is retained.
3. The object’s current table reference and entry at offset `0xA0` are obtained.
4. That entry is invoked with the object argument and the retained byte.
5. The function returns `1` after the indirect operation returns.

The table lookup occurs after F-5710a8628cba returns. No assumption that this dependency preserves the object’s table reference is valid.

If the final equality fails, the function returns `0`.

## Side effects

### Direct effects

No direct writes to the object, event, global comparison values, or other externally visible storage are established.

The function directly performs:

- Repeated event-type reads.
- Conditional object-field and event-field reads.
- Conditional reads of two distinct globals.
- Indirect-call table reads.
- Calls to classification, byte-producing, and dynamically selected operations.

The order and conditional occurrence of these operations are part of the behavior. In particular, repeated classification calls cannot be collapsed merely because stable inputs would yield the same result.

### Dependency effects

The effects of dependencies are not supplied. They may affect state later read by this function.

Consequently:

- Classification cannot be assumed pure.
- A dispatch operation returning zero cannot be assumed to leave the event unchanged.
- F-5710a8628cba cannot be assumed to preserve global or object state.
- Indirect operations cannot be assumed to allocate, retain, release, or leave memory untouched.

This function establishes no explicit ownership transfer, allocation, release, or reference-count update. That absence does not establish equivalent behavior for callees.

All later accesses still require valid storage. If a dependency invalidates an object or event that this function subsequently accesses, no safe recovery behavior is established.

### Arithmetic and comparisons

The established decisions use equality, inequality, and zero/nonzero tests over 32-bit values. There is no application-level arithmetic transformation of comparison fields.

Offsets locate fields and table entries. Signed ordering, overflow handling, saturation, and range validation do not participate in the established decisions.

The `0x9C` enable field is not restricted to `1`: every nonzero bit pattern passes its enable check.

## Dependencies

### F-468ee00d626a

Receives the event reference and supplies a 32-bit classification result.

This function compares that result with `0x6F`, `0x59`, `0x5A`, `0x57`, or `0x58`, depending on the candidate being evaluated.

Its classification meaning, side effects, ownership behavior, failure behavior, and stability across repeated calls are **documented_only** unknowns.

### F-5710a8628cba

Receives no explicit arguments and supplies a one-byte value for the secondary fallback operation.

It is called only after all secondary fallback comparisons succeed. Its semantic purpose and effects are **documented_only** unknowns. No relationship between its result and the secondary comparison global is established.

### Indirect dependencies

The callable identities behind the table offsets are unresolved. No opaque function IDs for these targets were supplied, so this contract does not invent identities or conflate entries.

Their established call roles are:

| Table offsets | Arguments | Treatment of result |
|---|---|---|
| `0x114`–`0x13C`, limited to the listed entries | Object reference, event reference | Nonzero 32-bit result propagated; zero permits fallback |
| `0xD4` | Object reference | Ignored; enclosing function returns `1` |
| `0xA0` | Object reference, one-byte dependency result | Ignored; enclosing function returns `1` |

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not reports of executed tests. Unless stated otherwise, referenced storage remains valid and dependencies return normally.

1. **First dispatch match**  
   Given a stable type of `0x06` and a first classification result of `0x6F`, the entry at `0x114` is invoked. If it returns `-3`, the function returns `-3` without running fallback comparisons.

2. **Repeated classification matters**  
   Given a stable type of `0x06`, classification results of `0x00` and then `0x59` cause the first applicable candidate to fail and the entry at `0x12C` to be invoked. Reusing the first classification result would violate the contract.

3. **Type-only dispatch**  
   Given a type that remains `0x15` throughout candidate evaluation, the entry at `0x134` is invoked without any classification call. If it returns zero and the type remains `0x15`, the final result is zero.

4. **Dispatch can enable fallback**  
   Given a selected dispatch operation that changes the event type to `0x13` and returns zero, the function proceeds with fallback comparisons using the post-call event state.

5. **Primary object-value match avoids the alternate global**  
   Given fallback type `0x13`, equality between event `0x0C` and object `0x44`, and equality between event `0x08` and object `0x40`, the primary alternate global is not read. The entry at `0xD4` is invoked, followed by a return of `1`.

6. **Primary global alternative**  
   Given the same first equality, but event `0x08` differs from object `0x40` and equals the primary alternate global, the primary fallback still succeeds.

7. **Secondary enable is nonzero, not exactly one**  
   Given a failed primary condition, object `0x9C` equal to `0x80000000`, matching event `0x0C` and object `0x98`, and matching event `0x08` and the secondary global, the secondary fallback is enabled. The byte-producing dependency and then entry `0xA0` are called; the result is `1`.

8. **Secondary byte is forwarded without a zero-value rejection**  
   Under successful secondary comparisons, a byte result of `0x00` from F-5710a8628cba is passed to the indirect operation. It does not cause this function to return zero.

9. **Post-dependency table selection**  
   If F-5710a8628cba changes the object’s table reference while leaving the object valid, the secondary operation is selected from the updated table.

10. **Primary priority**  
    If both fallback conditions would match, only the primary operation is invoked. The secondary enable field and secondary comparison global need not be read.

## Uncertainties

The following remain **documented_only**:

- Semantic meanings of type codes, classification codes, comparison fields, and globals.
- Concrete identities and full contracts of indirect callees.
- Ownership, lifetime, mutation, and failure behavior of every dependency.
- Complete object, event, and table layouts.
- Exact target ABI, pointer width, byte order, alignment rules, and byte-argument extension rules.
- Synchronization requirements and behavior under concurrent mutation.
- Whether callers interpret zero as “unhandled” and nonzero as “handled.” That interpretation is inferred intent; the established behavior is the return-value rule itself.

The supplied representation supports 32-bit integer accesses and a one-byte dependency result. Its indexed object accesses are consistent with four-byte integer elements. A likely 32-bit address model is an inference, not an independently verified ABI guarantee.

## Evidence

This contract is based solely on static inspection of the supplied representation of F-0f7e4cb62cf1 and the supplied mappings for F-468ee00d626a and F-5710a8628cba.

Established evidence includes branch ordering, short-circuit classification calls, field and table offsets, comparison widths, return propagation, fallback priority, conditional global reads, absence of direct external writes, and lookup of the secondary indirect target after the byte-producing dependency returns.

Application-level intent and dependency internals are not established. No runtime tests, call traces, binary-layout checks, or execution-based validation have been performed.
