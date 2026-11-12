<!-- Trusted provenance: F-3ac6367a0ace; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3ac6367a0ace

## Purpose

F-3ac6367a0ace coordinates conditional processing of an indexed collection and a linked list, updates selected state, and performs a shared completion protocol.

Its established contract concerns dependency interactions, conditional storage access, and state changes. Application meanings such as rendering, acquisition success, destruction, or resource release are not established.

Admission rejection, preparation rejection, an absent acquired object, and an enabled early-completion condition all bypass remaining main processing. They do not bypass completion. Every normal return follows the same completion protocol.

## Interface

### Invocation and results

The representation declares no parameters and no returned value, but consumes incoming execution context, including address-bearing values and frame-resident storage. The external calling convention is **documented_only**.

The represented integer words, opaque values, and addresses are 32 bits; byte accesses are eight bits. These are properties of the supplied representation, not independent ABI validation.

Operational inputs comprise:

- Addresses of primary, state, and feature objects.
- Addresses of status and positional-eligibility arrays.
- A 32-bit unsigned selection mask and suppression value.
- Extended control, auxiliary control, a list override, and a selector compared with 5.
- Secondary and completion resources.
- Two work-area addresses with different starting locations.

Observable context changes include replacement of an opaque result slot, establishment of a list override, and conditional suppression refresh. The result slot is not explicitly consumed afterward and is not established as a function-return channel.

### Storage contract

Offsets are byte offsets from the indicated object.

| Storage | Offset | Reached interpretation |
|---|---:|---|
| Primary | +0x04 | Signed 32-bit collection count |
| Primary | +0x0c | 32-bit distinguished index |
| Primary | +0x10 | List-head address; zero means empty |
| Primary | +0x39 | Unsigned suppression byte |
| State | +0x13 | Unsigned final-gate byte |
| State | +0x38 | Signed 32-bit recorded maximum |
| State | +0x3c | Opaque 32-bit associated value |
| State | +0x40 | 32-bit early-completion word |
| State | +0x48 | Signed 32-bit counter |
| State | +0x188 | 32-bit gate word |
| Feature | +0x0c | Unsigned 32-bit flags; bit 0x80 is relevant |
| Acquired object | +0x04 | Discriminator byte, compared with zero |
| List node | +0x18 | Discriminator byte, compared with 5 |
| List node | +0x20 | Next-node address; zero terminates traversal |

Both arrays contain signed 32-bit entries at four-byte spacing. Status entries correspond to collection positions. Eligibility entries correspond to visited list positions, including nodes that do not qualify.

Only reached accesses require usable storage. The operation establishes no bounds checks, alignment checks, object extents, nonoverlap guarantees, or lifetime management. Different work-area starts do not prove disjoint extents.

## Behavior

### Observation and routing guarantees

Inputs are not captured as one snapshot. Collection counts, list heads, feature flags, state gates, and controls are observed at their respective decision points. A later read cannot be replaced by an assumed entry value.

Two values are explicitly retained for later decisions or writes:

- The discriminator captured after a nonzero acquisition result.
- The result of F-cb297b788d6f, used for a possible associated-value update.

Changing the discriminator’s backing byte afterward does not change discriminator-dependent behavior. Dependencies have no established guarantee of preserving globals or reachable memory. These observation rules do not establish concurrent-mutation safety.

### Admission and acquisition contract

Entry requires, in order, F-2ea344827b50, F-887ba2404a49, F-2ea344827b50, F-887ba2404a49, and F-bb6565faa675. The last result replaces the context result slot.

Admission is controlled by two occurrences of F-7c66ac5392a6:

| Observation | Required consequence |
|---|---|
| First result nonzero | Completion without a second admission interaction |
| First result zero | F-887ba2404a49 before the second admission interaction |
| Second result nonzero | Completion without preparation |
| Both results zero | F-887ba2404a49 followed by F-a2b3e96577b2 |

After admission, a nonzero suppression observation requires F-0c702c9bd61f. Preparation then uses F-efc2c252d6ce with the first work-area address and the secondary resource as represented explicit inputs.

A nonzero preparation result selects completion. A zero result requires F-2ea344827b50 before acquisition through F-fc2a3dd9f0c0.

A zero acquisition result selects completion without reading an acquired-object field. For a nonzero result, the discriminator byte is captured before F-887ba2404a49. A captured zero additionally requires F-2ea344827b50. F-2399e3fb9a97 then receives the acquired address in either discriminator case.

