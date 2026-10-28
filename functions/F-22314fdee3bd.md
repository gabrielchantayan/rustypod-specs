<!-- Trusted provenance: F-22314fdee3bd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-22314fdee3bd

## Purpose

F-22314fdee3bd performs a conditional operation on a caller-supplied state object and returns a 32-bit result restricted to zero or one on normal completion.

The established behavior is:

- Return zero immediately if a pointer-like field at offset `0x48` is zero.
- Otherwise obtain a 32-bit unsigned baseline value through F-8ce70de0f02c.
- Read a selector at offset `0x30`.
- If the selector is zero, invoke F-f58767a63d87 and return one.
- If the selector is nonzero, invoke an indirectly selected callback with the current object reference and the value one, obtain another value through F-8ce70de0f02c, and return one only if the second value is strictly greater than the baseline under unsigned comparison.

A possible interpretation is that this operation initiates or services work and, on one path, checks whether some measure advanced. That interpretation is **inferred intent**, not an established semantic meaning for the fields, dependencies, or result. In particular, a return value of one is not uniformly evidence of progress: the zero-selector path returns one without a second measurement.

## Interface

### Input

The function accepts one argument representing the base address of a state object.

The supplied representation treats this argument as an integer used for address calculations. Its semantic role is an address, not an arbitrary numeric identifier. No null check, allocation check, object-size check, or alignment check is visible.

The following offsets are relative to that base:

| Offset | Access and role | Width or extent |
|---|---|---|
| `0x14` | Address passed to F-f58767a63d87 on the zero-selector path | No direct read or write at this offset by F-22314fdee3bd; dependency access extent is unknown |
| `0x30` | Selector, read after the first call to F-8ce70de0f02c | 32-bit integer |
| `0x48` | Initial presence guard and, on the nonzero-selector path, reference used to locate and invoke a callback | Integer/pointer-sized field in the supplied 32-bit representation |
| `0xC4` | Address passed to F-8ce70de0f02c | No direct read or write at this offset by F-22314fdee3bd; dependency access extent is unknown |

On the nonzero-selector path, the reference at offset `0x48` is interpreted as pointing to an object whose first pointer-sized field identifies a dispatch table. A callable entry is obtained at byte offset `0x10` within that table.

The dispatch layout is a structural fact of the supplied representation. A particular source-language object model or class identity is not established.

### Output

The result occupies 32 bits and is always either:

- **Zero:** the initial guard was zero, or the nonzero-selector path produced a second unsigned value less than or equal to the first.
- **One:** F-f58767a63d87 returned normally on the zero-selector path, or the second unsigned value was strictly greater than the first on the nonzero-selector path.

These meanings apply only to normal returns. Invalid memory access, a dependency that does not return, or an abnormal exit from the callback can prevent any result from being produced.

### Widths and comparison rules

The two results from F-8ce70de0f02c are held and compared as 32-bit unsigned quantities.

The comparison is a direct unsigned ordering comparison. It is not:

- A signed comparison.
- A subtraction-based elapsed-value calculation.
- A wraparound-aware sequence comparison.
- A test merely for inequality.

The guard and selector are tested only for zero versus nonzero. Their sign does not alter the branch decision.

The callback receives the object reference and an additional argument whose value is one. The representation supplies that argument as an integer literal; the callback’s independently declared parameter type and calling convention are not available.

## Behavior

### Initial guard

The first visible object-content access reads the field at offset `0x48`.

If that field is zero, the function returns zero immediately. On this path it does not:

- Call either named dependency.
- Read the selector at offset `0x30`.
- Traverse the dispatch structure.
- Invoke the callback.

Calculating an interior address is not needed on this early-return path. The input must nevertheless permit the initial field read; a null input address is not itself handled as a benign “absent object.”

If the field is nonzero, processing continues. This initial check does not establish that later uses of the field will observe the same value.

### Baseline acquisition

F-8ce70de0f02c is called with the address at base plus `0xC4`.

