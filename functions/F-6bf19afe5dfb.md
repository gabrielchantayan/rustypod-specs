<!-- Trusted provenance: F-6bf19afe5dfb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6bf19afe5dfb

## Purpose

F-6bf19afe5dfb populates three plane-reference fields in a mutable state object, optionally invokes a descriptor-selected initializer, and obtains a downstream value that it stores and passes to an attachment dependency.

The function supports two storage arrangements:

- Three separately requested buffers.
- Three references calculated from one candidate base and two dimension-derived sizes.

Selection depends on an incoming byte flag, dependency results, and, when the incoming flag is nonzero, a later reread of that flag.

The function performs incremental updates. It does not visibly release previously stored references, validate buffer-producing results, or roll back earlier writes after a later failure.

“Plane,” “buffer,” “creation,” and “attachment” describe apparent usage. Image-format semantics, allocation contracts, and downstream object types are not established by the supplied static evidence.

## Interface

### Invocation and result

The function accepts one 32-bit address identifying a mutable state object. It has no explicit return value. Its observable outputs are direct memory writes and dependency invocations.

No local checks establish that the state address, associated descriptor, derived addresses, or returned references are valid. Normal completion requires the selected memory accesses and dependency calls to complete successfully. Invalid memory and dependency failures have no locally specified recovery behavior.

### State layout

All offsets are byte offsets from the supplied state address.

| Offset | Access width | Role |
|---|---:|---|
| `0x10` | 32 bits | Associated descriptor reference; read at entry and again after plane-reference population. |
| `0x18` | 32 bits | Destination for the result of F-798fb9f4fe09. |
| `0x28` | 32 bits | Dimension A. |
| `0x2C` | 32 bits | Dimension B. |
| `0x30` | 32 bits | First plane reference. |
| `0x34` | 32 bits | Second plane reference. |
| `0x38` | 32 bits | Third plane reference. |
| `0x49` | 8 bits | Storage request flag, directly replaced with zero or one. |
| `0x4C` | Extent unknown | Beginning of a region whose address may be passed to F-58ddb7b0d1b5. |

The descriptor has these relevant fields:

| Offset | Access width | Role |
|---|---:|---|
| `0x08` | 8 bits | Optional-initialization selector. |
| `0x0C` | 32 bits | Initial dimension B. |
| `0x10` | 32 bits | Initial dimension A. |

The evidence does not establish a complete structure layout, alignment requirement, or minimum extent for the region beginning at state offset `0x4C`. Passing that region’s address does not establish whether the recipient reads, modifies, retains, or ignores it.

### Arithmetic and argument widths

The represented address and word operations are 32-bit. Dimension copies, plane-reference stores, and the downstream-result store preserve 32-bit values.

For this contract:

- **N** is the low 32 bits of the product of the dimension B and dimension A fields at the size-computation stage.
- **Q** is **N** shifted right logically by two bits, equivalent to unsigned division by four with the remainder discarded.
- Adjacent-plane address calculations use 32-bit addition, with wraparound and no visible overflow check.

There is no local rejection of negative-looking dimension bit patterns, multiplication overflow, nonmultiples of four, zero sizes, or wrapped addresses. These statements describe the represented operations, not a guarantee that resulting dependency calls or memory references are valid.

Argument positions below are established. The supplied representation does not establish an evaluation order among separate field reads used to form arguments to the same call.

## Behavior

### Dimension transfer

The function first reads the descriptor reference from state offset `0x10` and obtains dimension A from descriptor offset `0x10`. It retains that value.

It then accesses the descriptor reference through the state again, obtains dimension B from descriptor offset `0x0C`, and writes dimension B to state offset `0x2C`. It subsequently writes the retained dimension A to state offset `0x28`.

The direct write order is therefore dimension B, then dimension A. Dimension A is captured before either write. The operation is not an atomic snapshot of the descriptor. If descriptor and state storage overlap, effects must follow these accesses rather than an assumed independent copy.

### Storage-request resolution

A retained candidate base starts at zero.

If the incoming byte at state offset `0x49` is zero, the function skips all three storage-mode dependencies and writes zero to that byte.

If the incoming byte is nonzero:

1. F-e9a871f6149b is called without explicit arguments. Its result is discarded.
2. F-6412127cad82 is called without explicit arguments.
3. If its result is not exactly `1`, F-e9a871f6149b is called again.
4. On that conditional path, the second F-e9a871f6149b result and the address of state offset `0x4C` are passed, in that order, to F-58ddb7b0d1b5. Its result becomes the retained candidate base.
5. After the calls, the function rereads the flag byte. It writes `1` only when that reread is nonzero and the retained candidate base is nonzero. Otherwise it writes `0`.

An incoming nonzero value need not equal one. The comparison against the F-6412127cad82 result is specifically equality with `1`, not a generic zero/nonzero decision.

If F-6412127cad82 returns exactly `1`, candidate acquisition is skipped. The candidate remains zero, so the function writes a zero flag regardless of the flag reread.

A nonzero candidate is insufficient if a dependency has cleared the flag. No explicit release of such a rejected candidate occurs.

