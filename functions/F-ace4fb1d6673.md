<!-- Trusted provenance: F-ace4fb1d6673; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ace4fb1d6673

## Purpose

F-ace4fb1d6673 coordinates processing of a counted array of 32-bit words into destination storage. It selects a processing dependency from the retained input count, requests temporary descriptors, conditionally requests storage growth, and locally finalizes result metadata.

For positive input counts, the provisional output count is twice the input count. Local finalization clears one metadata field and may reduce the provisional count by one after inspecting its highest word.

**Inferred intent:** The doubled count, specialized processing sizes, and high-word adjustment suggest multiprecision squaring. The supplied body does not establish the numerical operation performed by the processing dependencies. Mathematical squaring is therefore not an acceptance requirement of this contract.

Identical source and destination descriptor addresses trigger a temporary-destination request. This does not establish support for arbitrary overlapping data buffers.

**Observation boundary:** Positive-path count and metadata writes describe the effective destination immediately after local finalization. An optional subsequent F-cec6695e396c call and the mandatory F-4bd8a1faf697 call have unknown descriptor-preservation and propagation behavior. The caller-visible descriptor state at return is not established solely by those local writes.

## Interface

### Arguments and descriptor layout

The function accepts:

1. A destination descriptor address.
2. A source descriptor address.
3. An opaque 32-bit context token.

The represented address model is 32-bit. Descriptor accesses establish this layout:

| Byte offset | Width | Established use |
|---|---:|---|
| 0 | 32 bits | Data address for word storage |
| 4 | 32 bits | Count, interpreted as signed at entry |
| 8 | 32 bits | Capacity-like field used in size comparisons |
| 12 | 32 bits | Metadata field cleared during positive-path local finalization |

Word storage is accessed in four-byte units. The numerical significance and ordering of those words are not established.

“Capacity,” “context,” “scratch,” and “transfer” are descriptive labels for observed use, not complete dependency contracts. In particular, the field at offset 12 is not proven to represent sign.

The source count is read once at entry and retained. Dispatch, size calculations, and the high-word displacement use that retained value even if dependencies subsequently modify the source descriptor.

Other descriptor fields are read later, at their points of use. Their entry values are not snapshots guaranteed to survive intervening calls.

### Return value

The represented return type is pointer-shaped, but the function locally selects only:

- Address-valued **1** for success.
- Address-valued **0** for an explicitly detected failure.

These values are status indicators, not result-storage addresses.

A nonpositive input count returns success after setting the destination count to zero. Positive-path success means execution reached the success exit and its final dependency returned normally. It does not certify numerical correctness, preservation of locally written metadata, or a checked successful result from processing or transfer dependencies.

### Preconditions

The following are needed for valid execution; the function does not comprehensively check them:

- The source count field is readable at entry.
- The destination count field is writable on the nonpositive path.
- On positive paths, descriptors and storage remain valid whenever accessed after dependency calls.
- The effective destination supports accesses through offset 12.
- Data addresses and accessible storage satisfy the selected processing dependency’s requirements.
- Capacity-like fields describe sufficient usable storage when relied upon to skip growth.
- Arithmetic and address calculations remain valid in the applicable execution model.

No independent validation establishes source storage capacity, data-pointer validity, arbitrary overlap safety, or an upper bound on the positive input count.

## Behavior

### Entry and nonpositive counts

The first observable descriptor access reads the source’s signed 32-bit count.

If that count is less than one, the function writes zero to the caller’s destination count and returns status 1. It calls no dependency and explicitly changes no other destination field.

Zero and negative counts take the same path. For identical descriptor addresses, the count is read before being overwritten.

### Positive-path setup

For a retained positive count \(n\), the function first calls F-ea92bf179106 with the context token.

It then selects an effective destination:

- Distinct source and destination descriptor addresses: retain the caller’s destination address.
- Identical addresses: call F-834e12b6dfde with the context token and use its result as the effective destination.

The function next calls F-834e12b6dfde with the context token to obtain a scratch descriptor. This request occurs on every positive path, including paths whose processing call receives no scratch argument.

