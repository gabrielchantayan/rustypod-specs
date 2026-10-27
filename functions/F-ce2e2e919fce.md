<!-- Trusted provenance: F-ce2e2e919fce; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ce2e2e919fce

## Purpose

F-ce2e2e919fce changes a caller-held position by following relationships between records. Its only explicit output is the updated position slot.

Two traversal behaviors are established:

- A nonzero link at byte offset 12 selects a descent through links at byte offset 8.
- A zero link at byte offset 12 selects an upward traversal through links at byte offset 4, governed by relationship comparisons involving byte offset 12.

The position is updated during traversal, not committed only at the end. Those intermediate writes are part of the behavior.

**Inferred intent:** The relationships resemble an in-order successor operation, with parent, left-child, and right-child links. That interpretation is not an established structural precondition. The supplied evidence does not establish a valid tree, an end-position representation, or a sentinel convention.

This contract therefore calls the fields **upward**, **inner**, and **outer** links. These are descriptive labels, not guarantees about the surrounding data structure.

## Interface

### Argument and result

The sole argument identifies a readable and writable position slot. The slot contains an integer-valued record address.

There is no explicit return value, success indicator, exhaustion indicator, or error result. A caller obtains the result by inspecting the position slot after normal completion.

The current record address is used without a preliminary zero check. A zero initial position is not defined as an empty iterator or a harmless no-op.

### Accessed representation

| Storage | Contract name | Access |
|---|---|---|
| Caller-provided slot | Position | Read and written |
| Record address plus 4 bytes | Upward link | Read |
| Record address plus 8 bytes | Inner link | Read |
| Record address plus 12 bytes | Outer link | Read |

The slot, link loads, stored positions, and retained traversal candidate use the same integer type in the supplied representation.

The field offsets are established byte displacements. They do **not**, by themselves, establish the width of that integer type, the width of a machine address, or alignment requirements. A 32-bit address-bearing representation is consistent with the layout, but remains an inference without independent target information.

Consequently:

- The offsets must remain 4, 8, and 12 bytes.
- Replacing these accesses with native-pointer-width fields on another target is not justified by this evidence alone.
- No explicit masking, narrowing, widening, or address normalization is present.
- Neither the complete record size nor the contents of offset 0 are established.
- Four-byte field spacing alone does not establish that the accessed integer-sized regions are nonoverlapping on every possible target.

Zero is used as the termination value for the initial outer-link decision and the inner-link descent. It is not used as an upward-traversal termination guard.

### Conditions for normal completion

The argument must identify storage supporting every position read and write that execution reaches. Each accessed link location must likewise support its required integer load.

Normal completion additionally depends on the selected traversal reaching its stopping condition. No cycle detection or traversal bound is present.

For ordinary stable-structure interpretations, the following conditions are necessary:

- Link storage remains valid throughout traversal.
- Address calculations produce usable field addresses.
- Memory does not change independently during the operation.
- Position-slot writes do not modify links subsequently consulted.

The last condition is not an established prohibition on aliasing. If the position slot overlaps record storage, its writes can change the traversal itself. Such cases must be interpreted using the actual access dependencies below rather than the stable-structure shorthand.

## Behavior

### Path selection

The first relationship inspected is the outer link of the record identified by the position slot.

A nonzero outer link selects descent. A zero outer link selects upward traversal. There is no separate validation of the current record, its ownership, or the consistency of its relationships.

### Descent through inner links

On this path, the original outer-link value is retained as the first candidate. That candidate is written to the position slot **before** its inner link is read.

Every further nonzero inner link supplies another candidate and causes another position write. A zero inner link ends the operation without being written into the position slot.

For valid, stable, non-aliasing storage with a finite chain, the final position is therefore the last record on the inner-link chain beginning at the original record’s outer link.

The write-before-read ordering is significant:

- Each candidate becomes visible in the slot before its inner-link access.
- An unsuccessful inner-link access can occur after the position has already changed.
- Aliasing can cause the position write to alter the very inner link about to be read.

