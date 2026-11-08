<!-- Trusted provenance: F-988ccbdda49d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-988ccbdda49d

## Purpose

F-988ccbdda49d selects and performs an ordered sequence of indirect operations. Selection depends on a signed 32-bit selector, a shared threshold, and fixed selector constants. The operations use an object obtained from F-9d0ecc5f6adf.

A recognized selection performs two, three, or four indirect calls and returns the 32-bit value **1**, provided execution completes normally. An unrecognized selection performs no indirect calls and returns **0**. Object acquisition and shared base-value accesses occur even for unrecognized selections.

The evidence establishes dispatch conditions, argument values, and ordering constraints. It does not establish the application-level meaning of the selected operations or whether they succeed.

## Interface

### Arguments

| Argument | Contract established by the supplied representation |
|---|---|
| First argument | An opaque 32-bit value that the function does not explicitly use. |
| Selector | A signed 32-bit integer used in equality and signed ordering comparisons. |
| Forwarded argument | An opaque 32-bit value supplied unchanged as the third argument of every indirect call. |

No explicit validation is performed on these arguments.

The first argument does not affect the visible selection logic. The forwarded argument might represent a scalar, address, or handle; its interpretation and validity requirements are **documented_only** unknowns.

### Result

On normal completion, the result is a 32-bit value:

- **0:** No dispatch sequence was selected.
- **1:** A sequence was selected and all its calls returned normally.

Indirect return values are ignored. A result of 1 does not establish that any dispatched operation achieved its intended effect.

These result guarantees do not define behavior for invalid memory accesses, unresolved arithmetic overflow, or abnormal dependency completion.

### Object and indirect-call layout

F-9d0ecc5f6adf receives no explicit arguments. Its result is treated as an object address.

For each indirect call, the object’s first 32-bit word identifies a dispatch table. The callable target is obtained from the table entry at byte offset **0x54**, equivalent to **84 bytes**. This describes the represented 32-bit layout, not a portable object format or complete calling convention.

Every indirect call receives:

1. The acquired object address.
2. One selected 32-bit dispatch value.
3. The unchanged forwarded argument.

The acquired object address remains the same throughout the invocation. The dispatch target need not remain the same: later calls consult the object’s dispatch structure again.

## Behavior

### Shared-state terminology

The following descriptive labels identify distinct shared storage locations. They do not assign application meanings.

| Label | Use |
|---|---|
| **Base** | A signed 32-bit value accessed twice before selection. |
| **Threshold** | A signed 32-bit value used for selection. |
| **Equality value** | The first dispatch value in the equality sequence. |
| **6004 values 1–4** | Four shared values used by the lower-side 0x6004 selection. |
| **6097 values 1–3** | Three shared values used by the lower-side 0x6097 selection. |
| **60BE values 1–2** | Two shared values used by the lower-side 0x60BE selection. |
| **60F0 values 1–2** | Two shared values used by the upper-side 0x60F0 selection. |

Distinct locations may hold equal contents. The dispatch values are passed as 32-bit arguments; their signedness and application interpretation are not established.

### Acquisition and the two base accesses

Object acquisition occurs before the function accesses Base or evaluates Threshold. Effects of F-9d0ecc5f6adf can therefore influence the shared state subsequently observed.

The representation contains **two ordered accesses to Base**, not one common snapshot:

- The first supplies the **retained base**, used as the final argument value in every base-pair sequence.
- The second supplies the input from which 2 is subtracted, producing the **adjusted base**.

Both resulting values are retained before dispatch selection and before any indirect operation.

If Base is stable across these accesses, the adjusted base equals the retained base minus 2. The function does not itself establish that stability. If the two accesses observe different values, the final pair reflects those different observations.

There is no intervening explicit call between the base accesses. Nevertheless, the supplied evidence does not justify merging them into one snapshot or establishing concurrent-access behavior. Exact machine load counts are also not established.

### Selection and precedence

Equality with Threshold is tested first. If that comparison succeeds, the equality sequence takes precedence over every fixed selector constant.

If equality fails, a later signed comparison with Threshold selects the lower-side or remaining-side family. The representation references Threshold separately for those comparisons; it does not establish a single retained threshold value.

For a Threshold that remains stable during selection, behavior is:

| Selection condition | Dispatch values, in call order | Normal result |
|---|---|---|
| Selector equals Threshold | Equality value; adjusted base; retained base | 1 |
| Selector below Threshold and equal to 0x6004 | 6004 values 1, 2, 3, 4 | 1 |
| Selector below Threshold and equal to 0x6097 | 6097 values 1, 2, 3 | 1 |
| Selector below Threshold and equal to 0x60BD | Adjusted base; retained base | 1 |
| Selector below Threshold and equal to 0x60BE | 60BE values 1, 2 | 1 |
| Selector below Threshold and matching no listed lower-side constant | None | 0 |
| Selector above Threshold and equal to 0x60F0 | 60F0 values 1, 2 | 1 |
| Selector above Threshold and equal to 0x60F1, 0x60F2, or 0x60F3 | Adjusted base; retained base | 1 |
| Selector above Threshold and matching no listed upper-side constant | None | 0 |