A null scratch result causes the common failure exit: call F-4bd8a1faf697 with the context token, then return status 0.

The temporary-destination result has no separate immediate null check. If it is null while the subsequent scratch request succeeds, the later capacity access through the effective destination is unprotected. No safe failure return is established for that case.

These setup dependencies may modify descriptors or shared state. Consequently, an entry-time capacity value does not establish the value seen by the later capacity check.

### Destination capacity handling

The function forms a provisional count \(2n\) and a destination capacity request \(2n+1\).

After setup, it reads the effective destination’s capacity-like field and compares it with \(2n+1\) using a signed comparison. If insufficient, it calls F-08bc2fc94856 with the effective destination and the request.

A 64-bit carrier is used for the subsequent checks and large-input dispatch:

- Without growth, its low 32 bits contain the effective destination address and its high 32 bits contain \(2n+1\).
- With growth, it contains the represented result of F-08bc2fc94856.

A zero low half causes the common failure exit. A nonzero low half permits processing to continue.

The growth result does **not** replace the effective destination address. Later destination accesses still use the previously selected descriptor. Thus, a nonzero replacement address alone is insufficient to establish usable destination storage; the original effective descriptor must remain valid and describe suitable storage.

The high half of this carrier is retained for the large-input selector. Its meaning after growth is unknown.

### Processing selection

Selection uses the retained entry count, not the current source count.

| Retained count | Selected dependency | Arguments |
|---|---|---|
| 4 | F-9d4416a58736 | Destination data address, source data address |
| 8 | F-888bcfb33b92 | Destination data address, source data address |
| Other positive counts below 16 | F-78a03e109d74 | Destination data address, source data address, retained count |
| At least 16 | Further selection described below | Depends on derived candidate |

The count-4, count-8, and small-general branches do not inspect or grow scratch capacity.

Data addresses are read after preceding setup and growth calls. For the small-general path, the supplied body explicitly reads the source data address before the destination data address. The same explicit order occurs on the large nonmatching path before its general-processing call. The argument expressions for other processing calls do not establish an independent relative read order.

No processing return value is checked.

### Large-input selection and scratch handling

For \(n \geq 16\), F-f7e02acefeaa receives:

1. The retained count \(n\).
2. The high 32 bits of the destination-capacity carrier.

The second argument is \(2n+1\) only when destination growth was skipped. After growth, it is the high half supplied by F-08bc2fc94856.

The selector result is used to derive a candidate: subtract one, retain the low eight bits as a shift count, and left-shift a one-valued operand. This establishes the represented expression, not portable behavior for every possible selector result. Counts outside the valid shift range remain unresolved.

**Candidate equals \(n\):**

- Read scratch capacity after the selector call.
- Compare it with a four-times-candidate request. The sign-and-overflow expression represents signed less-than comparison for the 32-bit operands.
- If insufficient, call F-08bc2fc94856 with the original scratch descriptor and that request.
- Check the resulting scratch-success value for null.
- On null, take the common failure exit.
- Otherwise call F-8dc449ebe358 with destination data address, source data address, \(n\), and the original scratch descriptor’s current data address.

A growth result in this branch is used only for the null check. It does not replace the descriptor from which the scratch data address is read.

**Candidate differs from \(n\):**

- Read scratch capacity after the selector call and compare it, as signed 32-bit data, with \(2n\).
- If insufficient, call F-08bc2fc94856 with the scratch descriptor and \(2n\), replacing the local scratch reference with its represented result.
- Check that reference for null; a null value causes the common failure exit.
- Otherwise call F-78a03e109d74 with destination data address, source data address, and \(n\).

The general-processing call receives no explicit scratch argument. Whether scratch preparation has an indirect effect through shared state is unknown.

### Local metadata finalization

After the selected processing dependency returns normally, the function:

