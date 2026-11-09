<!-- Trusted provenance: F-1b2a7c8fe0e8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1b2a7c8fe0e8

## Purpose

F-1b2a7c8fe0e8 performs one of two operations selected by a receiver flag:

- **Mode bit clear:** obtain a primary-object value, copy and transform a four-word descriptor, notify the receiver, and conditionally deliver the descriptor to a captured secondary object.
- **Mode bit set:** update the receiver’s selection using bounds, a window value, and a pending increment, then invoke F-92bea7c30bd6.

On normal completion through either path, the function writes zero to the pending increment. An initially null primary object and equal copied descriptor endpoints cause explicit early returns that bypass that final write.

Bypassing the write does not generally mean preserving entry state. In particular, two primary dispatches precede the equal-endpoint return and may already have changed the pending increment.

These are statically established operations. Interpretations such as geometry, scrolling, or circular selection are not established requirements.

## Interface

### Entry and result

The function accepts one receiver address. No defined return value is exposed. Its observable results are direct memory writes, dependency calls, and any effects produced by those dependencies.

The receiver itself is not checked for null.

### Storage layout

The supplied representation uses four-byte integer words and four-byte object-address fields. Offsets below are byte offsets.

| Receiver offset | Contract label | Observed use |
|---|---|---|
| `0x034` | Secondary object | Nullable address captured during mode-clear processing |
| `0x0EC` | Primary object | Nullable address checked at entry and reloaded at later stages |
| `0x100` | Lower value | Comparison operand, arithmetic baseline, replacement selection |
| `0x104` | Upper value | Comparison operand and endpoint-adjustment value |
| `0x118` | Pending increment | Movement operand; cleared at normal completion |
| `0x11C` | Selection | State read and conditionally written in mode-set processing |
| `0x120` | Window value | Span comparison and endpoint calculation |
| `0x124` | Mode flags | Bit `0x1` selects the path |

The primary object supplies four consecutive descriptor words:

| Primary offset | Content |
|---|---|
| `0x080` | First endpoint |
| `0x084` | Opaque second word |
| `0x088` | Third word, used as the other endpoint |
| `0x08C` | Opaque fourth word |

These words are copied, in that order, into a contiguous 16-byte temporary descriptor. The first and third words participate in signed arithmetic. The other two are initially copied without interpretation; dependency behavior may subsequently affect any descriptor word.

The secondary object supplies:

| Secondary offset | Observed use |
|---|---|
| `0x048` | Flag word; bit `0x2` enables delivery |
| `0x080` | Interior address passed to F-b01436b9ce2b |

Indirect dispatch targets are obtained through the relevant object’s dispatch-table address. The observed slot offsets are `0x174` and `0x170` for the primary object, `0x1D0` for the receiver, and `0x148` for the secondary object.

### Arithmetic and validity requirements

Integer comparisons involving endpoints, bounds, selection, window values, pending increments, and signed dependency results are 32-bit signed comparisons. Flag operations use 32-bit bit patterns.

Addition, subtraction, and multiplication have no visible overflow checks. A predictable mathematical interpretation requires all relevant intermediate results to fit signed 32-bit range. This includes bound differences, endpoint construction, transformation products, selection movement, and the later descriptor adjustment. Behavior outside that range remains **documented_only**.

Each accessed object, interior address, dispatch table, and callable target must remain valid when used. The initial primary-object check does not validate subsequent reloads after dependencies execute.

No explicit validation requires ordered bounds, a nonnegative window, a positive descriptor span, or an increment restricted to one domain traversal.

## Behavior

### Initial gate

The function first reads the receiver’s primary-object address.

- If null, it returns without a dependency call or direct write.
- Otherwise, it reads the mode flags and selects a path using bit `0x1`.

The mode decision is not repeated after dependency calls.

### Mode-clear descriptor processing

#### Readiness, capture, and equal endpoints

The function invokes primary dispatch slot `0x174`, using the primary object captured at entry to locate the target.

