<!-- Trusted provenance: F-cee38ec3663f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cee38ec3663f

## Purpose

F-cee38ec3663f selects one of three execution paths:

- **Primary:** the first status query returns nonzero.
- **Secondary:** the first status query returns zero and the second returns nonzero.
- **Object-local:** both status queries return zero.

The primary and secondary paths obtain a receiver from a dependency, conditionally perform operations on it and a nested object, write one byte of receiver state, and ordinarily reach a shared final command sequence. The secondary path also prepares a cache and processes an auxiliary object; it can return before the receiver commands.

The object-local path uses the supplied object, applies several gates, and may invoke one of two action slots. It does not reach the shared final sequence.

These path names describe control flow, not domain semantics. The evidence does not establish what subsystem the routine controls.

## Interface

### Entry, return, and validity

The routine has one explicit input: an object pointer. No return value is established.

The input object is directly accessed only on the object-local path. This does not establish that a null or otherwise invalid input is supported on the other paths: the requirements imposed by dependencies and shared state are unknown.

All reached object accesses and indirect dispatches require valid memory, suitable object layouts, and compatible targets. There are no visible null checks, dispatch-table checks, or recovery operations.

Zero-valued results select branches or cause returns. They are not independently established as error codes.

### Representation and layouts

The supplied representation uses four-byte integers and four-byte pointer-bearing fields. Offsets and dispatch slots below are byte offsets. A dispatch slot identifies an entry relative to the relevant dispatch-table pointer.

| Object role | Offset | Established use |
|---|---:|---|
| Input object | `0x00` | Dispatch-table pointer |
| Input object | `0x60` | Interior address passed to F-b7abcf60c990 |
| Primary or secondary receiver | `0x00` | Dispatch-table pointer |
| Retained receiver | `0x20` | Four-byte integer passed to F-9c5c2db853f7 on a secondary subpath |
| Retained receiver | `0x2C` | **Four-byte integer read for the secondary gate, followed by conversion to char and comparison with zero; separately, the starting address of explicit one-byte stores** |
| Retained receiver | `0x2D` | One-byte value passed to nested-object slot `0x30` |
| Retained receiver | `0x88` | Four-byte pointer to a nested object |
| Retained receiver | `0x8C` | Interior destination address passed to F-438680ffebfe |
| Retained receiver | `0x94` | Four-byte cache field, zero-tested and conditionally assigned |

The secondary gate is not represented as an explicit one-byte load at `0x2C`. Its established operation is a four-byte integer load, conversion of that integer to char, and a zero comparison of the converted result. Identifying this with a test of the byte at the lowest address requires target-layout and conversion assumptions not established here.

The stores at `0x2C`, by contrast, are explicitly one byte wide. They do not directly overwrite the neighboring byte at `0x2D` or the remaining bytes of the four-byte gate input.

### Opaque operands

External operands are named by role:

- **Cache key**.
- **Primary family**, **primary alternate family**, **primary payloads 1–5**, and **primary descriptor**.
- **Secondary family**, **secondary alternate family**, **secondary payloads 1–5**, and **secondary descriptor**.
- **Final family** and **final payloads 1–3**.

Distinct labels identify distinct source operands, not necessarily unequal values. Family values retained by the routine occupy four bytes. Descriptor arguments are addresses; their contents, extent, and ownership are unknown. No operand is established as a string, command identifier, allocation, or resource handle.

## Behavior

### Path selection

Selection begins with this observable call order:

1. F-93eb550dc612 is called; its result is discarded.
2. F-f2db0ea1d7cc is called.
3. A nonzero result selects the primary path.
4. Only if that result is zero, F-93eb550dc612 is called again and its result discarded.
5. F-cf9ff6ecb14d is then called.
6. Its nonzero result selects the secondary path; zero selects the object-local path.

The discarded-result calls remain part of the behavior. Nothing establishes that they are side-effect-free.

The first selector has precedence: its nonzero result prevents both the second discarded-result call and the second selector call.

### Object-local path

F-b7abcf60c990 receives the interior address at input-object offset `0x60`. This routine does not first load the contents at that address as the argument.