1. Writes \(2n\) to the effective destination count.
2. Writes zero to its field at offset 12.
3. If the computed doubled count is positive as signed 32-bit data, reads the effective destination’s current data address and inspects the 32-bit word at byte displacement \(8n-4\).
4. If that word is zero, writes \(2n-1\) to the effective destination count.

For ordinary nonoverflowing arithmetic, the inspected word has zero-based index \(2n-1\).

Only one word is inspected. The local count is reduced by at most one, even if several high words are zero. This is not a full leading-zero normalization.

The processing dependency may have changed descriptor fields. Finalization uses the retained effective descriptor address but reads its data address after processing.

Immediately after these local operations, the effective destination has the count and cleared metadata described above, assuming valid ordinary memory accesses. That state is not an unconditional return-time guarantee.

### Transfer and exit

After local finalization, if the effective destination address differs from the caller’s destination address, the function calls F-cec6695e396c with:

1. Caller destination.
2. Effective destination.

This condition uses the selected addresses, not merely whether the original source and destination were identical. A temporary acquisition is not independently proven to return a distinct descriptor.

No transfer return value is checked.

The function then selects success, calls F-4bd8a1faf697 with the context token, and returns status 1 if that call returns normally.

Every explicitly checked positive-path failure also calls F-4bd8a1faf697 before returning status 0. This cleanup ordering does not cover invalid accesses, nonlocal exits, or dependency failures not represented by the checked zero values.

## Side effects

### Direct effects

On the nonpositive path, the only explicit write is destination count zero.

On positive-path completion, direct writes affect the effective destination’s count and offset-12 metadata. Output-word production is delegated to processing dependencies.

The broad observable sequence is:

1. Read entry count.
2. Enter context.
3. Acquire an effective destination if required.
4. Acquire scratch.
5. Inspect and possibly grow destination storage.
6. Perform any large-input selection and scratch preparation.
7. Invoke processing.
8. Finalize effective-destination metadata and inspect at most one output word.
9. Invoke transfer if the effective destination differs.
10. Exit context.
11. Return status.

Capacity and data-address observations belong to their positions in this sequence. No unknown dependency is assumed to preserve earlier descriptor values.

### Ownership and failure state

The body directly exposes no allocation or release primitive. Temporary acquisition does not establish ownership, lifetime, uniqueness, or eventual release.

F-cec6695e396c is not proven to copy, move, exchange, or destroy storage. F-4bd8a1faf697 is not proven to release temporary objects or preserve descriptors.

Failure is not transactional. Setup and growth may already have changed descriptors, storage, or shared state before status 0 is returned. No explicit rollback occurs.

Caller-visible positive-path results require the relevant downstream contracts: transfer must propagate any required effective-destination state, and subsequent dependencies must preserve whichever state the caller expects.

## Dependencies

| Opaque ID | Established role | Unresolved behavior |
|---|---|---|
| F-ea92bf179106 | Positive-path context entry | State changes, preservation, failure behavior |
| F-834e12b6dfde | Temporary destination and scratch acquisition | Initialization, uniqueness, ownership, lifetime, nullability |
| F-08bc2fc94856 | Conditional capacity handling | Allocation policy, descriptor mutation, return interpretation, high carrier half |
| F-9d4416a58736 | Count-4 processing | Numerical output, storage and overlap requirements |
| F-888bcfb33b92 | Count-8 processing | Numerical output, storage and overlap requirements |
| F-f7e02acefeaa | Large-input selector | Result range, mathematical meaning, state changes |
| F-8dc449ebe358 | Processing with explicit scratch data address | Numerical output and workspace requirements |
| F-78a03e109d74 | General processing | Numerical output and implicit state use |
| F-cec6695e396c | Post-finalization destination transfer call | Propagation, ownership, preservation, failure signaling |
| F-4bd8a1faf697 | Common positive-path exit call | Cleanup scope, resource release, descriptor and global effects |

Normal return from a dependency does not itself establish successful growth, descriptor preservation, or ownership behavior.

## Acceptance examples

These are conditional static acceptance cases, not runtime observations. Unless stated otherwise, accesses and arithmetic are valid and invoked dependencies return normally.