The function does not establish that these dependencies preserve dimensions, the descriptor reference, the flag, or other reachable storage.

### Size computation and plane-reference population

After writing the normalized flag, the function reads the dimension fields and computes **N**. Earlier dependencies may have changed those fields since their initial transfer. **N** is retained for the subsequent plane operations; **Q** is derived from that retained value.

The function then reads the flag to choose the arrangement.

#### Separate-buffer arrangement

When the flag is zero, the function performs:

| Order | Dependency call | Direct write after return |
|---:|---|---|
| 1 | F-638beded7699 with **N**, `0x35` | Returned 32-bit value to `0x30`. |
| 2 | F-638beded7699 with **Q**, `0x35` | Returned 32-bit value to `0x34`. |
| 3 | F-638beded7699 with **Q**, `0x35` | Returned 32-bit value to `0x38`. |

All three calls occur on normal completion, including when requested sizes or earlier results are zero. Sizes are not recomputed from dimensions between calls.

Each result is stored without validation. There is no local failure branch, rollback, or explicit release of an earlier result.

These stores do not guarantee that all three returned values remain in their fields. For example, the second or third dependency invocation can modify a previously written field. Later consumers read the then-current fields.

#### Adjacent-plane arrangement

When the flag is nonzero, the direct stores occur in this order:

| State offset | Value |
|---|---|
| `0x30` | Retained candidate base. |
| `0x34` | Candidate base plus **N**, modulo 2³². |
| `0x38` | Candidate base plus **N** plus **Q**, modulo 2³². |

No F-638beded7699 call occurs on this path.

Lengths of **N**, **Q**, and **Q** would imply a total mathematical extent of **N + 2Q**, but the function does not verify that this capacity exists. The base-producing call does not receive these sizes as explicit arguments.

### Optional descriptor-selected initialization

After writing the third plane reference, the function reloads the descriptor reference from state offset `0x10`. This can differ from the descriptor used for the initial dimension transfer.

The function first reads the descriptor selector byte and compares it with `3`.

- If equal to `3`, it calls F-6d4e5f4e1b57 with the captured descriptor reference and `5`, then calls F-c0ab5b5d32b8.
- Otherwise, it reads the selector byte again and compares it with `2`. If equal to `2`, it calls F-6d4e5f4e1b57 with that descriptor reference and `4`, then calls F-28199ab2af9a.
- If neither comparison selects a branch, it skips both initializer paths.

The two selector comparisons are separate reads. There is no intervening explicit dependency call, but the evidence does not establish concurrency or external-memory stability.

Either selected initializer receives six arguments:

1. Then-current dimension A.
2. Then-current dimension B.
3. The retained result of F-6d4e5f4e1b57.
4. Then-current first plane reference.
5. Then-current second plane reference.
6. Then-current third plane reference.

The dimensions and plane references are read after F-6d4e5f4e1b57 returns. Consequently, changes made by that dependency affect the forwarded field values. Its returned value is not locally validated. The initializer result is unused.

An unrecognized selector does not prevent downstream creation and attachment.

### Downstream creation and attachment

The function calls F-798fb9f4fe09 with nine arguments:

| Position | Value |
|---:|---|
| 1 | `0` |
| 2 | Current dimension B |
| 3 | Current dimension A |
| 4 | Current dimension A, read for this argument position |
| 5 | `2` |
| 6–9 | `0` each |

The repeated dimension A appears in two argument positions. Their semantic roles are unknown; the representation does not establish a single shared snapshot for the two reads.

The returned 32-bit value is retained and written to state offset `0x18`. The function then invokes F-aaa12137cb38 with:

1. The retained creation result.
2. Then-current first plane reference.
3. Then-current second plane reference.
4. Then-current third plane reference.
5. `0`.

The first argument is not reloaded from state offset `0x18`. A mutation of that field by F-798fb9f4fe09 does not replace the returned value used for this store or attachment argument.

In contrast, plane references are read after creation returns. Changes made by the creation dependency can therefore affect attachment arguments.

Neither result is checked. After attachment returns, this function returns without an explicit result.

## Side effects

The function directly writes dimension B, dimension A, the normalized flag, all three plane-reference fields, and the downstream-result field, in the stages described above.

These updates are not transactional. No original field values are saved for rollback. There is no explicit release of previous plane references or a previous downstream value.

Zero buffer-producing results cause no local failure branch and are stored without validation. This does **not** guarantee that a later initializer or attachment call receives zero in that plane position: intervening dependencies may overwrite the field. Those consumers receive the then-current plane-field values.

The creation result has different dataflow. Its retained value is both stored at `0x18` and forwarded directly as the first attachment argument. Thus, if creation returns zero, the direct store and first attachment argument are zero, regardless of changes creation made to the state field. The attachment dependency may subsequently modify state; the function does not restore it afterward.

Dependency effects are otherwise unspecified. The visible calls do not establish ownership transfer, borrowed-reference duration, retained-reference behavior, allocation provenance, or responsibility for releasing memory. Absence of an explicit release in this function does not prove that dependencies perform no releases.