A zero result skips all remaining descriptor processing and leads to the final pending-increment clearing write.

For a nonzero result, the following order is established:

1. Capture the receiver’s secondary-object address.
2. Reload the primary-object address and invoke its slot `0x170`.
3. Retain the signed 32-bit query result.
4. Reload the primary-object address again.
5. Read the four descriptor words at `0x080`, `0x084`, `0x088`, and `0x08C`, in order.

Consequently, readiness, query, and descriptor copying may involve different primary objects if dependencies replace the receiver’s primary-object field.

If the copied first and third words are equal, the function returns immediately. It:

- Performs no direct pending-increment write and skips the final clearing write; changes made by earlier dependencies remain possible.
- Skips F-b01436b9ce2b, F-d156b0b83eb3, the receiver notification, and the secondary delivery operations that would otherwise follow.
- Does not use the second or fourth word to decide this return.

For example, a preceding primary dispatch may change the pending increment from `7` to `9`; equal copied endpoints then allow the function to return with `9`.

Unequal endpoints continue regardless of the sign of their difference.

#### Transformation and receiver notification

If the captured secondary address is nonnull, F-b01436b9ce2b is called with its interior address at `0x080`. This call’s result is discarded.

F-d156b0b83eb3 is then invoked with:

- The temporary descriptor address.
- A zero second argument.
- A third argument calculated as the queried value minus the receiver’s lower value, multiplied by the third descriptor endpoint minus the first.

The lower value used here is read after the optional secondary helper returns. It is not an entry-time snapshot.

The caller does not establish F-d156b0b83eb3’s transformation, which words it preserves, or whether it produces other effects.

After that call returns, receiver dispatch slot `0x1D0` is invoked with the receiver and the retained query result. This occurs whether or not a secondary object was captured.

#### Delivery gate and adjustment

Delivery uses the earlier captured secondary address, not a fresh read of the receiver’s secondary-object field.

If that captured address is null, no delivery occurs. Otherwise, the secondary flag word at `0x048` is read after the receiver notification. Delivery occurs exactly when bit `0x2` is set.

The receiver notification’s return value does not enable or suppress delivery: a nonnull secondary supplies the flag-derived decision, while a null secondary excludes delivery independently.

When delivery is enabled:

1. F-b01436b9ce2b receives the current primary-object address plus `0x080`.
2. If its signed result is positive, F-b01436b9ce2b receives the captured secondary address plus `0x080`.
3. F-870c4fcf7295 then receives the secondary helper result followed by the positive primary helper result.
4. The temporary descriptor’s third word is increased by the secondary helper result minus the product of the primary helper result and the F-870c4fcf7295 result.
5. Secondary dispatch slot `0x148` is invoked with the captured secondary object and the temporary descriptor address.

Steps 2–4 are skipped when the primary helper result is zero or negative; the secondary callback still occurs.

The adjustment uses the descriptor state remaining after preceding dependencies. F-870c4fcf7295 is not established to perform division, so this operation is not specified as a remainder calculation.

Normal completion clears the pending increment. F-92bea7c30bd6 is not called on this path.

### Mode-set selection processing

#### Bounds and readiness gates

The lower and upper values are compared before any readiness call.

If equal, the lower value is written to selection. Primary readiness and query dispatches are skipped. F-92bea7c30bd6 is then called, followed by clearing the pending increment.

For unequal bounds, the primary readiness slot `0x174` is invoked using the initially captured primary object.

If readiness returns zero, this routine makes no direct selection write. It still calls F-92bea7c30bd6 and then clears the pending increment. Dependencies may independently change selection.

If readiness succeeds, the function reads the lower value and compares the window value against upper minus lower:

- If the window is not strictly smaller, selection is written with that lower value. The query is skipped.
- If the window is strictly smaller, primary slot `0x170` is invoked using a reloaded primary-object address.

The later bounds and window comparison are therefore not guaranteed to use entry-state values.

#### Zero pending increment

After the query returns, the pending increment is tested.