1. **Zero count.** A readable source count of zero causes destination count zero and status 1. No dependency is called; other destination fields are not explicitly changed.

2. **Negative count.** A source count of −3 takes the same path. The function does not reject it with status 0.

3. **Count 4 without destination growth.** Suppose scratch acquisition succeeds and destination capacity is at least 9 **when inspected after context entry and scratch acquisition**. F-9d4416a58736 receives the data addresses read at dispatch. Immediately after local finalization, effective-destination count is 7 if word 7 is zero when inspected, otherwise 8. Return-time caller state additionally depends on any transfer and the exit dependency.

4. **Count 8 with successful capacity handling.** Suppose scratch acquisition succeeds and, at the capacity check, either capacity is at least 17 or required growth returns a nonzero low carrier half while leaving usable storage. F-888bcfb33b92 is selected. Immediately after local finalization, effective-destination count is 15 or 16 according to word 15 at inspection time. No unconditional caller-visible count is asserted.

5. **Small general count.** For retained count 5, successful scratch acquisition and destination-capacity handling permit F-78a03e109d74 with count 5. No scratch growth occurs. The local finalized count is 9 or 10 according to word 9 when inspected. Later dependency effects remain separate.

6. **Identical descriptors.** A positive entry count with identical descriptor addresses causes a temporary-destination request before scratch acquisition. If the selected effective destination is valid, remains usable through processing, and differs from the caller destination, F-cec6695e396c is called after local finalization. Its propagation of the locally finalized count is unknown.

7. **Scratch acquisition failure.** A null scratch-request result causes F-4bd8a1faf697 and status 0 before destination-capacity inspection. Earlier dependency effects are not rolled back.

8. **Destination growth failure.** If capacity is insufficient at its actual inspection and growth returns a zero low carrier half, processing and local finalization are skipped. The exit dependency runs before status 0.

9. **Large matching candidate.** If a valid derived candidate equals retained count \(n \geq 16\), scratch capacity is inspected after the selector call. Adequate capacity at that time, or successful growth leaving the original scratch descriptor usable, permits F-8dc449ebe358 with its current scratch data address.

10. **Multiple zero high words.** If the top two provisional words are zero when relevant, local finalization still removes only one word from the effective-destination count. It does not establish that the caller later observes that count unchanged.

## Uncertainties

- **documented_only — Numerical semantics:** Squaring is inferred intent; dependency implementations are absent.
- **documented_only — Arithmetic limits:** Counts and size expressions use 32-bit operands. No upper-bound check exists. Ordinary signed arithmetic requires \(2n+1 \leq 2^{31}-1\); the matching-candidate branch additionally requires a meaningful \(4n\) request. Address displacement validity is separate.
- **documented_only — Overflow behavior:** The representation does not establish a portable policy for every overflowing multiplication, addition, or address calculation.
- **documented_only — Shift behavior:** Retaining eight shift-count bits does not establish validity for a 32-bit shift.
- **documented_only — Growth carrier:** The high return half has observable downstream use but unknown meaning.
- **documented_only — Null temporary destination:** No local check guarantees safe handling before dereference.
- **documented_only — Preservation:** Setup, selection, growth, processing, transfer, and exit dependencies may affect descriptors or shared state.
- **documented_only — Return-time state:** Locally finalized metadata is not an unconditional caller-visible postcondition.
- **documented_only — Ownership and overlap:** Temporary release, buffer ownership, and general overlap safety remain unresolved.

## Evidence

The supplied body statically establishes the entry-count read, descriptor offsets, dependency order, dispatch branches, capacity checks, carrier use, explicit failure exits, local metadata writes, one-word inspection, optional transfer, and status selection.

The body also places capacity observations after unknown setup calls and places unknown transfer and exit calls after local metadata finalization. Those boundaries limit both entry-state assumptions and return-time postconditions.

Dependency implementations and external contracts were not supplied. Semantic intent, ownership guarantees, preservation guarantees, and numerical results remain unestablished where identified above.

No runtime tests have been performed.
