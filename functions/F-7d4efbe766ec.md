<!-- Trusted provenance: F-7d4efbe766ec; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7d4efbe766ec

## Purpose

F-7d4efbe766ec coordinates a bounded fitting operation over an opaque primary input, optionally accounting for an auxiliary input. It returns a signed 32-bit value and, on selected paths, writes four 32-bit output fields.

**Established static behavior:** The function obtains a bound, performs an initial fitting call, and compares that call’s direct result with the bound. A mismatch and a nonzero mode enable special handling. Mode 1 can consume elements through dependencies, mode 2 returns without publishing outputs, and mode 4 can perform a replacement fitting call with a reduced budget. Other nonzero modes also return without publishing outputs on a mismatch.

**Inferred intent:** The operations are consistent with fitting a sequence within a measurement budget, possibly reserving room for an auxiliary sequence. This does not establish that inputs are text or that any quantity measures bytes, characters, elements, or display width.

The return value is read from local measurement storage. The initial fitting dependency receives that storage’s address and may retain it. Consequently, later dependency calls can affect the eventual return independently of their direct return values. Preservation across those calls is not established.

## Interface

The function accepts eleven arguments and returns a signed 32-bit integer.

| Position | Contract name | Representation | Established role |
|---|---|---|---|
| 1 | Context | Opaque 32-bit value | Passed to fitting and measurement dependencies. |
| 2 | Primary input | Opaque 32-bit value | Passed to bound, fitting, and measurement dependencies; copied into local cursor storage. |
| 3 | Configuration A | Opaque 32-bit value | Passed to fitting and measurement dependencies. |
| 4 | Configuration B | Opaque 32-bit value | Passed to fitting and measurement dependencies. |
| 5 | Budget | Signed 32-bit integer | Used in signed comparisons and arithmetic. |
| 6 | Auxiliary input | Opaque 32-bit value | Measured on mismatch paths in modes 1 and 4. |
| 7 | Mode | Signed 32-bit integer | Selects special handling for values 1, 2, and 4. |
| 8 | Output A | Address of a writable 32-bit field | Receives zero whenever outputs are published. |
| 9 | Output B | Address of a writable signed 32-bit field | Receives a selected fitting result or zero. |
| 10 | Output C | Address of a writable signed 32-bit field | Receives a removal count or zero. |
| 11 | Output D | Address of a writable signed 32-bit field | Receives a bound, a fitting result, or zero. |

These are contract names, not recovered public API names. The opaque inputs may represent addresses, handles, flags, or other values. No input object layout is established.

### Storage requirements

Each reached output store requires valid writable storage for a 32-bit value, with the execution environment’s required alignment. No pointer validation is visible. The four addresses are separate arguments; no aggregate layout or non-overlap requirement is enforced.

Paths returning without direct output writes do not dereference these addresses in the wrapper. This is not a guarantee about dependency side effects.

Two local storage locations are exposed to dependencies:

- **Return-measurement storage:** Its address is passed to each fitting call. The wrapper does not initialize it before the first call.
- **Cursor storage:** It initially contains the primary-input value. Its address is passed to the extraction dependency during mode-1 iteration.

Updating cursor storage does not itself replace a caller-held argument value. Either exposed address may be retained by a dependency; the supplied body establishes no non-retention guarantee. Later dependencies may therefore modify exposed storage before the wrapper next reads it.

## Behavior

### Initial fitting and dispatch

Every invocation first calls F-4439a539b76a with the primary input. Its signed 32-bit return becomes the **initial bound**.

Next, F-d4f370b64986 receives, in order:

- Context.
- Primary-input value.
- Initial bound.
- Configuration A and Configuration B.
- Budget.
- Address of return-measurement storage.
- A final argument of zero.

Its direct signed 32-bit return becomes the **initial fit result**. That result is distinct from the contents of return-measurement storage.

Special handling occurs only if the initial fit result differs from the initial bound and the mode is nonzero. Otherwise, ordinary publication occurs immediately.

No sign or range relationship between the bound, fit result, measurement, and budget is validated.

### Ordinary publication

Ordinary publication performs these 32-bit stores in order:

1. Output D receives zero.
2. Output C receives zero.
3. Output A receives zero.
4. Output B receives the selected fit result.

The function then returns the value read from return-measurement storage.

The selected fit result is the initial fit result unless mode 4 performs a replacement fitting call. Changes to return-measurement storage do not, by themselves, change the selected fit result.

### Mode 1 on an initial mismatch

F-7feff8070389 is called with context, auxiliary input, Configuration A, and Configuration B. Its direct signed return is the **auxiliary measurement**.

This call occurs after the first fitting dependency has received the return-measurement address. The auxiliary-measurement call is not established to preserve that storage.

#### Auxiliary measurement exceeds the budget

If the budget is less than the auxiliary measurement under signed comparison, the function publishes:

1. Zero to Output B.
2. Zero to Output A.
3. Zero to Output C.
4. The initial fit result to Output D.

It then returns the current return-measurement storage value, after the auxiliary-measurement call and output stores. It does **not** restore the measurement originally supplied by the initial fitting call.