If zero, the queried value is written directly to selection. No lower-bound or upper-bound normalization occurs on this branch.

#### Nonzero pending increment

For a nonzero pending increment, the function reads selection and upper value, then forms an endpoint by adding the window value to the captured selection.

If the endpoint is greater than or equal to the captured upper value, the function subtracts that upper value once. It does not repeatedly reduce the endpoint and does not subtract the lower value.

Whether movement occurs depends on signed comparisons:

- **Selection below endpoint:** no movement occurs when the queried value is at least selection and strictly below endpoint. Otherwise, the pending increment is added to the captured selection and the result is written to selection.
- **Selection at or above endpoint:** movement occurs only when the queried value is at least endpoint and strictly below selection.

When selection equals endpoint, the movement condition is never satisfied.

The increment is read again if movement is performed; the earlier nonzero test is not itself a retained increment value.

Normalization then occurs even when movement was skipped:

1. The lower value is read again.
2. If current selection is below that lower value, selection is written as the captured upper value minus one.
3. Current selection is compared with the captured upper value.
4. If selection is greater than or equal to that upper value, selection is written with the lower value read for normalization.

This is a fixed correction sequence, not general modular reduction. Large overshoots are replaced by a boundary value rather than reduced by their full magnitude. The upper value used here is the value captured before endpoint construction, whereas the lower value is read later.

Every normally progressing mode-set branch then calls F-92bea7c30bd6 with the receiver and finally writes zero to the pending increment.

## Side effects

### Direct writes and dependency changes

The function directly writes only:

- Selection at receiver offset `0x11C`, on the mode-set branches described above.
- Zero to pending increment at `0x118`, at normal completion.

Movement followed by normalization can produce multiple selection writes before F-92bea7c30bd6 executes.

The pending-increment clearing write follows callbacks. Thus a pending increment written by a dependency is overwritten on normal completion. At the equal-endpoint return, no such overwrite occurs; the value may reflect earlier dependency changes. At the initial null-primary return, no dependency has been called.

No dependency is assumed to preserve receiver fields, object identity, dispatch tables, global state, or object validity.

### Storage lifetime and ownership

The descriptor is temporary storage belonging to this invocation. Its address is passed to F-d156b0b83eb3 and, conditionally, the secondary callback.

No allocation, release, reference-count operation, or ownership transfer is shown. Retaining the temporary descriptor address beyond the invocation requires an external lifetime contract not established here.

Addresses at object offset `0x080` identify interior object storage, not copies. The caller does not establish whether dependencies mutate that storage or release any object.

### Failure behavior

There is no explicit error result, rollback, exception handler, or abnormal-completion cleanup. The final pending-increment write is an ordinary action reached through normal control flow, not a guaranteed finalizer.

Invalid addresses, invalid dispatch targets, dependency failures, and arithmetic outside the established domain have no specified recovery behavior.

## Dependencies

| Dependency | Established use | Unresolved behavior |
|---|---|---|
| F-d156b0b83eb3 | Receives temporary descriptor address, zero, and calculated product | Transformation, other mutations, return meaning |
| F-b01436b9ce2b | Receives primary or secondary interior address at `0x080`; some results are signed arithmetic operands | Measurement meaning, mutation, ownership effects |
| F-870c4fcf7295 | Receives secondary-derived result followed by positive primary-derived result; output participates in multiply-subtract adjustment | Arithmetic operation and failure behavior |
| F-92bea7c30bd6 | Receives receiver on every normally progressing mode-set path | Notifications, selection changes, other effects |
| Primary dispatch slot `0x174` | Zero/nonzero readiness gate | Semantic predicate, implicit arguments, side effects |
| Primary dispatch slot `0x170` | Produces signed queried value | Meaning, implicit arguments, side effects |
| Receiver dispatch slot `0x1D0` | Receives receiver and queried value | Effects and result meaning |
| Secondary dispatch slot `0x148` | Receives captured secondary object and temporary descriptor address | Consumption, mutation, copying, retention |