If the dependency returns zero, execution returns without input-object dispatch.

Otherwise:

1. Input-object slot `0x1A4` is invoked with the input object as receiver.
2. A nonzero result ends this path.
3. For a zero result, input-object slot `0x190` is invoked with the same receiver.
4. A zero result selects slot `0xE8`; a nonzero result selects slot `0xE4`.
5. Only the selected action target is invoked, with the input object as receiver.

No explicit input-object write is shown. The invoked dependencies may modify it.

The final action is represented as an indirect invocation with unresolved control-transfer mechanics. The chosen slot and receiver are established; ordinary call-versus-tail-transfer behavior is not.

### Secondary path: cache and auxiliary object

F-93eb550dc612 supplies a receiver pointer that is retained for this path’s receiver operations.

The routine reads the four-byte cache at `0x94`. If it is nonzero, no direct cache assignment occurs. If it is zero:

1. F-938c950e5046 supplies another object.
2. That object’s slot `0xF8` is invoked with the object and cache key.
3. The returned four-byte value is stored at the retained receiver’s `0x94`.

There is no success check between production and assignment. A zero result is stored as zero; the routine does not substitute another value or return solely because cache production returned zero.

Next, F-87be4aa684d1 supplies an auxiliary object. Its slot `0x190` is invoked with that object as receiver.

- If the result is nonzero, auxiliary slot `0xEC` is invoked.
- Otherwise, auxiliary slot `0x194` is queried. A nonzero result invokes slot `0xEC`; zero skips it.

The second auxiliary query is short-circuited when the first returns nonzero. These operations finish before the secondary state gate is read.

### Secondary path: state gate and conditional operations

The routine reads the **four-byte integer at `0x2C`**, converts that value to char, and compares the converted result with zero.

- If the converted result is zero, the routine returns.
- Otherwise, it stores the one-byte value zero at address `0x2C` before querying the nested object.

This contract does not replace that conversion-based gate with a byte-load condition. In particular, the loaded integer occupies storage that also includes offset `0x2D`; whether those other bytes affect the converted result depends on conversion semantics not supplied here.

The routine then resolves slot `0x20` through the nested-object pointer at `0x88` and invokes the target. No explicit argument is shown for this query. An explicit nested receiver argument must not be assumed.

After the query, the secondary family value is loaded and retained.

If the query returns nonzero, the following operations occur in order:

1. Nested-object slot `0x24` receives the nested object and integer `1`.
2. Nested-object slot `0x30` receives the nested object and the one-byte value read at retained-receiver offset `0x2D`.
3. F-9c5c2db853f7 receives the retained receiver, the four-byte value read at its offset `0x20`, and integer `1`.
4. F-438680ffebfe receives the interior address at receiver offset `0x8C` and the secondary descriptor address.
5. Receiver slot `0x58` receives the receiver, retained secondary family, and secondary payload 1.
6. The secondary alternate-family value is captured.
7. Receiver slot `0x58` receives the receiver, the secondary alternate-family operand read for this invocation, and secondary payload 2.
8. Receiver slot `0x58` receives the receiver, the captured alternate-family value, and secondary payload 3.
9. F-6ac70711bbb5 is called.
10. F-7027ea1c0db7 is called.

Whether the nested query returns zero or nonzero, execution then invokes receiver slot `0x58` twice: first with the retained secondary family and secondary payload 4, then with the same retained family and secondary payload 5.

F-ac356e5603c7 then supplies the receiver for the common final sequence.

There is no later direct state write on this path. A zero nested-query result does not undo the preceding one-byte clear.

### Primary path

F-93eb550dc612 supplies the receiver retained for this path.

The routine resolves and invokes nested-object slot `0x20` through the pointer at receiver offset `0x88`. As on the secondary path, the representation shows no explicit query arguments.

The primary family value is loaded and retained after the query.

If the query returns nonzero:

1. F-29eca01f3b88 receives the retained receiver.
2. Nested-object slot `0x24` receives the nested object and integer `0`.
3. F-438680ffebfe receives the interior address at receiver offset `0x8C` and the primary descriptor address.
4. Receiver slot `0x58` receives the receiver, retained primary family, and primary payload 1.
5. The primary alternate-family value is captured.
6. Receiver slot `0x58` receives the receiver, the primary alternate-family operand read for this invocation, and primary payload 2.
7. Receiver slot `0x58` receives the receiver, the captured alternate-family value, and primary payload 3.
8. F-6ac70711bbb5 is called.
9. F-93bd7d506529 is called.

Regardless of the query result, provided execution reaches the continuation:

1. The one-byte value `1` is stored at receiver offset `0x2C`.
2. Receiver slot `0x58` is invoked with the retained primary family and primary payload 4.
3. The same slot is invoked with the retained primary family and primary payload 5.
4. F-ac356e5603c7 supplies the common-final receiver.

The state assignment follows the entire conditional sequence. A conditional dependency that does not return prevents this assignment from being reached.

### Common final sequence

The receiver is the pointer returned by F-ac356e5603c7. It need not equal the earlier receiver.

The final family value is captured. Three invocations through this receiver’s slot `0x58` follow, using final payloads 1, 2, and 3 in order.

The first invocation uses the final-family operand read for that invocation. The second and third use the previously captured family value. The representation therefore distinguishes capture from the first invocation’s operand read, without establishing an intervening call between them.

The first invocation also shows a slot-`0x58` target value and four incoming register-derived values as additional operands. Whether these are genuine interface arguments is unresolved. Their application-level meanings are unknown.

The third invocation has unresolved control-transfer mechanics. Its receiver, slot, ordinary operands, and ordering are established; an ordinary return to this routine is not independently established.

## Side effects

### Direct writes and overlapping access widths

The visible direct writes are:

| Condition | Direct write |
|---|---|
| Secondary cache read is zero | Four-byte cache result at `0x94` |
| Secondary converted gate result is nonzero | One-byte zero at `0x2C` |
| Primary continuation is reached | One-byte `1` at `0x2C` |

The secondary gate’s four-byte read and subsequent one-byte store have different widths. A one-byte store does not establish that the whole four-byte integer becomes zero or one. Nor does it establish the converted result of a future gate without target conversion rules and assumptions about intervening mutation.

These are direct-write facts, not final-state guarantees. Later dependencies may overwrite the same storage.

### Ordering and retained values

Observable ordering includes:

- Cache preparation and auxiliary-object operations before the secondary four-byte gate read.
- Secondary one-byte clear before the nested query.
- Primary one-byte assignment after conditional primary operations.
- Reads at `0x2D` and `0x20` at their respective later operations, not as entry snapshots.
- Branch-family capture after the nested query and before conditional work.

Later nested-object accesses and receiver dispatches resolve their relevant pointer fields and targets again. A retained outer receiver pointer does not establish that its nested pointer or dispatch table remains unchanged.

Captured family values remain available for their stated later uses even if a dependency changes the backing external storage. Payload operands are not established as entry-time snapshots. No within-invocation evaluation order beyond the supplied representation is required here.

### Ownership, errors, and arithmetic

No explicit allocation, deallocation, retain, release, lock, or unlock is identifiable. Accessor results and cache assignment do not establish ownership transfer. Passing an interior or descriptor address does not establish whether a dependency copies, retains, or modifies the referenced storage.

There is no visible rollback or error-recovery sequence. If a dependency fails or does not return, earlier effects may remain.

Address calculations use fixed offsets. The four-byte element spacing yields the interior offsets `0x60`, `0x8C`, and `0x94`; the state location is `0x2C`, and its adjacent byte is `0x2D`. No application-level length arithmetic or overflow policy is shown. Valid address formation and accessible storage are preconditions, not checked outcomes.

## Dependencies