Thus, the return equals the initial fitting measurement only if no intervening operation changes that storage. The auxiliary measurement itself is not explicitly assigned to the return-measurement storage.

There is no primary-input measurement, extraction, element measurement, or replacement fitting call on this branch.

#### Auxiliary measurement does not exceed the budget

F-7feff8070389 is called again, using the primary-input value instead of the auxiliary input.

The wrapper assigns this call’s direct return to return-measurement storage, replacing its previous contents. A mutation of that storage during this particular call is therefore superseded by the assignment after the call returns.

The auxiliary measurement is then added to the stored primary measurement. This combined value initializes two logically distinct quantities:

- The stored running measurement.
- A separate excess value, equal to the combined measurement minus the budget.

The removal count starts at zero. The removal limit is the initial bound, not the initial fit result.

Iteration continues only while the excess is strictly positive and the removal count is less than the limit, using signed comparisons. Each iteration has this observable dependency order:

1. F-2be305f439e7 receives the address of cursor storage and returns an opaque 32-bit element value.
2. F-122d0efb9091 receives context, that element value, and both configurations, returning a signed 32-bit element measurement.

After these calls, the wrapper subtracts the element measurement from the separate excess and from the **then-current** return-measurement storage. It then increments the removal count.

The distinction between those two subtractions is material:

- The excess is maintained separately; no address of it is passed to these dependencies.
- Return-measurement storage has already been exposed. Either dependency may modify it through a retained address before the wrapper’s subtraction.

Therefore, without a preservation assumption, the stored running measurement need not remain equal to the initial combined measurement minus the accumulated element measurements. Nor must it remain equal to the excess plus the original budget.

Cursor progression is likewise dependency-controlled. The wrapper does not independently advance or validate the cursor. Subsequent extraction calls use the cursor storage as it then exists.

After iteration, publication occurs in this order:

1. Zero to Output B.
2. Zero to Output A.
3. Removal count to Output C.
4. Initial bound to Output D.

The function returns the current return-measurement storage value.

Boundary behavior:

- Nonpositive initial excess causes zero iterations.
- A nonpositive initial bound causes zero iterations even when excess is positive.
- Zero element measurement leaves excess unchanged but still consumes one iteration.
- Negative element measurement increases excess, subject to arithmetic limits.
- Reaching the bound stops iteration even if excess remains positive.
- No replacement fitting call occurs.
- A final return within budget is not guaranteed, including when dependencies mutate the exposed measurement storage.

### Mode 2 on an initial mismatch

The function returns the current return-measurement storage value without further dependency calls or direct output writes.

There is no wrapper-issued dependency call between the initial fitting call and this return.

### Other nonzero modes on an initial mismatch

Every mode other than 1, 2, and 4 returns the current return-measurement storage value without further dependency calls or direct output writes.

Negative mode values follow this rule. No invalid-mode error or normalization is visible.

### Mode 4 on an initial mismatch

F-7feff8070389 receives context, auxiliary input, and both configurations.

If its auxiliary measurement is less than or equal to the budget under signed comparison, a second F-d4f370b64986 call receives the same context, primary-input value, initial bound, and configurations, but with:

- Budget reduced by the auxiliary measurement.
- The same return-measurement storage address.
- A final argument of zero.

The second call’s direct return replaces the selected fit result. After that call returns, the wrapper adds the auxiliary measurement to the then-current return-measurement storage. Ordinary publication follows.

If the auxiliary measurement exceeds the budget, the replacement fitting call is skipped. Ordinary publication uses the initial fit result. The returned measurement is nevertheless the storage value **after** the auxiliary-measurement call, not necessarily the initial fitting measurement.

Mode 4 neither directly measures the primary input through F-7feff8070389 nor extracts elements.

## Side effects

The wrapper’s direct caller-visible writes are the ordered 32-bit output stores described above. It does not read the previous output values.

Publication is sequential, not atomic. A failed store can prevent later stores and normal return; no rollback is visible.

Overlapping outputs follow store order. If all four addresses identify the same 32-bit field:

- Ordinary publication leaves the selected fit result.
- Mode-1 publication leaves its Output D value.
- Mismatch returns for mode 2 and unsupported nonzero modes perform no direct store.

All wrapper-issued dependency calls precede output publication. Within each iteration, extraction precedes element measurement, and both precede the wrapper’s measurement subtractions.

No explicit allocation, release, or ownership transfer appears in the wrapper. This does not establish that dependencies avoid allocation, release referenced memory, preserve globals, or refrain from modifying input objects.

Retaining a local address does not establish ownership or permission to use it after the invocation. The wrapper supplies no lifetime-extension mechanism and no visible cleanup protocol.

There is no separate error channel. Negative dependency results are used as signed values rather than explicitly recognized as errors. Abnormal dependency termination and invalid memory accesses are not handled locally.

## Dependencies

