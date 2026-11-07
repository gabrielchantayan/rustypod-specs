<!-- Trusted provenance: F-cfbcb8a0551d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cfbcb8a0551d

## Purpose

F-cfbcb8a0551d classifies an input record and either:

- Returns success immediately.
- Conditionally invokes a no-argument dependency and then returns success.
- Dispatches the record through a callback selected from a context-associated table, optionally invoking a second callback when the first returns zero.

The established behavior is numeric classification and callback dispatch. Interpreting the record as an event, the context as an object, or the conditional dependency as initialization is inferred intent, not an established semantic requirement.

The function is not a Boolean normalizer. Although several paths return exactly 1 or 0, callback results can be returned unchanged, including negative values.

## Interface

### Inputs

The function receives two arguments:

1. **Context:** a pointer to integer-addressable storage. Its first field supplies the base address of a callback table.
2. **Record:** an integer-valued address used to read fields at fixed byte offsets.

The supplied representation uses integer fields and integer-valued addresses in a layout consistent with 32-bit words and pointers. This contract describes that apparent 32-bit model. The excerpt alone does not independently establish the target ABI, integer size, pointer size, alignment rules, or byte order.

Offsets below are byte offsets. The context offsets derived from indexed integer accesses assume four-byte integers.

| Structure | Offset | Role |
|---|---:|---|
| Context | 0 | Callback-table base address |
| Context | 80 | Value compared with the record payload on the conditional secondary-dispatch path |
| Context | 184 | Integer slot whose conversion to a character controls the no-argument dependency |
| Record | 4 | Numeric record kind |
| Record | 8 | Nested-record address for kind 22, or comparison payload for kind 15 |
| Nested record | 4 | Numeric subtype inspected for kind 22 |
| Callback table | 224 | Secondary callback address |
| Callback table | 268 | Primary callback address selected for kind 22 with nested subtype 42 |
| Callback table | 272 | Primary callback address selected for other dispatched records |

The callback-table offsets are independent byte offsets in the supplied representation; they are not indices into a demonstrated array.

### Output

The return value is integer-valued:

- Exactly 1 for kinds 6, 7, 8, 13, and 21, provided execution completes.
- The selected primary callback’s result when that result is nonzero.
- The secondary callback’s result when that callback is invoked and returns nonzero.
- Exactly 0 when primary dispatch returns zero and no nonzero secondary result is obtained.

No separate error output, error object, or explicit exception-handling interface is shown.

### Preconditions

The caller must provide readable storage for every field reached on the selected path, and callable addresses for every callback invoked.

These requirements are path-dependent:

- Every invocation reads the record kind.
- Kind 21 does not require a context read in the shown body.
- Kinds 6, 7, 8, and 13 require the context slot at offset 184.
- Primary dispatch requires the context’s table address and the selected callback entry.
- A dispatched kind 22 requires a readable nested-record address and nested subtype field.
- The conditional secondary path can additionally require the record payload, context comparison field, and secondary callback entry.

There are no explicit null checks, bounds checks, address validation, or callback-presence checks. Invalid storage or callback addresses have no defined recovery behavior in this excerpt.

## Behavior

### 1. Initial classification

The function first reads the record kind at offset 4.

The initial value determines one of three routes:

| Initial kind | Route |
|---:|---|
| 6, 7, 8, 13 | Conditional no-argument dependency, then return 1 |
| 21 | Return 1 immediately |
| Any other value | Primary callback dispatch |

The ordering comparisons used to reach these routes do not establish a valid-kind range. Values outside the listed set, including negative values under the apparent signed-integer interpretation, reach primary dispatch.

### 2. Conditional no-argument dependency

For initial kinds 6, 7, 8, and 13, the function reads the context’s integer slot at offset 184 and converts that value to a character.

If the converted value is zero, it invokes F-bb78617ea748 with no explicit arguments. Otherwise, it does not invoke that dependency.

After either outcome, the function returns exactly 1. It does not inspect a result from F-bb78617ea748 and does not re-read the controlling context slot.