| Opaque ID | Established role |
|---|---|
| F-93eb550dc612 | Selection-stage calls and primary/secondary receiver acquisition |
| F-f2db0ea1d7cc | First zero/nonzero selector |
| F-cf9ff6ecb14d | Second selector, reached only after a zero first result |
| F-b7abcf60c990 | Object-local gate receiving the input’s `0x60` interior address |
| F-938c950e5046 | Supplies the cache-producing dispatch receiver |
| F-87be4aa684d1 | Supplies the auxiliary object |
| F-9c5c2db853f7 | Receives the secondary receiver, its `0x20` value, and `1` |
| F-438680ffebfe | Receives a `0x8C` interior address and branch-specific descriptor address |
| F-29eca01f3b88 | Primary conditional operation before nested slot `0x24` |
| F-6ac70711bbb5 | Conditional operation after receiver payloads 1–3 |
| F-7027ea1c0db7 | Secondary operation following F-6ac70711bbb5 |
| F-93bd7d506529 | Primary operation following F-6ac70711bbb5 |
| F-ac356e5603c7 | Supplies the common-final receiver |

Indirect dependencies are identified by receiver role and byte slot because no opaque IDs for their targets are supplied. None is assumed to preserve globals, object fields, ownership, or unrelated memory.

## Acceptance examples

These are conditional criteria, not executed tests. Reached calls are assumed to return normally unless stated otherwise.

1. **Primary precedence:** A nonzero first-selector result prevents the second selector. With a zero nested-query result, primary conditional operations are skipped, but the one-byte assignment, payloads 4–5, and final sequence remain required.

2. **Object-local rejection:** With both selectors zero and F-b7abcf60c990 returning zero, no input-object dispatch follows.

3. **Object-local action:** With a nonzero initial gate and zero slot-`0x1A4` result, slot `0x190` selects exactly one action: zero selects `0xE8`; nonzero selects `0xE4`. A nonzero slot-`0x1A4` result skips both the query and action.

4. **Secondary converted-zero exit:** If the four-byte value loaded at `0x2C` converts to char zero, the routine returns after cache and auxiliary work. No state clear, nested query, or final sequence follows. A zero byte at address `0x2C` alone is not an unconditional substitute for this premise.

5. **Qualified byte interpretation:** If independently established target layout and conversion rules make the gate equivalent to testing the byte at `0x2C`, a zero byte predicts the preceding exit. Without those additional facts, this byte-based prediction is not required behavior.

6. **Secondary nested rejection:** If the converted gate result is nonzero and the nested query returns zero, a one-byte zero store precedes the query. Payloads 1–3 and their conditional dependencies are skipped; payloads 4–5 and the final sequence still occur.

7. **Zero cache production:** A zero cache result is stored as a four-byte zero without an immediate fallback. A later invocation repeats production only if its then-current cache read is zero.

8. **Auxiliary short-circuit:** A nonzero auxiliary slot-`0x190` result invokes `0xEC` without querying `0x194`. Two zero query results skip `0xEC`.

9. **Write-width boundary:** The routine’s one-byte state stores do not directly overwrite `0x2D`. This does not guarantee that dependencies preserve that byte or that a subsequent four-byte gate read has a particular converted result.

10. **Nonreturning primary operation:** If conditional primary work does not return, the later one-byte assignment is not reached. Earlier effects have no promised rollback.

## Uncertainties

The following remain **documented_only**:

- Target byte order, char conversion semantics, signedness, and any resulting equivalence between the four-byte gate and a particular stored byte.
- Domain meanings of selectors, state storage, cache values, descriptors, and payloads.
- Exact external operand values and their mutability.
- The nested query’s calling convention and implicit machine-state inputs.
- Whether the first final invocation’s extra operands are genuine arguments.
- Call-versus-tail-transfer details at unresolved indirect transfers.
- Dependency mutation, ownership, retention, failure, and synchronization behavior.
- Aliasing among input, retained, nested, auxiliary, and final objects.
- External invariants maintaining object validity across calls.

Coordinated state switching is a plausible interpretation, not an established requirement.

## Evidence

Established static evidence comprises branch precedence, zero/nonzero decisions, dependency order, dispatch-slot selection, fixed offsets, the cache’s four-byte accesses, the secondary gate’s **four-byte integer read followed by char conversion**, and the explicitly one-byte state stores.

The gate is not established as an explicit byte read. Any equivalent byte-level interpretation requires additional layout and conversion evidence.

Calling-convention and indirect-control-transfer ambiguities remain unresolved. No runtime tests have been performed. Acceptance examples express conditional obligations derived from the supplied static representation, not measured execution.