| Opaque ID | Established interaction | Unresolved contract |
|---|---|---|
| F-4439a539b76a | Called once with the primary input; supplies the initial bound. | Units, validity rules, errors, and side effects. |
| F-d4f370b64986 | Called initially and optionally again in mode 4; returns a fit result and receives the measurement-storage address. | Whether it writes the storage, retains its address, or changes referenced inputs; ownership and failure behavior. |
| F-7feff8070389 | Measures auxiliary input in modes 1 and 4; also measures primary input in one mode-1 branch. | Units, sign constraints, determinism, and possible mutation of storage exposed by an earlier call. |
| F-2be305f439e7 | Receives cursor-storage address and returns an opaque element. | Cursor updates, termination rules, address retention, and effects on other exposed storage. |
| F-122d0efb9091 | Receives an extracted element and returns its signed measurement. | Units, error conventions, and effects on exposed storage or other state. |

Matching argument values do not imply interchangeable results or absence of side effects. No dependency is assumed to preserve globals or own or release memory.

## Acceptance examples

These are conditional static cases, not executed tests. Assume normal dependency returns, valid output storage, distinct output fields unless stated otherwise, and representable signed 32-bit arithmetic.

1. **Equality bypasses special handling.**  
   Bound 6, initial fit result 6, fitting-written measurement 20, mode 1. No auxiliary call or iteration occurs. Return: 20. Outputs A, B, C, D: 0, 6, 0, 0.

2. **Zero mode publishes a partial fit.**  
   Bound 6, initial fit result 3, fitting-written measurement 12, mode 0. Return: 12. Outputs: 0, 3, 0, 0.

3. **Mode 2 omits publication on mismatch.**  
   Bound 6, initial fit result 3, fitting-written measurement 12, mode 2. Return: 12. No further dependency calls or direct output writes occur.

4. **Unsupported mode omits publication on mismatch.**  
   With the preceding bound, fit result, and measurement, mode −1 returns 12 without direct output writes.

5. **Mode 1 rejects auxiliary measurement above budget.**  
   Bound 6, fit result 3, initial measurement 12, budget 4, auxiliary measurement 5. Outputs: 0, 0, 0, 3. **If the auxiliary call preserves return-measurement storage**, return: 12. If it instead writes 27 through a retained address, return: 27.

6. **Mode 1 removes enough measured elements under preservation.**  
   Bound 5, fit result 2, budget 9, auxiliary measurement 3, primary measurement 12. Initial excess: 6. **Assuming extraction and element-measurement calls preserve return-measurement storage**, element measurements 4 and 3 cause exactly two iterations. Return: 8. Outputs: 0, 0, 2, 5.

7. **Mode 1 stops at its bound under preservation.**  
   Bound 2, fit result 1, budget 4, auxiliary measurement 2, primary measurement 10. **Assuming iteration dependencies preserve return-measurement storage**, two element measurements of 1 leave return 10. Outputs: 0, 0, 2, 2. Excess remains positive.

8. **Mode 4 performs replacement fitting.**  
   Bound 8, initial fit result 5, budget 10, auxiliary measurement 3. The replacement fitting call receives budget 7. If its direct result is 4 and its measurement storage contains 6 when it returns, final return: 9. Outputs: 0, 4, 0, 0.

9. **Mode 4 skips replacement without restoring measurement.**  
   Initial fit result 5, initial measurement 12, budget 10, auxiliary measurement 11. No replacement call occurs. Outputs: 0, 5, 0, 0. **If the auxiliary call preserves measurement storage**, return: 12. If it changes that storage to 19, return: 19.

10. **Iteration mutation separates return from excess.**  
    Mode 1, bound 5, mismatching fit result 2, budget 9, auxiliary measurement 3, primary measurement 12. Initial stored measurement: 15; excess: 6. Suppose the first extraction call changes measurement storage to 100, the element-measurement call leaves it unchanged and returns 6. The wrapper reduces excess to zero and stored measurement to 94. Exactly one iteration occurs; return: 94. Outputs: 0, 0, 1, 5.

## Uncertainties

The following remain **documented_only**:

- Input representations, units, and output meanings beyond the assignment rules.
- Dependency validation, error conventions, memory ownership, allocation, release, and global effects.
- Whether fitting always writes its measurement output or retains its address.
- Whether later calls mutate exposed measurement or cursor storage.
- The return on paths that read measurement storage without an established prior write. Mode 1’s primary-measurement assignment can establish a value even if initial fitting did not.
- Overflow behavior for measurement addition, excess calculation, repeated subtraction, reduced-budget calculation, and mode-4 final addition. Signed 32-bit arithmetic is exposed, but wrapping or saturation is not established.
- Platform byte order, address encoding, and alignment requirements.

Within representable arithmetic, the removal counter cannot require incrementing beyond the maximum signed 32-bit value: it starts at zero and increments only while less than a signed 32-bit bound.

## Evidence

This contract derives from static inspection of the supplied function body and opaque dependency mapping. Established claims concern visible calls, argument flow, signed comparisons, arithmetic, address exposure, output-store order, and returns.

Retained-address mutation is a permitted dependency behavior, not an observed event. The source provides no dependency contract excluding it; numerical preservation claims are therefore conditional where later calls intervene.

No dependency implementations or external interface declarations were supplied. No runtime tests have been performed. Acceptance examples describe conditional consequences, not observed executions.