On the apparent eight-bit-character model, this condition examines the low eight bits of the integer value. Consequently, a nonzero full-width slot can still trigger the dependency if those bits are zero. The excerpt does not establish whether the actual machine performs a narrow load or a full-width load followed by conversion.

If the dependency does not return normally, the final return of 1 is not reached.

### 3. Primary callback selection

For all initially dispatched kinds, the function reads the record kind at offset 4 again. It does not reuse the initial classification value for callback selection.

If this second value is 22, the function:

1. Reads the nested-record address from record offset 8.
2. Reads the nested subtype at offset 4 from that address.
3. Selects callback-table offset 268 if the subtype equals 42.

For every other selection outcome, it selects callback-table offset 272. This includes:

- A second kind value other than 22.
- Kind 22 with a nested subtype other than 42.

It then obtains the table base through context offset 0, reads the selected callback address, and invokes that callback with the original context and record arguments, in that order.

The callback receives the original record address, not the nested-record address.

Because selection uses a fresh kind read, an initially dispatched record whose kind changes before selection is classified using the new value for callback selection. However, selection does not restart initial classification: a new value of 6, for example, still selects the ordinary primary callback rather than taking the conditional-dependency route.

### 4. Nonzero primary result

If the primary callback returns any nonzero integer, the function returns that value unchanged.

No subsequent record-kind read, payload comparison, or secondary callback invocation occurs on this path.

### 5. Zero primary result and conditional secondary dispatch

If the primary callback returns zero, the function reads the record kind again.

When this post-callback kind is not 15, the function returns exactly 0.

When it is 15, the function:

1. Reads the payload at record offset 8.
2. Reads the context comparison value at offset 80.
3. Compares those values for equality.

If they differ, it returns exactly 0.

If they match, it obtains the callback-table base again from context offset 0, reads the callback address at table offset 224, and invokes that callback with the original context and record arguments.

A nonzero secondary result is returned unchanged. A zero secondary result produces a return value of 0. Thus, when invoked and returning normally, the secondary callback determines the final numeric result without Boolean normalization.

These checks use state visible after the primary callback. The primary callback may change the record kind, record payload, context comparison field, table address, or callback entries; the function does not preserve earlier versions for this decision.

In particular, secondary dispatch uses a newly read table address. It need not use the table that supplied the primary callback.

## Side effects

The shown body contains no explicit stores to the context, record, nested record, or callback table.

Its direct observable actions are reads and calls. Under ordinary sequential execution with stable memory, their relevant ordering is:

- Initial record-kind read precedes all route-specific activity.
- On the conditional-dependency route, the context-slot read precedes the optional call.
- On the primary-dispatch route, the second kind read precedes any nested-record reads and callback lookup.
- Primary callback completion precedes the post-callback kind read.
- On a qualifying secondary path, the payload read precedes the context comparison-field read, and both precede secondary callback lookup and invocation.

The source representation does not establish instruction-level memory-access widths, barriers, atomicity, volatile semantics, or synchronization guarantees. This ordering describes the represented evaluation and call boundaries, not a concurrency protocol.

At most two calls are made directly by this function during a normal invocation:

- One call to F-bb78617ea748 on its route; or
- One primary callback, possibly followed by one secondary callback.

There is no loop or explicit retry.

No allocation, ownership transfer, retention, or release operation is directly shown. That does not establish that dependencies are side-effect-free or that they preserve ownership. Their memory-management behavior is unknown.

If a callback releases or invalidates storage that the function subsequently reads, the excerpt provides no protection against that invalidation. In particular, a primary callback returning zero must leave any subsequently accessed context and record storage usable for defined execution.

## Dependencies

### F-bb78617ea748

This is the only explicit fixed-target dependency call shown.

- **Condition:** initial kind is 6, 7, 8, or 13, and the context slot’s character conversion is zero.
- **Arguments:** none explicitly supplied.
- **Result handling:** no result is consumed.
- **Following behavior:** return 1 if the dependency returns normally.
- **Unknowns:** global effects, memory effects, ownership behavior, failure behavior, and whether it changes the controlling context slot.

