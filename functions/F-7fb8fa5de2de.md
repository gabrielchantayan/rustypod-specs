<!-- Trusted provenance: F-7fb8fa5de2de; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7fb8fa5de2de

## Purpose

F-7fb8fa5de2de performs an operation on a supplied object while bracketing an initial dependency call with byte stores. It optionally refreshes a 32-bit object field, then either returns one according to two observed indicators or performs an auxiliary-object completion sequence.

The established normal outcomes are:

- Return the 32-bit value one if either indicator is nonzero when tested.
- Otherwise, after successful auxiliary acquisition and normally returning completion calls, return the saved 32-bit result of the initial dependency.

Failure to acquire the required auxiliary object leads to F-a817d8d58c10, which is identified as nonreturning.

Terms such as “indicator,” “refresh,” “provider,” and “auxiliary object” describe observable relationships. They do not establish application-specific meanings.

## Interface

### Inputs and result

The function takes two explicit inputs:

| Input | Representation | Use |
|---|---|---|
| Original object address | Pointer represented within a 32-bit address model | Identifies storage directly accessed by this function and passed to dependencies. |
| Control value | Unsigned 32-bit integer | Enables the optional refresh when nonzero and supplies arguments to the final object dispatch. |

The control value is not normalized to zero or one. Every nonzero bit pattern enables the refresh, and the complete 32-bit value is forwarded during final dispatch.

Normal return values are 32-bit quantities. The initial dependency’s result is preserved as a 32-bit value without local interpretation of its sign or meaning. The early-return result is exactly one.

### Directly accessed object layout

All offsets are byte offsets from the original object address.

| Offset | Access width | Established use |
|---|---:|---|
| `0x00` | 32 bits | Dispatch-table address used for the final indirect call. |
| `0x20` | 32-bit word, then narrowed to 8 bits | First early-return indicator. |
| `0x41` | 8 bits | Second early-return indicator, read only if the first is zero. |
| `0xC0` | 32 bits | Destination of the optional refresh result. |
| `0xE8` | 32 bits | Argument read for the final indirect call. |
| `0xF0` | 8 bits | Written with one before the initial call and zero after it returns normally. |

The first indicator is represented as a word access followed by narrowing to its low-order eight bits. It is not established as an isolated byte access. Relating those bits to a particular byte in memory requires byte-order information not supplied here.

The dispatch-table entries used by this function are addressed by byte offsets. The final entry on the original object is at `0x12C`, equivalent to decimal 300.

### Preconditions and validity

The original object must support every access reached during execution. No local validation establishes its address, bounds, alignment, lifetime, or freedom from concurrent modification.

Even an early-return execution requires writable storage at `0xF0`, acceptance of the original object by the initial dependency, and a readable word at `0x20`. Other requirements depend on the path:

- Nonzero control requires writable 32-bit storage at `0xC0`.
- A zero first indicator requires readable storage at `0x41`.
- Final dispatch requires readable storage at `0x00` and `0xE8`, a usable dispatch table, and a callable entry at `0x12C`.
- Reached provider and auxiliary dispatches require valid receiver storage, dispatch tables, and method entries.
- A nonnull result from F-682adfd5a561 must support the dispatch reached through its entry at `0x58`.

These requirements apply when the corresponding access or call occurs. Entry-time validity alone does not establish that dependencies preserve an object’s lifetime or dispatch table.

## Behavior

### Initial operation

The function writes the byte value one at original-object offset `0xF0`, calls the initial unmapped dependency with the original object address, saves its 32-bit result, and then writes zero at `0xF0`.

The second store occurs only after that dependency returns normally. Neither store preserves the previous value of the byte.

The dependency may observe or modify the flag and other object state. The function does not guarantee that the flag remains one throughout the call; it guarantees its own store before the call and its zero store after normal return. No local cleanup is established for a call that does not return normally.

### Optional refresh

When the unsigned control value is nonzero, F-76b1c50b12f7 is called, followed by F-22be49cfb47c. The latter’s 32-bit result is stored at original-object offset `0xC0`.

Neither call has explicit arguments in the supplied representation. The first call’s result, if any, is not consumed locally. The second result is stored without a zero, sign, range, or error-sentinel check.

This sequence occurs after the flag has been cleared and before either indicator is read. An eventual early return therefore does not suppress the refresh.

With control zero, these two calls and this direct store are omitted. That omission does not establish preservation of `0xC0` by other dependencies.

### Indicator selection

After the initial call and any refresh, the function reads the word at `0x20` and tests only its low-order eight bits.

- A nonzero narrowed value causes an immediate return of one. The byte at `0x41` is not read by this function.
- A zero narrowed value permits the byte read at `0x41`.
- A nonzero value from that byte read causes an immediate return of one.
- Only two zero observations permit auxiliary acquisition.

The relevant values are those obtained at the actual reads, not necessarily those present on entry. The initial dependency and, when enabled, either refresh dependency may change either indicator before it is examined.

Any nonzero eight-bit pattern selects the corresponding early return. Signed versus unsigned interpretation of the narrowed character does not change this decision.