The address being passed does not establish ownership transfer, retention, or continued validity.

### Initial state and list obligations

After F-2399e3fb9a97, feature bit 0x80 controls whether state +0x188 is read. A clear feature bit excludes that read. With the bit set and the gate zero, the counter at state +0x48 is read, incremented, and written before F-2ea344827b50. A nonzero gate excludes both the counter access and that interaction.

Initial list processing requires zero context suppression and a zero primary suppression byte. Nonzero context suppression excludes the byte read. This check does not replace context suppression.

When initial list processing is enabled:

- A retained zero discriminator requires F-d0e9fafa98a9 before override establishment.
- Selector 5 establishes override one without searching node discriminators.
- Any other selector establishes override zero and searches from the then-current head. The first node whose discriminator equals 5 establishes override one and ends the search.

The search reads a next link only for a nonmatching node. A matching node’s next link is not needed for that search. An empty list requires no node-field access.

Processing starts from a separate, fresh head read. Qualification and traversal obey these shared rules:

| Property | Obligation |
|---|---|
| Position | Starts at signed 32-bit zero for each processing traversal |
| Qualification | Nonzero override, otherwise a strictly positive positional eligibility entry |
| Override short circuit | Omits eligibility access and its offset calculation |
| Nonqualifying node | Still consumes a position |
| Advancement | Position advances before the current node’s next-link read |
| Link observation | Any qualifying dependency interaction precedes that next-link read |

Each qualifying node in initial processing requires F-71980e76ca3a followed by F-0b0cd8d831b7.

If initial processing is disabled, this operation does not establish a new override. The incoming override remains relevant to trailing processing.

### Discriminator-dependent transition

Extended control is observed before the transition. A nonzero observation requires F-2148ac0f693b.

The transition is required regardless of whether extended processing will run:

| Retained discriminator | Interaction obligation | Result-slot replacement |
|---|---|---|
| Zero | F-887ba2404a49 | Result of that interaction |
| Nonzero | F-211638bb75da, then F-7f44e0e8f855, then F-a2b3e96577b2 | Result of F-7f44e0e8f855 |

A separate extended-control observation determines whether the extended phase runs.

### Extended phase and selection-sensitive collection

The extended opening requires F-2ea344827b50, two occurrences of F-e4b8d56f2a5d, and three further occurrences of F-2ea344827b50, in that order. Zero auxiliary control then requires F-2ea344827b50; nonzero auxiliary control requires F-13d2f4206a76. F-cb297b788d6f follows, and its result is retained.

The collection starts at unsigned 32-bit position zero. Every continuation decision rereads the signed collection count and compares it with the signed interpretation of the position.

The distinguished index is compared unsigned in this phase. Its match has precedence over selection calculation and status access.

| Position category | Required interaction | Excluded access or calculation |
|---|---|---|
| Distinguished-index match | F-887ba2404a49 | Selection calculation and status read |
| Nondistinguished and excluded by selection | F-887ba2404a49 | Status read |
| Selected with negative status | F-2ea344827b50 followed by F-eec489a1f23b | — |
| Selected with nonnegative status | F-13d2f4206a76 | — |

Selection shifts a signed 32-bit one left by the low eight bits of the position and intersects the result with the unsigned mask. A zero intersection excludes the position unless the mask is all ones.

The shift is evaluated before the all-ones exception. Therefore, an all-ones mask does not make problematic shifts defined. A distinguished-index match does avoid the shift.

F-2ea344827b50 follows the collection. A zero suppression observation additionally requires F-ef2b1817173a followed by F-607c57cec83e.

### Maximum-associated update and early completion

The signed recorded maximum is compared with the signed collection count. Their relative read order is not established.

If the maximum is smaller, the count is read again. That fresh count is written to state +0x38 before the retained F-cb297b788d6f result is written to state +0x3c. Equality or a greater maximum excludes both writes.

The replacement count need not equal the count used for comparison. Under intervening mutation, this does not establish globally monotonic maximum tracking.

The early-completion word at state +0x40 is then read. Nonzero selects completion and excludes suppression refresh and all remaining main processing.

Otherwise, the extended closing requires F-2ea344827b50, two occurrences of F-65af7db35d2d, F-2ea344827b50, and F-a2b3e96577b2, in that order.