Describing it as an initializer would be inferred intent only.

### Indirect callbacks

Three callback-table locations participate:

- Offset 268: primary callback for selection kind 22 and nested subtype 42.
- Offset 272: primary callback for other selection outcomes.
- Offset 224: conditional secondary callback after a zero primary result.

Each invoked callback receives the context and record arguments and supplies an integer-valued result as consumed by this function. No independent opaque function identity is provided for these dynamic targets.

Their implementations, contracts, ownership effects, and global-state effects are unknown.

### F-d5dd567991e2

The supplied mapping associates this opaque ID with a name that appears as an internal control-flow label in the excerpt. There is no demonstrated call to that function identity.

The label’s presence is not evidence of an additional dependency invocation.

## Acceptance examples

These are conditional acceptance cases derived from static evidence, not executed tests. Unless stated otherwise, memory is stable, accesses are valid, and dependencies return normally.

1. **Kind 21:** The function returns 1. It performs no context-field read or callback invocation in the shown body.

2. **Kind 8 with a nonzero converted flag:** If converting the context slot at offset 184 to a character yields a nonzero value, the function returns 1 without calling F-bb78617ea748.

3. **Kind 13 with a zero converted flag:** The function calls F-bb78617ea748 once and then returns 1. No callback-table lookup occurs.

4. **Nonzero slot with zero low byte:** Under the eight-bit-character model, a slot value of 256 converts to zero for this check. For initial kind 6, this triggers F-bb78617ea748 despite the full integer being nonzero.

5. **Special primary selection:** With initial and selection kind 22 and nested subtype 42, the callback comes from offset 268. If it returns −7, the function returns −7 and performs no secondary check.

6. **Other nested subtype:** With selection kind 22 and nested subtype 41, the primary callback comes from offset 272. The record payload is dereferenced as a nested address during selection, not treated merely as an opaque comparison value.

7. **Zero result without kind 15:** If the primary callback returns zero and the subsequent kind read yields 14, the function returns 0 without reading the comparison payload or invoking the secondary callback.

8. **Matching secondary condition:** If the primary callback returns zero, the subsequent kind is 15, and record offset 8 equals context offset 80, the callback at table offset 224 is invoked. A result of 9 becomes the function’s result.

9. **Mismatching secondary condition:** Under the same conditions except unequal comparison values, the function returns 0 without secondary callback lookup.

10. **Callback-mutated state:** An ordinary primary callback may change the record kind to 15, set matching comparison values, and replace the context’s table address before returning zero. The subsequent secondary decision uses that changed state and reads offset 224 from the replacement table.

11. **Zero secondary result:** When the secondary callback is invoked and returns zero, the function returns 0; it does not attempt another callback.

## Uncertainties

The following are **documented_only**:

- Exact ABI, pointer width, integer width, character width, byte order, and alignment requirements.
- Semantic meanings of the numeric kinds, nested subtype 42, and context fields.
- Identity and complete contracts of the indirect callbacks.
- Whether F-bb78617ea748 initializes anything or changes the slot controlling its invocation.
- Whether dependencies mutate globals, retain pointers, transfer ownership, allocate memory, or release memory.
- Behavior following invalid addresses, unavailable callbacks, nonlocal control transfer, or dependency failure.
- Supported concurrency model and whether external mutation between reads is valid.
- Instruction-level load widths and compiler transformations.

No arithmetic overflow case is introduced by numeric accumulation or multiplication in the shown behavior. The relevant calculations are fixed-offset field accesses. Their validity depends on the target address model and valid backing storage; address wrapping is not established as supported behavior.

## Evidence

The contract is based solely on the supplied static function excerpt and dependency mapping.

Established evidence includes the numeric branch conditions, field offsets, repeated reads, callback selection, callback argument order, return-value propagation, conditional fixed-target call, and absence of explicit stores in the shown body.

Inferred intent is limited to optional interpretations such as event dispatch or initialization; those interpretations do not define acceptance behavior.

No implementation access, runtime execution, smoke run, or tests were performed. The acceptance examples are static predictions conditional on the stated preconditions, not observed results.