### Auxiliary acquisition

When both indicator observations are zero, the first call to F-d99467d4d2b3 receives an explicit zero-valued argument. That argument follows from the indicator path; no nonzero indicator value reaches this call.

If the first result is zero, acquisition fails and execution reaches F-a817d8d58c10.

Otherwise, F-938c950e5046 obtains a provider address. F-d99467d4d2b3 is then called a second time, and this second result is passed to the provider’s dispatch entry at `0xE8`, together with the provider address.

The first gate result is not reused as that argument. The calls are distinct, with the provider-acquisition call between them; neither equal results nor unchanged global state is guaranteed. No second zero-result check is performed before passing the second result to the provider method.

The provider’s dispatch returns the auxiliary-object address. A null result reaches F-a817d8d58c10. A nonnull result permits further processing. The provider address itself is not locally null-checked before dispatch.

The first F-d99467d4d2b3 call has an explicit argument, while the second has no explicit arguments in the supplied representation. A complete unified signature is therefore unresolved.

### Auxiliary processing

The nonnull auxiliary object is passed to its dispatch entry at `0x1C`. The returned value is treated as a 32-bit quantity and tested against zero.

A zero result bypasses all processing in this subsection and proceeds to contextual completion.

A nonzero result is passed to F-682adfd5a561 together with `0x6800`. There is no positivity test: a value whose high bit is set follows the nonzero branch as well. The dependency’s interpretation of that value and of the constant is not established.

The result of F-682adfd5a561 is treated as an object address:

- **Null result:** F-8157a180d901 is called with `0x1C`. F-2606ad6d534d is then called. Its result is passed, with the auxiliary-object address, to the auxiliary object’s dispatch entry at `0x2C`.
- **Nonnull result:** The returned object’s dispatch entry at `0x58` is invoked. No explicit arguments are shown for that call.

Both branches proceed to contextual completion if their calls return normally. In particular, a null result from F-682adfd5a561 does not locally cause a failure return or invocation of F-a817d8d58c10.

No implicit receiver argument or calling convention is established for the dispatch at `0x58`.

### Contextual completion

After auxiliary processing, or after its zero-result bypass, F-fdd8589eca2d is called. Its result and the original object address are then passed to F-8f9bd59e3ec5, in that order.

After F-8f9bd59e3ec5 returns, the original object’s dispatch entry at `0x12C` is invoked with five explicit arguments:

1. The original object address.
2. The current 32-bit value at original-object offset `0xE8`.
3. The original unsigned 32-bit control value.
4. Zero.
5. The original control value with bit zero flipped by exclusive OR with one.

The final transformation is a 32-bit bitwise operation, not logical negation. It does not add or subtract and does not introduce an arithmetic-overflow case.

The final dispatch-table address and the field at `0xE8` are read for this call after F-8f9bd59e3ec5 returns. They are not entry-time snapshots. Their relative evaluation order is not established by the supplied representation.

The final method’s return value is ignored. Following its normal return, the function returns the initial dependency’s saved 32-bit result.

### Terminal acquisition failure

F-a817d8d58c10 is reached if the first acquisition gate returns zero or the provider method returns a null auxiliary-object address.

The supplied representation identifies this dependency as nonreturning. It does not establish the termination mechanism or behavior if that annotation is violated.

## Side effects

The direct writes are limited to:

- One at `0xF0`, as a one-byte store before the initial dependency.
- Zero at `0xF0`, as a one-byte store after its normal return.
- The optional 32-bit result store at `0xC0`.

Direct reads include the indicators, final field, receiver dispatch-table addresses, and reached dispatch entries.

Dependencies may have additional effects on objects, globals, lifetimes, and dispatch tables. No preservation contract is supplied. In particular:

- Indicator selection observes state after the preceding dependencies.
- Auxiliary dispatch at `0x2C` occurs after F-8157a180d901 and F-2606ad6d534d; preservation of the auxiliary object cannot be assumed.
- Final dispatch observes the original object after contextual completion.

There is no direct allocation, deallocation, reference-count adjustment, or explicit ownership transfer in the supplied body. A returned address and a null check do not establish allocation or ownership. Calling a returned object’s method does not establish release or consumption.

No rollback is shown. If a later operation fails to return normally, preceding stores and dependency effects have already occurred. The supplied representation establishes no synchronization guarantees.

## Dependencies

Mapped dependencies are identified by opaque function IDs only.

| Function ID | Locally established relationship |
|---|---|
| F-76b1c50b12f7 | First optional refresh call; no explicit arguments. |
| F-22be49cfb47c | Second optional refresh call; supplies the 32-bit store at `0xC0`. |
| F-d99467d4d2b3 | First supplies the acquisition gate; later supplies a separate provider-method argument. |
| F-938c950e5046 | Supplies the provider address between the two acquisition calls. |
| F-682adfd5a561 | Receives the nonzero auxiliary result and `0x6800`; supplies a null-tested address. |
| F-8157a180d901 | Receives `0x1C` on the null processing-result branch. |
| F-2606ad6d534d | Supplies the second explicit argument to auxiliary dispatch at `0x2C`. |
| F-fdd8589eca2d | Supplies the first argument to F-8f9bd59e3ec5. |
| F-8f9bd59e3ec5 | Receives that result and the original object before final dispatch. |
| F-a817d8d58c10 | Nonreturning destination for auxiliary acquisition failure. |

