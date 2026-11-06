<!-- Trusted provenance: F-d260cfbdb3c8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d260cfbdb3c8

## Purpose

F-d260cfbdb3c8 conditionally performs a state-transition sequence when a queried 32-bit value differs from a target captured at entry. Its observable behavior consists of ordered dependency calls, an optional preliminary 16-bit write, and an optional final 32-bit write derived from a packed 64-bit result.

Two independent comparisons control these effects:

- The preliminary write depends on equality between a **separate read of the target global** and a read of the comparison global.
- The transition sequence depends on inequality between the queried value and the **originally captured target**.

These target observations must not be conflated. The supplied representation does not establish that the target global remains stable between its initial capture and its separate evaluation for the preliminary equality.

The controller/history interpretation is inferred from the dispatch structure and textual data passed to a dependency. It is not an established application-level meaning.

## Interface

### Invocation and result

The function takes no explicit arguments and has no defined return value. Callers observe dependency effects and memory changes rather than a returned status.

There is no visible success indicator, error code, or ordinary failure-return branch.

### Values and widths

| Item | Represented width and use |
|---|---|
| Initially captured target | 32 bits; retained for the later comparison and controller dispatch |
| Separately evaluated target and comparison values | 32 bits; used only for the preliminary equality |
| Queried current value | 32 bits; compared with the captured target |
| Preliminary field | 16-bit replacement write |
| Two separately saved dependency results | 32 bits each |
| Payload intermediates | Opaque 32-bit values |
| Packed dependency result | 64 bits |
| Final output field | 32-bit replacement write |

The representation uses 32-bit pointer-bearing values and 32-bit dispatch-table entries. This describes the supplied representation, not a portable ABI requirement.

No signed ordering comparison occurs. Equality and zero/nonzero tests are unaffected by interpreting the same 32-bit patterns as signed or unsigned. Relevant bit operations are:

- Setting bit 14 of a 32-bit saved value with mask 0x00004000.
- Testing bit 46 of the 64-bit result with mask 0x0000400000000000.
- Extracting bits 32–63 of that result.
- Clearing bit 14 of the extracted portion with mask 0xFFFFBFFF.

### Global storage roles

The contract uses these semantic labels:

- **Target global:** read once for retention, then separately evaluated for the preliminary equality.
- **Comparison global:** evaluated against the second target observation.
- **Transition-setting global:** read during the transition and passed to F-e2d95185a0cc.
- **Output-base global:** read when the final write is required.

The labels do not establish non-aliasing, immutability, volatility, synchronization, or independent storage.

### Preconditions for normal completion

Every executed dependency and indirect dispatch must return normally for subsequent behavior to occur. Required memory and dispatch accesses must be valid.

In particular:

- When the preliminary equality holds, F-fdd8589eca2d must provide an address supporting a 16-bit write at byte offset 0x80.
- Returned service objects must support the table accesses used on the executed path.
- The result of the first controller dispatch must support the subsequent query dispatch.
- When the final write is required, the then-read output base must support a 32-bit write at byte offset 4.

There are no visible null, bounds, allocation-result, or address-overflow checks. These preconditions do not define recovery behavior for invalid inputs or dependency failures.

## Behavior

### Initial capture and independent preliminary equality

Before any dependency invocation, the function reads the target global and retains that 32-bit value for later use.

It then evaluates a separate equality between:

- A new read of the target global.
- A read of the comparison global.

The representation does not establish which of these two equality operands is evaluated first. It does establish that the initial target capture precedes this equality evaluation.

If the separately evaluated operands are equal:

1. F-fdd8589eca2d is called without explicit arguments.
2. The 16-bit value 6 is written at byte offset 0x80 from its returned address.

If those operands differ, both the call and the write are skipped.

The preliminary decision does not compare the comparison global with the retained target. Equating the retained target with the second target observation requires an additional stability assumption that is not established here.

The preliminary decision is completed before F-fdd8589eca2d is invoked. Changes that this dependency makes to the relevant globals do not retroactively change that decision.

### Object acquisition and query

After the preliminary branch, the following calls occur in order:

1. F-17fe47085641 obtains the first primary service object.
2. F-76b1c50b12f7 obtains a retained controller-like object.
3. F-000000000001 is dispatched through the controller’s table at byte offset 0x14, receiving the controller explicitly.
4. F-000000000002 is dispatched through the resulting object’s table at byte offset 0x68.

F-000000000002 has no explicit arguments in the supplied representation. Its 32-bit result is the queried current value.

That result is compared with the **initially captured target**, not with the target global’s second observation or any later global value.

If the queried value equals the captured target, the function returns. No transition-stage calls and no final output write occur. Earlier calls and any preliminary write remain observable.

### Transition sequence