### Suppression refresh and unmasked collection

At this boundary, zero context suppression is replaced by the current primary suppression byte, zero-extended to 32 bits. Nonzero context suppression is retained without reading the byte.

The unmasked phase requires suppression to have been zero before refresh and to remain zero afterward. It opens with F-2ea344827b50. Nonzero auxiliary control adds F-c2e9c7166e37 followed by F-211638bb75da before collection processing.

Positions start at signed 32-bit zero, with a fresh signed count read at every continuation decision. The distinguished index is compared signed. The selection mask is not consulted.

| Position category | Required interaction |
|---|---|
| Distinguished-index match | F-887ba2404a49; no status read |
| Nondistinguished with negative status | F-2ea344827b50 followed by F-eec489a1f23b |
| Nondistinguished with nonnegative status | F-c2e9c7166e37 |

The closing order is F-4409597a74af, F-2ea344827b50, and F-8d59c555cc6f. A nonzero auxiliary-control observation then adds F-2ea344827b50. Both outcomes require F-a2b3e96577b2 followed by F-ffae10fb342b.

### Trailing processing and final gate

A fresh feature-bit check conditionally reads state +0x188. Feature bit 0x80 set with a zero gate requires F-2ea344827b50, but no counter increment.

Nonzero extended control adds F-2ea344827b50 followed by F-a2b3e96577b2. A separate occurrence of that pair is required unconditionally afterward.

Trailing list processing starts from the then-current head and uses the qualification and advancement rules specified above. Each qualifying node requires F-887ba2404a49. Suppression does not independently disable this traversal.

F-2ea344827b50 follows traversal. Nonzero extended control adds two further occurrences of that dependency.

The final gate tests feature bit 0x80, then state +0x188 for zero, then state byte +0x13 for zero. An earlier failed condition excludes later field reads. Satisfaction requires F-2ea344827b50, F-6f2595a8254b, and F-b3011fb31d40, in that order.

### Completion protocol

Every normal-return path requires the following ordered interactions:

| Order | Dependency | Represented explicit input |
|---:|---|---|
| 1 | F-20e21ec9b4c5 | Second work-area address |
| 2 | F-94cb53877c15 | Eligibility-array address |
| 3 | F-94cb53877c15 | None established for this occurrence |
| 4 | F-ed9979230848 | Completion resource |
| 5 | F-e5380ae57ea1 | None established |
| 6 | F-841dc33ee157 | Secondary resource |

There is no explicit returned value afterward. The second work area’s required preparation, including on early rejection paths, is **documented_only**.

## Side effects

Direct designated object writes are the conditional counter increment and maximum-associated pair. Context changes are result-slot replacement, override establishment, and suppression refresh. Dependencies and storage aliasing may produce additional effects.

The counter increment is not established as atomic. The maximum and associated-value writes are ordered but not atomic as a pair. No rollback, synchronization, or transactional grouping is established.

Arithmetic imposes additional preconditions:

- Incrementing a counter already at 2,147,483,647 has no established defined signed result.
- Selection shift counts 0–30 have representable signed results. Count 31 has no established portable signed result; counts 32–255 exceed the operand width.
- Selection-sensitive position increment and four-byte offset scaling use unsigned 32-bit arithmetic. Scaling can wrap; defined wrapping does not prove valid addressed storage.
- Unmasked position increment and offset scaling use signed arithmetic. The count bound protects the loop’s own increment, but a reached status access at position 536,870,912 already requires unrepresentable signed multiplication by four.
- List processing has no count bound or cycle detection. Signed position advancement can overflow even with override enabled. With override zero, eligibility access also requires representable signed scaling.

For either collection pass, a stable nonpositive count produces no indexed iterations. Under defined progression, the signed count bound prevents the collection position from incrementing beyond 2,147,483,647.

Invalid storage, invalid address formation, problematic arithmetic, nonterminating traversal, or a nonreturning dependency can prevent normal return. No explicit error channel addresses these conditions. Routing results do not establish application-level error categories.

No interaction establishes allocation, release, ownership transfer, or preservation of storage required later.

## Dependencies

All dependency IDs are opaque. Their required interactions are specified above; names carry no application semantics.