The constants are not recognized independently of their comparison family. For example, 0x6004 above a stable Threshold does not select the four-call sequence, and 0x60F0 below it does not select the two-call fixed-value sequence.

Without threshold stability, “above Threshold” is not a complete description of the remaining-side path. That path means the equality comparison failed and the later less-than comparison also failed. The function provides no retry or consistency check if those comparisons observe different threshold values. Concurrent or asynchronous modification semantics remain unresolved.

### Dispatch argument timing

The retained base and adjusted base are determined before dispatch begins. Later changes to Base do not cause either retained value to be refreshed.

The other dispatch values are accessed when their respective calls are reached. They are not collected as one entry-time snapshot. Consequently, an earlier operation can change a shared value that a later operation will receive.

For example, in the 0x6004 sequence, the second shared value is obtained after the first operation has returned. If the first operation changes that location and the change is visible to the later access, the second operation receives the changed value.

The equality value is likewise obtained for the first equality dispatch, rather than as part of the earlier base accesses.

### Target resolution and call order

Calls occur serially in the order shown in the table. A later call is not reached until the preceding call returns.

Each call resolves its target through the acquired object’s dispatch structure. A preceding operation can therefore influence a later target by changing either:

- The object’s first word identifying the table.
- The relevant table entry.

The function does not reacquire the object between calls.

For the final operation of each fixed-value sequence, target resolution precedes the access to that operation’s final shared dispatch value. This ordering is explicit for the 0x6004, 0x6097, 0x60BE, and 0x60F0 sequences.

For base-pair sequences, the final target is resolved after the preceding operation returns; the final argument is the already retained base value.

The evidence does not establish a universal relative order between target lookup and shared-argument access for every other call. Nor does it establish atomicity across the individual memory accesses involved in resolving a target.

### Widths and arithmetic

Selector ordering is signed 32-bit ordering. Treating the selector as unsigned would change behavior for values with the high bit set.

The adjusted base is formed by subtracting 2 from the **second** Base observation. In signed 32-bit arithmetic:

- Inputs from −2,147,483,646 through 2,147,483,647 have representable results.
- Inputs of −2,147,483,648 or −2,147,483,647 have mathematical results below the representable range.

The supplied representation does not establish a wraparound rule, another overflow interpretation, or the behavior of the surrounding function in those two cases. Overflow behavior is **documented_only**.

The subtraction precedes selection, including paths that otherwise return 0 or use only fixed shared dispatch values. An unsupported selector does not establish that this arithmetic is avoided.

## Side effects

### Directly established effects

The function:

- Invokes F-9d0ecc5f6adf once on ordinary entry.
- Accesses Base twice and reads shared selection state.
- Reads the selected shared dispatch values.
- Reads object and table memory when dispatching.
- Makes zero, two, three, or four indirect calls.
- Returns 0 or 1 when execution completes normally under the stated conditions.

No explicit write to shared storage, object fields, or caller-supplied memory appears in the supplied body.

An unrecognized selector avoids indirect dispatch and its associated object/table dereferences. It does not avoid acquisition, the base accesses, or the subtraction.

### Delegated effects and retained state

No dependency is established to preserve globals, object contents, or resources.

F-9d0ecc5f6adf may affect state observed during selection. An indirect operation may affect shared values or dispatch targets used by later operations. These possibilities are not evidence that a particular dependency actually performs such mutations.

Once selection has completed, later threshold changes do not trigger reselection. Once both base-derived values have been retained, later Base changes do not replace them.

There is no explicit retry, rollback, compensation, or status-based early termination.

### Ownership and lifetime

The function performs no explicit allocation, release, retention, or ownership transfer.

The acquisition result’s ownership status is unknown. It may be borrowed, owned, shared, or governed by another contract not supplied here. The absence of a release in this function does not establish that callers or dependencies have no release obligations.

On any reached dispatch path, the acquired object and the table memory must be suitable for each required access, and the selected target must be callable under the actual ABI. Earlier operations must leave subsequent required accesses and calls viable if normal completion is expected.

Forwarding the third argument does not establish whether a callee may retain it, modify memory it denotes, or release an associated resource.

### Errors and abnormal completion

There are no explicit null checks, layout checks, callable-target checks, error handlers, or cleanup paths.

If acquisition does not return normally, the function does not proceed to selection. If an indirect operation does not return normally, later operations and the final return are not established.