Its returned 32-bit unsigned value becomes the baseline for this invocation. This call occurs on every path that passes the initial guard, including the path that subsequently invokes F-f58767a63d87 and never compares the baseline.

The selector at offset `0x30` is read **after** this call returns. Consequently, a mutation of the selector by F-8ce70de0f02c can affect the branch taken. A contract that treats the selector as captured at function entry would not match the visible ordering.

There is no established guarantee that F-8ce70de0f02c leaves the state object, its referenced objects, or global state unchanged.

### Zero-selector path

If the selector read after baseline acquisition is zero:

1. F-f58767a63d87 is invoked with the address at base plus `0x14`.
2. If that call returns normally, F-22314fdee3bd returns one.

There is no second call to F-8ce70de0f02c on this path. The baseline does not influence the result.

No return value from F-f58767a63d87 is consulted. Its side effects or internal outcome therefore do not select between zero and one in this function. A normal return from that dependency is sufficient for the visible function to return one.

### Nonzero-selector path

If the selector read after baseline acquisition is nonzero, the function invokes an indirect callback.

The callback selection uses the reference held at offset `0x48`, the dispatch-table reference at the start of the referenced object, and the callable entry at table offset `0x10`. The callback is supplied with an object reference loaded from offset `0x48` and the value one.

The initial nonzero guard is not repeated immediately before dispatch. F-8ce70de0f02c has already executed between the guard and callback selection. If it changed the field to zero or made the dispatch structure invalid, the visible function contains no recovery check.

The supplied expression contains accesses to offset `0x48` for both dispatch lookup and the callback argument. It does not justify a stronger guarantee about the relative order of those accesses or their atomic consistency. In a stable state they refer to the same object; behavior under concurrent mutation is not established.

After the callback returns normally, F-8ce70de0f02c is called again with the address at base plus `0xC4`.

The function then compares the second returned value with the saved baseline:

- Second value less than or equal to baseline: return zero.
- Second value strictly greater than baseline: return one.

The callback’s return value, if any, is not consulted. The function does not reread the selector after the callback, restart branch selection, or retry the measurement.

### Arithmetic and boundary behavior

The offsets identify byte displacements. No bounds checks or checked-address arithmetic are visible. Valid address formation and valid memory access are caller/environment preconditions; the evidence does not establish portable behavior for overflowing address calculations.

The unsigned comparison has important boundaries:

- A baseline of zero requires a second value of at least one to return one on the nonzero-selector path.
- Equal values return zero.
- A baseline of `0xFFFFFFFF` cannot be exceeded by another 32-bit unsigned value, so the nonzero-selector path returns zero for every possible second value.
- A transition from `0xFFFFFFFF` to zero returns zero, even if an external interpretation would call that a counter wrap and forward progress.

## Side effects

F-22314fdee3bd contains no explicit store to the state object or to global memory in the supplied body.

Its observable effects nevertheless include dependency and callback execution:

| Path | Ordered calls |
|---|---|
| Initial field at `0x48` is zero | None |
| Guard passes; selector is zero | F-8ce70de0f02c, then F-f58767a63d87 |
| Guard passes; selector is nonzero | F-8ce70de0f02c, then indirect callback, then F-8ce70de0f02c |

Calls can mutate memory, affect global state, perform external operations, or change resource ownership. None of those effects is ruled out by this body.

No explicit allocation, release, ownership transfer, locking, or synchronization operation is visible. Passing an interior address or object reference does not establish that the callee borrows it, retains it, takes ownership, or releases it.

For normal execution, dependencies must leave sufficiently valid state for subsequent operations. In particular:

- After baseline acquisition, reading the selector must remain valid.
- On the nonzero-selector path, the dispatch object and callable entry must be usable when accessed.
- After the callback, the second measurement call must remain valid.

If a dependency invalidates required memory, the visible function supplies no lifetime repair or error conversion.

## Dependencies

### F-8ce70de0f02c

**Established use:** receives the address at base plus `0xC4`; produces a value consumed as a 32-bit unsigned quantity.