No opaque ID is supplied for the initial dependency. Its established local interface is one original-object-address argument and a saved 32-bit result.

Indirect dependencies have no supplied opaque identities. They are distinguished by receiver and dispatch-table byte offset:

| Receiver | Offset | Explicit arguments |
|---|---:|---|
| Provider | `0xE8` | Provider address, second acquisition-call result |
| Auxiliary object | `0x1C` | Auxiliary-object address |
| Auxiliary object | `0x2C` | Auxiliary-object address, F-2606ad6d534d result |
| F-682adfd5a561 result | `0x58` | None shown |
| Original object | `0x12C` | Five arguments specified above |

These relationships do not establish broader dependency contracts.

## Acceptance examples

These are conditional requirements, not executed tests. They assume valid reached accesses and normally returning calls except where a terminal path is specified. Indicator conditions refer to values observed at the actual reads. No dependency-preservation assumption is implicit.

1. **First observed indicator causes early return.**  
   With control zero, if the word read at `0x20` has low-order eight bits `0x7F`, the function returns one after bracketing the initial dependency with the flag stores. It does not itself read `0x41`, perform the optional refresh, or begin acquisition. An entry word of `0x7F` alone is insufficient: the initial dependency could change it.

2. **Refresh precedes a second-indicator early return.**  
   With control two, if the first indicator is observed as zero and the subsequently read byte at `0x41` is nonzero, both refresh calls and the result store at `0xC0` occur before the return of one. These observations are made after the refresh dependencies, either of which may have modified the indicators. Final dispatch does not occur.

3. **Higher bits do not activate the first indicator.**  
   If the actual word read at `0x20` is `0x00000100`, its narrowed value is zero and the byte at `0x41` is consulted. If that byte is observed as zero, acquisition begins. Merely placing `0x00000100` in the object before invocation does not guarantee this path.

4. **Normal completion returns the saved initial result.**  
   If the initial dependency returns `0x12345678`, both indicators are observed as zero at their respective reads, acquisition succeeds, and all completion calls return normally, the function returns `0x12345678`. The final method’s result does not replace it.

5. **Zero auxiliary result bypasses processing.**  
   A zero result from auxiliary dispatch at `0x1C` omits F-682adfd5a561 and both result branches. Contextual completion and final dispatch still occur.

6. **Null processing result is recoverable locally.**  
   A null F-682adfd5a561 result selects F-8157a180d901 with `0x1C`, then F-2606ad6d534d, then auxiliary dispatch at `0x2C`. Normal returns lead to completion, not the terminal acquisition-failure dependency.

7. **The transformed control argument remains full-width.**  
   On a path reaching final dispatch, control two produces final arguments two and three in the third and fifth positions. Control `0xFFFFFFFF` produces `0xFFFFFFFF` and `0xFFFFFFFE`.

8. **Acquisition results are not interchangeable.**  
   A nonzero first acquisition result permits provider acquisition. If the second result is zero, that zero is still forwarded to the provider method. Only the provider’s returned auxiliary address then determines whether this acquisition stage succeeds.

9. **Acquisition failure is terminal locally.**  
   A zero first gate result skips provider acquisition and reaches F-a817d8d58c10. A nonzero gate followed by a null provider-method result also reaches that dependency, without auxiliary processing or contextual completion.

## Uncertainties

The following are **documented_only**:

- Application meanings of the fields, constants, and dependency operations.
- Identity and broader contract of the initial dependency.
- Complete signature of F-d99467d4d2b3.
- Calling convention and any implicit arguments for dispatch at `0x58`.
- Ownership, borrowing, retention, destruction, and release rules for returned objects and contextual values.
- Dependency preservation of indicators, globals, dispatch tables, object contents, and lifetimes.
- Exact relative order of reads within individual call expressions where not otherwise established.
- Platform byte order and exact machine-instruction access granularity beyond the supplied representation.
- Nonreturning dependency mechanism.
- Behavior for invalid storage, concurrent mutation, or violation of the nonreturning annotation.

The flag bracketing suggests an operation-in-progress marker, and the conditional store suggests refreshed state. These are inferred intents, not established semantics.

## Evidence

This contract uses the supplied static function representation and opaque dependency mapping. The prior draft and reviewer feedback are not independent evidence of behavior.

Established static evidence supports the access offsets and represented widths, constants, branch conditions, call sequencing, dispatch offsets, locally consumed results, and nonreturning annotation. In particular, it places the indicator reads after the initial dependency and optional refresh, so entry-state indicator values alone cannot establish an early-return outcome.

No runtime tests have been performed. The acceptance examples specify conditional behavior rather than observed executions.