Opaque identities are available for the mapped direct dependencies only. The indirect targets are identified by their observed dispatch slots; no independent function identity is established. The two primary dispatches expose no explicit argument list, so their implicit receiver convention is **documented_only**.

## Acceptance examples

These are conditional static expectations, not executed tests. Unless an example states otherwise, dependencies return normally, preserve relevant state, and arithmetic remains within signed 32-bit range.

1. **Initially absent primary.** Primary address null and pending increment `7`: no dependency calls or direct writes occur; the increment remains `7`.

2. **Equal endpoints without dependency mutation.** Mode clear, readiness nonzero, copied endpoints both `12`: the function returns after capture, skips subsequent helpers and callbacks, and performs no pending-increment write.

3. **Equal endpoints after dependency mutation.** The increment begins at `7`; readiness or query changes it to `9`; the copied endpoints are equal. The function returns without clearing it, leaving `9`, provided no other dependency changes it again.

4. **Mode-clear readiness failure.** Readiness returns zero: query, descriptor capture, transformation, and delivery are skipped; pending increment is cleared.

5. **Absent secondary.** Unequal copied endpoints and captured secondary null: transformation and receiver notification occur, but secondary helpers and callback do not. The receiver notification result cannot enable delivery.

6. **Enabled delivery, nonpositive primary helper.** Secondary flag bit `0x2` is set and the later primary helper returns `0`: the secondary callback receives the descriptor without the additional multiply-subtract adjustment.

7. **Equal mode-set bounds.** Lower and upper both `10`: selection is written as `10`, F-92bea7c30bd6 executes, and the increment is cleared. Neither primary dispatch occurs.

8. **Window threshold equality.** Lower `10`, upper `30`, window `20`, readiness successful: selection becomes `10` without a query.

9. **Unnormalized zero-increment query.** Lower `10`, upper `30`, window `5`, readiness successful, increment zero, query `99`: selection is written as `99` before F-92bea7c30bd6.

10. **Nonwrapped membership.** Lower `0`, upper `100`, selection `20`, window `10`, increment `4`: query `25` retains `20`; query `30` produces `24`.

11. **Wrapped membership.** Lower `0`, upper `100`, selection `95`, window `10`, increment `3`: endpoint becomes `5`. Queries `97` and `2` retain `95`; query `50` produces `98`.

12. **Upper correction after movement.** With the preceding bounds, selection `95`, window `10`, increment `10`, query `50`: selection is first written as `105`, then as `0`.

13. **Lower correction after movement.** Lower `10`, upper `30`, selection `12`, window `5`, increment `-5`, query `20`: selection is first written as `7`, then as `29`.

14. **Equal constructed endpoint.** Lower `0`, upper `100`, selection `20`, window `0`, increment nonzero: no queried value causes movement. Normalization still runs.

## Uncertainties

The following remain **documented_only**:

- Application meanings of the objects, descriptor, queried value, bounds, and flags.
- Whether the endpoint and selection operations intentionally implement a circular domain.
- Required ordering or permitted ranges of bounds and window values.
- Overflow behavior and target-specific arithmetic guarantees.
- Dependency mutations, global effects, object replacement, destruction, and ownership.
- Implicit arguments and calling conventions of primary dispatches.
- Reentrancy, concurrent mutation, synchronization, and failure recovery.
- Whether a dependency retains the temporary descriptor address.

A geometric or scrolling interpretation is plausible inferred intent, not an acceptance requirement.

## Evidence

This contract is based solely on the supplied static representation and direct-dependency identity mapping.

Established evidence comprises field widths and offsets, dispatch slots, explicit call arguments, descriptor-copy order, branch conditions, arithmetic expressions, direct writes, and the placement of both early returns.

The equal-endpoint return follows both primary dispatches and precedes every direct pending-increment write. It therefore establishes omission of clearing, not preservation of the entry value.

Dependency internals and application semantics are not established. No runtime tests have been performed.