If the queried value differs from the captured target, the transition proceeds in this order:

1. F-5c5958cfd5fe returns the first saved 32-bit value.
2. The transition-setting global is read and supplied to F-e2d95185a0cc.
3. F-000000000003 is dispatched through the first primary service’s table at byte offset 0x1DC, receiving that service explicitly.
4. F-000000000004 is dispatched through the retained controller’s table at byte offset 0x50, receiving the controller, the initially captured target, and zero.
5. F-e2d95185a0cc receives the first saved value with bit 14 set.

The transition-setting read occurs after the first F-5c5958cfd5fe call returns. It is not an entry-time snapshot.

Setting bit 14 preserves all other bits of the first saved value. If that bit is already set, the argument is unchanged. This operation does not establish that an earlier state is restored.

### Payload construction and submission

The next sequence is:

1. F-8157a180d901 receives 120, equivalently 0x78.
2. Its opaque 32-bit result is passed to F-1e30c9d0c388 together with a reference to textual data identified as **CntrlHistoryFn**.
3. F-000000000005 is dispatched through the retained controller’s table at byte offset 4, receiving the controller and the result of F-1e30c9d0c388.

The submitted value is the second dependency’s result. The function does not establish that it equals the value returned by F-8157a180d901.

Neither intermediate result is checked for zero or another failure indication. Allocation, construction, and ownership-transfer interpretations remain inferred rather than established.

### Second service acquisition and packed result

The function then:

1. Calls F-17fe47085641 again, obtaining a second primary service reference.
2. Calls F-5c5958cfd5fe again, retaining a second saved 32-bit value.
3. Calls F-e2d95185a0cc with exactly 0x10.
4. Dispatches F-000000000006 through the second primary service’s table at byte offset 0x19C, receiving that service explicitly.
5. Calls F-d2bc4a2e390c with the second saved value as both explicit arguments.

The second service reference is used for this dispatch. It is not assumed to equal the first reference.

Both arguments to F-d2bc4a2e390c come from the saved value obtained before the call supplying 0x10 and before the subsequent service dispatch. They are not refreshed after either intervening call.

### Packed-result decision

F-d2bc4a2e390c returns a 64-bit result, interpreted here as:

- **Low portion:** bits 0–31.
- **High portion:** bits 32–63.

The function returns without the final direct write exactly when:

- The low portion is nonzero; and
- Bit 46 of the full result is clear.

Bit 46 corresponds to bit 14 of the high portion. A low portion of 0x80000000 is nonzero and satisfies the first condition.

In every other case, the function:

1. Reads the output-base global.
2. Uses byte offset 4 from that base as the destination.
3. Writes the high portion with bit 14 cleared, as a 32-bit value.
4. Returns.

Thus, a zero low portion always leads to the final write, regardless of bit 46. A set bit 46 also leads to the write, regardless of the low portion.

This is a replacement write. The destination’s prior contents are not read by the visible operation. The output base is obtained after F-d2bc4a2e390c returns, so dependency changes to that global can affect the destination.

## Side effects

### Direct writes

At most two direct writes are visible:

| Condition | Destination | Width | Value |
|---|---|---:|---|
| The separately evaluated target and comparison observations are equal | F-fdd8589eca2d result plus 0x80 bytes | 16 bits | 6 |
| The transition runs and the packed-result no-write condition is false | Then-read output base plus 4 bytes | 32 bits | High result portion AND 0xFFFFBFFF |

The first condition does not depend directly on the initially retained target. The first write can occur even when the later queried value equals that retained target and the transition is skipped.

The second write cannot occur when the queried value equals the retained target.

### Dependency effects and retained state

Dependencies may have additional effects not visible in this function. No dependency is assumed to preserve globals, object contents, dispatch tables, or object identity.

The function retains the original target and reuses it for the later comparison and controller dispatch. It likewise retains the controller reference and saved values for their specified uses. Retaining a reference does not establish that the referenced object or its table remains unchanged.

Dispatch targets are obtained through the relevant objects’ tables at their respective call sites; a fixed target across calls is not established.

### Ownership and failures

No explicit release, deallocation, rollback, or cleanup operation is visible.

The evidence does not establish:

- Whether the size-taking dependency allocates memory.
- Whether the construction-like dependency returns its input address.
- Whether submission transfers ownership.
- Whether any dependency retains or releases memory.
- Who owns returned service objects.
- How failures, exceptions, or nonlocal exits affect resources.

If a dependency does not return normally, the later sequence is not established. No visible mechanism reverses earlier effects.

## Dependencies

Resolved dependencies use these opaque identities:

| Function ID | Observed role |
|---|---|
| F-fdd8589eca2d | Supplies the preliminary-write address |
| F-17fe47085641 | Supplies the first service reference and, on the transition path, a second service reference |
| F-76b1c50b12f7 | Supplies the retained controller reference |
| F-5c5958cfd5fe | Supplies two separately saved 32-bit values on the transition path |
| F-e2d95185a0cc | Receives the transition-setting observation, then the first saved value OR 0x4000, then 0x10 |
| F-8157a180d901 | Receives 120 and produces the first payload intermediate |
| F-1e30c9d0c388 | Receives that intermediate and the textual-data reference |
| F-d2bc4a2e390c | Receives two copies of the second saved value and produces the 64-bit result |

Unresolved indirect call sites are identified separately:

| Call-site ID | Table byte offset | Explicit arguments and result use |
|---|---:|---|
| F-000000000001 | 0x14 | Controller; result supplies the query object |
| F-000000000002 | 0x68 | No explicit arguments; produces the queried value |
| F-000000000003 | 0x1DC | First service; result unused |
| F-000000000004 | 0x50 | Controller, captured target, zero; result unused |
| F-000000000005 | 0x04 | Controller, constructed payload; result unused |
| F-000000000006 | 0x19C | Second service; result unused |

These indirect IDs identify call sites, not resolved runtime functions. They do not imply that the callees are distinct. Offsets are within dispatch tables, not direct field offsets within the objects.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Each assumes valid accesses and normal return from executed dependencies.

### Matching current value with stable preliminary observations

Suppose the captured target is 17, the separately evaluated target observation is 17, and the comparison observation is 17.

The preliminary dependency is called and the 16-bit value 6 is written. If the queried current value is also 17, the function then returns after the query. No transition-stage call or final output write occurs.

### Preliminary equality differs from the captured target

Suppose the captured target is 17, while the preliminary equality observes target 29 and comparison 29.

The preliminary write occurs because those two equality observations match. If the queried value is 17, the transition is skipped. If it is 29, the transition runs and F-000000000004 receives 17 as its target argument.

Conversely, with captured target 17, separately observed target 29, and comparison observation 17, the preliminary write is skipped.

These examples distinguish read roles. They do not establish a mechanism or memory model permitting the observations to differ.

### A later dependency changes the target global

Suppose the captured target is 17 and a dependency later changes the target global to 29.

The current-value comparison still uses 17. A queried value of 29 causes the transition to run, and the controller’s target argument remains 17. The preliminary decision was already made from its own separate observations.

### Setting the saved bit

If the first saved value is 0x00000025, its later argument to F-e2d95185a0cc is 0x00004025.

If it is already 0x00004025, the later argument remains 0x00004025.

### Nonzero low portion with clear flag

For low portion 1 and high portion 0x12340000, no final direct write occurs.

The same holds for low portion 0x80000000 and that high portion. The sign bit does not change a nonzero test.

### Zero low portion

For low portion zero and high portion 0x12344005, the final write stores 0x12340005.

If both portions are zero, the final write still occurs and stores zero.

### Set flag with nonzero low portion

For low portion 7 and high portion 0x00004003, the final write stores 3. The set high-portion bit 14 prevents the no-write return despite the nonzero low portion.

### Changed output base

If F-d2bc4a2e390c changes the output-base global and returns a result requiring a write, the destination uses the changed base plus 4 bytes. No earlier output-base snapshot is used.

## Uncertainties

The following remain **documented_only**:

- Application meanings of the globals, queried value, constants, and packed-result fields.
- Stability of the target global between its initial capture and separate equality evaluation.
- Evaluation order between the two preliminary equality operands.
- Volatility, concurrency, synchronization, aliasing, and machine-level load guarantees.
- Concrete indirect targets and calling conventions beyond the represented explicit arguments.
- Dependency effects on globals, retained objects, dispatch tables, and memory ownership.
- Allocation failure, exceptions, nonlocal exits, invalid-address behavior, and cleanup.
- Address arithmetic outside valid object ranges, including overflow behavior.
- Byte order and external serialization of the written fields.
- Full extent and encoding of the referenced textual data.
- Behavior outside the supplied representation, including regions reported as removed.

The lack of visible checks does not prove that failures are impossible or that execution continues normally after them.

## Evidence

This contract is based solely on static inspection of the supplied function representation and dependency map.

Established evidence includes the initial target capture, the separate target evaluation for preliminary equality, conditional 16-bit write, ordered dependency calls, comparison against the captured target, reuse of that target in controller dispatch, packed-result predicate, and conditional 32-bit replacement write.

The preliminary equality does not establish an evaluation order between its operands. Equality between its target observation and the initially captured target requires an additional stability assumption.

Controller/history, allocation, construction, and ownership-transfer terminology describes inferred intent only.

No dependency bodies were inspected. No runtime execution, smoke run, or tests were performed. The acceptance examples are conditional specifications, not observed results.