## Dependencies

| Opaque ID | Established use |
|---|---|
| F-e9a871f6149b | Called once for a nonzero incoming flag and conditionally a second time; only the second result is forwarded. |
| F-6412127cad82 | Result compared with exactly `1` to decide whether to obtain a candidate base. |
| F-58ddb7b0d1b5 | Receives the second F-e9a871f6149b result and the address at state offset `0x4C`; returns a candidate base. |
| F-638beded7699 | Three separate-buffer calls with sizes **N**, **Q**, **Q**, each paired with `0x35`. |
| F-6d4e5f4e1b57 | Receives a captured descriptor reference and `5` or `4`; result is retained for the selected initializer. |
| F-c0ab5b5d32b8 | Six-argument initializer selected by a selector read equal to `3`. |
| F-28199ab2af9a | Six-argument initializer selected by the second selector read equal to `2`. |
| F-798fb9f4fe09 | Returns the retained value stored at `0x18` and forwarded to attachment. |
| F-aaa12137cb38 | Receives the retained creation result, three current plane references, and zero. |

These roles do not establish complete dependency interfaces or preservation guarantees.

## Acceptance examples

These are conditional static acceptance scenarios, not executed tests. Except where mutation is explicitly described, assume dependencies return normally and preserve relevant state, and memory remains stable between reads.

### Separate buffers and selector 3

Given dimension A `8`, dimension B `6`, incoming flag zero, and selector `3`:

- Dimension B is written as `6`, followed by dimension A as `8`.
- The flag is written as zero.
- **N = 48** and **Q = 12**.
- F-638beded7699 receives size arguments `48`, `12`, `12`, each with `0x35`; its results are stored in plane order.
- F-6d4e5f4e1b57 receives the descriptor and `5`.
- F-c0ab5b5d32b8 receives `8`, `6`, the descriptor-data result, and the three current planes.
- Creation receives `(0, 6, 8, 8, 2, 0, 0, 0, 0)`.
- Its returned value is stored and forwarded to attachment with the current planes and zero.

### Adjacent planes

Given dimensions `8` and `6`, a nonzero incoming flag, F-6412127cad82 returning `0`, and candidate base `0x1000`, with the flag remaining nonzero:

- F-e9a871f6149b is called twice.
- The normalized flag is `1`.
- Direct plane stores are `0x1000`, `0x1030`, and `0x103C`.
- No separate-buffer call occurs.

### Candidate rejection

If F-6412127cad82 returns exactly `1`, neither the second F-e9a871f6149b call nor candidate acquisition occurs. The function writes a zero flag and takes the separate-buffer path.

If candidate acquisition returns nonzero but clears the flag, the same path is selected. No explicit candidate release occurs.

### Zero buffer result overwritten before consumption

Suppose separate-buffer mode is selected, the first buffer call returns zero, and the second buffer call overwrites the first plane field with `0x2000`.

The function first stores zero at `0x30`. It does not branch on that result. If no later operation changes that field, initialization and attachment receive `0x2000` for the first plane, not the earlier zero result.

Likewise, a descriptor-data or creation dependency may change plane fields before the corresponding consumer reads them.

### Retained creation result

Suppose creation changes state offset `0x18` to `0x9000`, changes the first plane field to `0x3000`, and returns zero.

The function overwrites `0x18` with zero. Attachment receives zero as its first argument and `0x3000` as its first-plane argument, assuming no intervening external changes. The creation dependency’s write of `0x9000` does not determine the first attachment argument.

### Truncation, overflow, and selector bypass

For dimensions `3` and `3`, **N = 9** and **Q = 2**. Secondary sizes are truncated, not rounded up.

For dimension bit patterns `0x00010000` and `0x00010000`, **N = 0** and **Q = 0**. Separate-buffer mode makes three zero-size requests. Adjacent mode directly stores the same candidate base into all three plane fields.

For a stable selector of `7`, no descriptor-data or initializer dependency is called. Creation and attachment still occur.

## Uncertainties

The following remain **documented_only**:

- Dimension meanings, valid ranges, and signed interpretation.
- Actual data format, plane lengths, stride, padding, and capacity requirements.
- Meaning of `0x35`, the creation constants, and repeated dimension arguments.
- Allocation, alignment, failure, ownership, and release contracts.
- Extent and use of the region beginning at `0x4C`.
- Dependency mutation of state, descriptors, buffers, or globally reachable memory.
- Behavior on invalid addresses, zero references, wrapped addresses, or dependency non-return.
- Supported aliasing relationships, concurrency guarantees, and repeated-invocation safety.
- Final state after dependency side effects, beyond the direct writes and retained-value forwarding established here.

## Evidence

Established static evidence comprises the supplied function’s access widths, field offsets, explicit writes, branches, 32-bit arithmetic, dependency call order, argument positions, and distinctions between retained results and later field reads.

Buffer allocation, image-like organization, initialization, creation, and attachment are inferred usage descriptions, not independently verified dependency semantics.

No runtime tests have been performed. The acceptance examples describe conditional consequences of the supplied static evidence, not observed execution.