| Dependency | Established result use |
|---|---|
| F-bb6565faa675 | Replaces the context result slot |
| F-7c66ac5392a6 | Zero permits the corresponding admission boundary |
| F-efc2c252d6ce | Zero permits acquisition |
| F-fc2a3dd9f0c0 | Supplies an acquired address or zero |
| F-887ba2404a49 | Replaces the result slot at the zero-discriminator transition |
| F-7f44e0e8f855 | Replaces the result slot |
| F-cb297b788d6f | Supplies the possible associated-value write |

Other results have no explicit consuming use here. This does not make those interactions effect-free.

Represented explicit inputs are established only where identified above. In particular, node-processing interactions are not shown receiving the current node address explicitly. An occurrence without represented explicit inputs does not establish a parameterless interface.

Complete signatures, hidden inputs, global effects, ownership effects, exceptional exits, termination, and preservation guarantees are **documented_only**.

## Acceptance examples

These are conditional acceptance requirements, not executed tests. They assume usable reached storage, defined arithmetic, and normally returning dependencies. Relevant values remain stable unless mutation is explicitly described.

1. **First admission rejection.** A nonzero first admission result permits no second admission interaction, preparation, acquisition, or main processing. Entry interactions and the complete completion protocol remain required.

2. **Absent acquisition.** A zero acquisition result excludes the discriminator read and F-2399e3fb9a97, then selects completion.

3. **Retained discriminator.** If zero is captured and the backing byte later becomes nonzero, the zero-discriminator transition remains required.

4. **Forced list qualification.** Enabled initial processing with selector 5 establishes override one without a discriminator search. Processing traversals omit eligibility reads and scaling, but retain position advancement and next-link reads.

5. **Search termination.** A matching search node’s next link is not read during the search. The separate processing traversal may subsequently read that link.

6. **Positional eligibility.** With override zero, zero and negative eligibility entries suppress node interactions, not position advancement. Later nodes use their visited positions.

7. **Preserved incoming override.** Disabled initial processing does not reset override. A nonzero incoming override can qualify all trailing nodes without eligibility access.

8. **Selection precedence and width.** A distinguished position excludes selection calculation and status access. At nondistinguished position 31, even an all-ones mask does not establish defined signed-shift behavior.

9. **Different nonnegative-status behavior.** A selected nondistinguished nonnegative status requires F-13d2f4206a76 in the selection-sensitive pass and F-c2e9c7166e37 in an enabled unmasked pass.

10. **Maximum and early completion.** Equality excludes both maximum-associated writes. A smaller maximum requires a fresh count read, then the maximum write, then the associated-value write. A subsequent nonzero early-completion word excludes all remaining main phases.

11. **Suppression refresh.** A zero primary byte can permit initial processing, while a later nonzero byte becomes context suppression and excludes the unmasked phase. That exclusion alone does not exclude trailing processing.

12. **Short-circuit gates.** A clear feature bit excludes the corresponding state-gate read. A nonzero final state gate excludes the final-byte read.

13. **Changed next link.** If a qualifying interaction changes the current node’s next link and returns, traversal uses the subsequently read link rather than a previously assumed successor.

## Uncertainties

- **documented_only — Invocation:** External ABI, context construction, and any external interpretation of the result slot.
- **documented_only — Storage:** Object extents, alignment, work-area initialization, ownership, overlap restrictions, and lifetimes across interactions.
- **documented_only — Dependencies:** Complete interfaces, hidden state, preservation, ownership effects, exceptional exits, and nontermination.
- **documented_only — Arithmetic:** Target-specific outcomes for problematic signed shifts, signed overflow, and invalid address formation.
- **documented_only — Mutation:** Permitted concurrency and synchronization requirements. Repeated observations do not themselves authorize concurrent mutation.
- **Inferred intent:** The maximum-associated pair resembles high-water tracking, and completion resembles finalization. Neither interpretation establishes application semantics or resource release.

## Evidence

Established behavior derives from static inspection of the supplied representation and opaque dependency mapping: widths, storage offsets, branch reachability, short-circuit omissions, interaction order, represented explicit inputs, consumed results, retained values, repeated reads, and designated writes.

Arithmetic boundaries follow from the represented widths and signedness. In particular, an unsigned mask does not make the preceding signed shift unsigned, and collection-count bounds do not validate every offset or address.

Prior prose and reviewer feedback are not independent behavioral evidence. No runtime tests, execution traces, ABI validation, ownership checks, or target-specific arithmetic experiments have been performed.