The next inner-link address is based on the retained candidate. It does not require reloading that candidate from the position slot.

This path does not inspect upward links. It also bypasses the final relationship test described for upward traversal.

### Upward traversal

When the initial outer link is zero, the upward link of the current position supplies a candidate.

Progress depends on a directional relationship: the candidate’s outer link must equal the current position. A matching relationship causes the candidate to replace the position. The next candidate then comes from that candidate’s upward link.

The position write precedes the upward-link read used to advance. If those locations overlap, the write can change the next candidate.

The relationship is checked again for each new candidate. Traversal stops at the first mismatch between the candidate’s outer link and the current position.

A zero candidate is not a stopping condition. Its outer-link location is still needed for the relationship comparison. In particular, a zero upward link can lead to an attempted load at address 12 rather than a safe return or a zero result.

For stable, non-aliasing storage, this behavior can be described as climbing while each candidate names the current position through its outer link. That description does not imply that any tree invariant is checked.

### Final relationship decision

After the upward traversal finds a mismatch, the last candidate is considered for one final position update.

This decision uses the relationship in the **opposite direction**: the outer link of the current position is compared with the retained candidate.

- If they differ, the candidate is written to the position slot.
- If they are equal, there is no final write.

An unconditional assignment of the last candidate would therefore be incorrect.

“No final write” does not mean “no change during the call.” The upward traversal may already have updated the position one or more times.

This final decision uses the position as it exists after any earlier updates. It is not based on a saved copy of the initial position.

### Access ordering and stability limits

The supplied representation establishes the following dependencies:

- Path selection precedes traversal.
- Each descent position write precedes that candidate’s inner-link read.
- Each matching upward relationship precedes its resulting position write.
- That upward position write precedes the next upward-link read.
- The final relationship decision follows the first upward mismatch.

The relative evaluation order of the two sides of an upward relationship comparison is not established as a portable guarantee. In particular, the candidate’s outer-link read and the position-slot read should not be assigned an invented ordering.

Reading an outer link through the position necessarily depends on obtaining the position value used as its address. No snapshot, synchronization, or transactional consistency is provided.

## Side effects

### Writes and visible progress

All explicit stores target the caller’s position slot. There are no explicit stores through a separately addressed record field.

That distinction does not guarantee that record links remain unchanged: an overlapping position slot makes a position store also modify record storage.

For a normally completing descent, the number of position writes equals the number of nonzero candidates visited. For upward traversal, each matching relationship causes one write, followed by either one final write or none.

Stores count as behavior even when they repeat the same value. A replacement implementation must not assume that only the final slot value matters in contexts where intermediate writes are observable.

### Ownership and lifetime

No allocation, deallocation, reference-count adjustment, or explicit ownership transfer appears.

The operation does not establish ownership of either the position slot or the records. Their lifetime requirements belong to the surrounding interface, which is not supplied.

There are no unknown callees whose memory-management behavior must be assumed.

### Failure and nontermination

There is no explicit validation, error reporting, rollback, or recovery path.

Invalid addresses, unsuitable alignment, invalid address calculations, or concurrent changes may prevent normal completion. The evidence does not define the resulting platform behavior. No visible mechanism restores earlier position values after a failed access.

Cycles can cause nontermination. On descent, a stable cycle of nonzero inner links produces repeated position writes indefinitely. Upward cycles can likewise continue when their relationship comparisons keep matching.

Address formation uses fixed offsets applied to integer-valued addresses. The target arithmetic width and overflow rules are not independently established. No wrapping-address behavior should be promised.

No atomic-access or synchronization guarantee is supplied.

## Dependencies

No external function calls are present. There are no dependent function IDs to enumerate.

The operation depends directly on:

- The caller-provided position slot.
- The reached upward, inner, and outer link locations.
- The target’s integer representation, address formation, alignment, and memory-access rules.

No explicit global storage is referenced. This does not exclude the caller from supplying a slot or records located in globally accessible storage.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Unless an example states otherwise, named records have distinct nonzero addresses, storage is valid and stable, the position slot does not overlap record storage, and all required address calculations are valid.