**Call count:** zero when the initial guard fails, one on the zero-selector path, and two on the nonzero-selector path.

**Ordering:** the first call precedes the selector read; the second, when present, follows callback completion.

**Unknown:** the value’s meaning, reads and writes, external effects, error signaling, lifetime effects, and preservation of object or global state.

### F-f58767a63d87

**Established use:** receives the address at base plus `0x14`, only when the post-baseline selector is zero.

**Result handling:** no result is used; normal return is followed by a return value of one.

**Unknown:** the operation performed, its parameter extent, ownership behavior, side effects, and failure modes.

### Indirect callback

**Established use:** selected through the object reference at offset `0x48` and dispatch-table slot at byte offset `0x10`; receives an object reference and the value one.

**Result handling:** no callback result participates in the final comparison.

**Unknown:** callback identity, signature beyond the visible arguments, object lifetime effects, global effects, and exceptional behavior. No opaque function ID for this dynamically selected target is supplied.

## Acceptance examples

These examples are conditional static expectations, not executed tests. Unless otherwise stated, all referenced storage is valid, relevant state remains stable during accesses, and every invoked dependency returns normally.

1. **Absent reference.** The initial field at offset `0x48` is zero. The result is zero, with no dependency or callback calls, regardless of the selector’s contents.

2. **Zero selector ignores baseline magnitude.** The guard passes, the first measurement returns `0xFFFFFFFF`, and the selector subsequently reads as zero. F-f58767a63d87 is called once with the address at base plus `0x14`; the result is one. No second measurement occurs.

3. **Equal measurements.** The selector is nonzero, and both measurements return 42. The callback is invoked between them. The result is zero.

4. **Strict increase.** The selector is nonzero, the first measurement returns 42, and the second returns 43. The result is one.

5. **Decrease.** The selector is nonzero, the first measurement returns 43, and the second returns 42. The result is zero.

6. **Unsigned, not signed, ordering.** The selector is nonzero, the baseline is `0x7FFFFFFF`, and the second value is `0x80000000`. The result is one.

7. **Wrap is not recognized as progress.** The selector is nonzero, the baseline is `0xFFFFFFFF`, and the second value is zero. The result is zero.

8. **Selector mutation during baseline acquisition.** The selector starts nonzero, but the first call to F-8ce70de0f02c changes it to zero. The subsequent branch invokes F-f58767a63d87, not the callback.

9. **Later selector mutation does not reselect the path.** The selector reads as nonzero, then the callback changes it to zero. The function still performs the second measurement and bases its result on the unsigned comparison.

10. **Invalidated dispatch reference.** The initial guard passes, but baseline acquisition clears the field at offset `0x48` while leaving the selector nonzero. No defined normal result is established: the function does not revalidate the reference before dispatch.

## Uncertainties

The following are **documented_only** limitations:

- The semantic identities of the state object, selector, measurement, callback, and interior subobjects are unknown.
- The 32-bit integer widths and pointer-oriented layout are supported by the supplied representation; the original ABI, endianness, and calling conventions are not independently established.
- Memory extents required by the dependencies are unknown.
- Callee preservation of globals, object fields, resources, and ownership is unknown.
- Concurrent modification, atomicity, synchronization requirements, and reentrancy safety are unknown.
- Fault handling, exceptions, nonlocal exits, and dependency-specific error conventions are unknown.
- The relative ordering of accesses within the indirect-call expression is not established beyond their role in preparing that invocation.
- A result of one cannot be assigned a stronger application-level meaning such as successful completion or durable progress from this evidence alone.

## Evidence

The supplied static function body establishes the field offsets, initial guard, dependency call order, post-baseline selector read, dispatch-table displacement, callback arguments, unsigned comparison, and normal return values described above.

The purpose-level interpretation of servicing work or measuring advancement is inferred intent only. Dependency internals and callback semantics were not supplied and are not assumed.

No runtime tests, execution traces, implementation inspection beyond the supplied body, or independent ABI validation have been performed.