A failure-like value returned normally by an indirect operation is ignored. Earlier completed effects are not explicitly undone after a later failure.

## Dependencies

### F-9d0ecc5f6adf

This is the only dependency with a supplied opaque identity.

Its established relationship to F-988ccbdda49d is:

- One invocation before the shared base accesses and threshold comparisons.
- No explicit arguments.
- A result used as the object address for every subsequent indirect call.

Its implementation, failure modes, state-preservation guarantees, ownership rules, and lifetime guarantees are unknown.

### Unresolved indirect dependency identities

No opaque identities are supplied for the indirect targets. Their identities may vary between calls, including within one invocation.

Each is selected through the object’s table entry at byte offset 0x54 and receives the three arguments described in the interface. Returned values are discarded.

Their effects, resource behavior, argument requirements, and abnormal-completion behavior remain **documented_only**. No additional dependency contract is inferred from the table offset alone.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Unless an example states otherwise, assume stable shared state during selection, representable subtraction, valid reached memory accesses, compatible call targets, and normal dependency returns.

1. **Equality overrides a fixed constant.**  
   With selector and Threshold both 0x6004, both Base accesses observing 100, and Equality value 7, the dispatch arguments are **7, 98, 100**, followed by result **1**. The four 6004 values are not dispatched.

2. **Two base observations are not one snapshot.**  
   If the first Base access observes 100 and the second observes 200, a selected base-pair sequence uses **198, 100**. This example is conditional on those observations being permitted by the execution environment; it does not prescribe concurrency semantics.

3. **Lower-side four-call sequence.**  
   With selector 0x6004 and Threshold 0x7000, the four 6004 values are dispatched in numbered order. Normal completion returns **1**.

4. **Lower-side three-call sequence.**  
   With selector 0x6097 and Threshold 0x7000, only the three 6097 values are dispatched. No base-derived operation is appended.

5. **Lower-side base pair.**  
   With selector 0x60BD, Threshold 0x7000, and both Base accesses observing 20, the dispatch arguments are **18, 20**.

6. **Upper-side alternatives.**  
   With Threshold 0x6000, selector 0x60F0 selects the two 60F0 values. Selectors 0x60F1, 0x60F2, and 0x60F3 instead select the adjusted-base/retained-base pair.

7. **Correct constant, wrong side.**  
   With selector 0x60F0 and Threshold 0x7000, there is no indirect dispatch and the normal result is **0**. Acquisition and both Base accesses still occur.

8. **Signed comparison.**  
   With selector −1 and Threshold 0, the lower-side family is selected, no listed constant matches, and the normal result is **0**.

9. **Base changes after dispatch starts.**  
   In the equality sequence, if both initial Base accesses observe 100 and the first operation changes Base to 500, the remaining arguments are still **98, 100**.

10. **Later shared argument changes.**  
    In the 0x6004 sequence, if the first operation changes the second dispatch-value location and the subsequent access observes that change, the second operation receives the new value.

11. **Later target changes.**  
    If the first operation replaces the object’s dispatch-table reference with another valid table, the next operation resolves its target through that replacement rather than through an entry-time cached target.

12. **Ignored operation result.**  
    If an operation returns a failure-like value normally, the function continues the selected sequence and ultimately returns **1**, assuming the remaining operations also return normally.

No numeric outcome is required here for the two overflowing subtraction inputs.

## Uncertainties

### Inferred intent

The function appears to expand a selector into a predefined sequence of operations. This is an interpretation of its structure, not an established application-level contract. The meanings of the constants and dispatch values remain unknown.

### Documented-only unknowns

The supplied evidence does not resolve:

- Initial contents or application meanings of the shared locations.
- Exact indirect-target identities and their contracts.
- The full calling convention and portable object representation.
- Ownership, lifetime, or release obligations for the acquired object.
- Requirements or ownership implications of the forwarded argument.
- Synchronization, atomicity, visibility, and concurrent-modification rules.
- Exact machine-level load counts or scheduling within expressions.
- Subtraction overflow behavior for the two lowest signed inputs.
- Behavior following invalid memory access or abnormal dependency completion.

These unknowns are not replaced with assumptions of purity, state preservation, ownership transfer, or modulo arithmetic.

## Evidence

The supplied static representation supports the interface widths, unconditional acquisition, two distinct ordered Base accesses, preselection subtraction, signed threshold comparisons, equality precedence, selector families, ordered dispatch sequences, repeated target resolution, final fixed-value target-before-argument ordering, ignored indirect results, and normal return values.

It supplies no dependency implementations, execution observations, synchronization contract, or complete ABI specification.

**No runtime tests have been performed.** The acceptance examples are conditional specifications derived from static evidence.