### Direct descent

The initial position is A. A’s outer link identifies B, and B’s inner link is zero.

Required behavior:

- B is written into the position slot before B’s inner link is read.
- The operation completes with B as the position.
- No upward-link access is required.

### Longer descent

The initial position is A. Its outer link identifies B. The inner links lead from B to C, from C to D, and from D to zero.

The position writes occur in the order B, C, D. The final position is D. The terminating zero is not stored.

### Immediate upward selection

The initial position is A. A’s outer link is zero, and its upward link identifies P. P’s outer link does not identify A.

There is no matching upward relationship, so the upward traversal performs no position write. The final comparison finds that A’s zero outer link differs from nonzero P. The position is then written with P.

### Multiple upward updates

The initial position is A. A has a zero outer link and upward link P. P’s outer link identifies A and its upward link identifies Q. Q’s outer link identifies P and its upward link identifies R. R’s outer link does not identify Q.

The matching relationships cause position writes to P and then Q. The final comparison uses Q’s outer link, which identifies P. Because P and R are distinct, the final write selects R.

Required write order: P, Q, R.

### Final assignment suppressed by a cycle

This example uses two distinct records, A and P:

- A’s outer link is zero, and its upward link identifies P.
- P’s outer link identifies A, and its upward link identifies A.

Starting at A, the first upward relationship matches and the position becomes P. The next candidate is A. A’s zero outer link does not identify P, so upward traversal stops.

The final comparison finds that P’s outer link identifies the retained candidate A. No final write occurs. The position remains P rather than becoming A.

This is a relationship example, not an assertion that cyclic records form a valid tree.

### Zero upward link

The initial position is A, whose outer and upward links are both zero.

The operation attempts the candidate outer-link access at address 12. Safe completion, a zero result, or any particular fault is not guaranteed. The outcome depends on target memory behavior and any values successfully read.

### Cyclic inner chain

A’s outer link identifies B. B’s inner link identifies C, and C’s inner link identifies B.

With stable storage, the position repeatedly alternates between B and C. There is no normal completion or cycle-error result.

### Position slot overlapping an inner link

The position slot occupies B’s inner-link storage and initially contains A. A’s outer link identifies B.

The first descent write places B in the position slot, thereby also changing B’s inner link to B. The following inner-link read observes that value under ordinary stable-memory semantics.

The descent then repeatedly selects B. An implementation that reads B’s inner link before updating the slot would not preserve this behavior.

## Uncertainties

**documented_only — Exact widths and ABI.** The accesses use a common integer representation, but its bit width, pointer width, alignment, and address-conversion rules are not independently established. Four-byte spacing is not sufficient proof of a 32-bit ABI.

**documented_only — Arithmetic limits.** The evidence does not establish usable behavior when fixed-offset address calculations overflow or fail to identify valid storage.

**documented_only — Structural intent.** Tree-successor terminology is inferred. No independent evidence establishes parent/child invariants, acyclicity, or membership rules.

**documented_only — Boundary conventions.** The final comparison may support a special boundary relationship, but no sentinel layout or end-position value is established.

**documented_only — Invalid-memory outcomes.** Faults, language-level invalid execution, and platform recovery behavior remain unspecified.

**documented_only — Aliasing and concurrency.** No non-aliasing restriction or synchronization discipline is supplied. Stable-chain results are conditional; they do not override the write-before-read behavior.

## Evidence

The supplied static body establishes the accessed byte offsets, common integer-valued representation, branch conditions, traversal relationships, intermediate position writes, and asymmetric final comparison.

It also establishes the absence of explicit external calls, allocation, release, validation, cycle detection, error reporting, and recovery within this operation.

The structural interpretation and possible boundary purpose are inferences, not independently established interface guarantees. Exact target widths and invalid-memory behavior remain unresolved.

No runtime tests have been performed. The acceptance examples are static consequences under their stated conditions, not observed execution results.
