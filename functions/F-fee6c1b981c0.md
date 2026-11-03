<!-- Trusted provenance: F-fee6c1b981c0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fee6c1b981c0

## Purpose

F-fee6c1b981c0 searches a bounded sequence of numeric candidate positions by repeatedly selecting a midpoint and consulting an indirect comparison dependency.

A zero comparison result returns the selected position immediately. A negative result retains the logically preceding candidates. A positive result retains the logically following candidates. Exhausting the candidates returns zero.

**Established static behavior:** the routine calculates candidate positions and branches on signed comparison results. It does not itself inspect candidate contents.

**Inferred intent:** the inputs describe an ordered collection, with an opaque search value compared against selected elements. Correct ordered-search results require a suitable ordering and comparison direction; the routine does not verify either.

Zero is both the unsuccessful-search result and a possible successful position. The return value alone cannot distinguish those outcomes.

## Interface

### Inputs

The routine accepts five values in this order:

| Input | Representation | Role |
|---|---|---|
| Opaque search value | 32-bit value | Passed unchanged as the first argument to each comparison invocation. |
| Starting position | Signed 32-bit integer | Initial numeric position of the first candidate. |
| Candidate count | Unsigned 32-bit integer | Initial number of eligible candidates, from zero through 4,294,967,295. |
| Position spacing | Signed 32-bit integer | Determines midpoint displacement and advancement after a positive comparison. |
| Encoded dependency target | Unsigned 32-bit integer | Identifies an indirect call target after clearing its lowest two bits. |

“Position” denotes a numeric value, not an established pointer or object type. The evidence does not establish element layout, allocation size, alignment requirements, or whether the opaque search value refers to memory.

The effective dependency target is the encoded target masked with `0xFFFFFFFC`. Encoded values differing only in their lowest two bits therefore select the same target. The meaning of those bits is unknown.

### Output

The routine returns a signed 32-bit value:

- The selected candidate position if a comparison returns zero.
- Zero if the search exhausts its candidates.

A successful result need not be positive. Its value is the signed candidate representation used for that comparison.

There is no separate status, match index, output buffer, or error result.

### Comparison interface

**F-indirect-01** names the unresolved indirect dependency within this contract. It does not assert a recovered concrete identity.

Each invocation receives:

1. The original opaque 32-bit search value.
2. A signed 32-bit candidate position.

Its result is interpreted as a signed 32-bit integer. Zero means immediate success. Negative and positive results select opposite search portions when more than one candidate remains. Their magnitudes are otherwise irrelevant.

### Preconditions and validity limits

Normal completion requires every invoked target to support the observed interface and every dependency invocation to return normally. Any dependency memory access must satisfy that dependency’s requirements.

Arithmetic validity is stage-specific:

- Midpoint multiplication and addition have established unsigned 32-bit modular behavior.
- Conversion of an out-of-range unsigned midpoint value to the signed candidate representation is implementation-defined.
- Advancing the starting position uses signed addition; overflow has no defined portable result.

No validation of targets, memory, comparison consistency, or arithmetic range is visible. Zero count, zero starting position, zero spacing, and negative spacing are not explicitly rejected.

## Behavior

### Empty range

The current unsigned count is checked before any comparison invocation.

If the count is zero, the routine returns zero without selecting or examining a candidate. No indirect call is attempted. An empty search therefore does not require a callable dependency target for this routine’s own execution path.

### Midpoint selection

For a count greater than one, the routine takes the unsigned count divided by two, rounded down, as the midpoint offset.

It calculates a candidate position from the current starting position, midpoint offset, and original spacing, using the arithmetic stages specified below. It then invokes F-indirect-01 with the opaque search value and that candidate position.

In ordinary, nonwrapping collection arithmetic, this selects the upper of the two central candidates when the count is even. For two candidates, the second is examined first.

### Midpoint result and range transition

A zero comparison result returns the calculated candidate position immediately. No range advancement occurs afterward.

A negative result retains the logically preceding portion:

- The starting position is unchanged.
- The new count is the midpoint offset.
- The selected midpoint and all logically later candidates are excluded.

A positive result retains the logically following portion:

- The new starting position is calculated by adding the spacing to the signed midpoint position.
- The new count is the previous count minus the midpoint offset minus one.
- The selected midpoint and all logically earlier candidates are excluded.

The signed starting-position addition occurs before the next empty-range check. It is performed even when the resulting candidate count will be zero.

These transitions alter local search state. They do not move or modify candidate storage.

### Single remaining candidate

When the count is exactly one, F-indirect-01 receives the current starting position directly. No midpoint displacement is calculated.

- A zero comparison result returns that starting position.
- Any nonzero result returns zero.

At this stage, negative and positive results have identical consequences. Neither causes an additional spacing increment.

### Position arithmetic

The midpoint calculation and the subsequent positive-branch advancement have different arithmetic rules.

**Midpoint multiplication:** the midpoint offset is unsigned 32-bit. The signed spacing is therefore converted to unsigned 32-bit for multiplication. The product is reduced modulo 4,294,967,296.

**Midpoint addition:** the signed starting position is converted to unsigned 32-bit and added to that unsigned product. This addition is also reduced modulo 4,294,967,296.

Thus, negative starting positions or spacings participate through their unsigned conversions. Exceeding the signed range during these unsigned stages is not signed overflow.

**Candidate conversion:** the resulting unsigned 32-bit value is converted to the signed 32-bit candidate representation.

- Values from zero through 2,147,483,647 retain their value.
- For values from 2,147,483,648 through 4,294,967,295, the conversion behavior is implementation-defined. This contract does not assume a particular negative result or exact target-machine behavior.

**Positive-branch advancement:** after a positive comparison, the routine adds the signed candidate position and signed spacing. This is signed 32-bit addition, not the preceding unsigned modular calculation. A mathematical result outside −2,147,483,648 through 2,147,483,647 overflows, with no defined portable result.

There is no visible checked-arithmetic error, saturation, wider intermediate guarantee, or overflow recovery. In particular, the modular midpoint rule does not establish modular behavior for positive-branch advancement.

Zero spacing can make distinct logical candidates produce identical numeric positions. Negative spacing can make later logical candidates have lower numeric positions. Neither receives special handling.

### Count arithmetic and termination

The count remains unsigned. A high bit does not make it negative.

For a count greater than one, the midpoint offset is smaller than the count. Both possible retained portions have fewer candidates than before. The following-portion count calculation does not underflow.

Assuming normally returning comparisons and arithmetic behavior that permits the specified search transitions:

- Zero initial count produces no comparison calls.
- Any nonzero 32-bit count requires at most 32 comparison calls.
- Equality can end the search earlier.

This is a bound on calls issued by this routine, not elapsed time or work performed inside the dependency.

### Ordered-search interpretation

For conventional ordered-search correctness, comparison results must justify excluding the relevant portion:

- A negative result must rule out the selected midpoint and all logically later candidates.
- A positive result must rule out the selected midpoint and all logically earlier candidates.
- A zero result must identify an acceptable match.

These are caller/dependency obligations for the inferred collection interpretation, not properties checked by the routine. Inconsistent comparisons, changing contents, unsuitable ordering, or position arithmetic incompatible with the collection can cause a match to be missed.

When several candidates would compare equal, the routine returns the first equal midpoint encountered. It does not continue toward an earliest or latest equal candidate.

## Side effects

### Direct effects

The routine updates its current starting position and count. It does not visibly reassign the opaque search value, spacing, or encoded dependency target.

No direct reads of candidate contents, writes to caller-owned candidate storage, allocation, deallocation, ownership transfer, persistent global update, or resource acquisition are visible.

The supplied interface establishes numeric argument widths, not a memory layout for candidate objects.

### Observable ordering

Each comparison follows calculation of its candidate argument. Its result is obtained before the routine chooses the next search state or returns.

Only the selected branch is followed; the routine does not compare both retained portions. A zero result ends further comparison calls immediately.

For a positive midpoint result, signed starting-position advancement precedes calculation of the following-portion count and the next iteration’s count check. Consequently, an empty following portion does not eliminate the signed-addition overflow risk.

### Dependency-mediated effects and ownership

F-indirect-01 may read or modify memory, change globals, allocate or release resources, or fail to return. Its complete effects are not established.

Passing the two numeric arguments establishes neither ownership transfer nor preservation of referenced objects. In particular:

- The dependency is not assumed to preserve globals.
- It is not assumed to preserve candidate contents.
- It is not assumed to preserve or release memory associated with the opaque search value.
- Later comparisons may observe changes made by earlier comparisons.
- Reusing the same opaque argument value does not guarantee that any referenced memory remains valid.

No cleanup or recovery path for abnormal dependency termination is visible.

### Errors

There is no explicit error channel.

Zero represents unsuccessful search, not a general failure conversion. Invalid targets, dependency access failures, implementation-specific conversion behavior, and signed-overflow consequences are not visibly caught and translated into zero.

## Dependencies

| Opaque ID | Selection | Established interface | Unresolved properties |
|---|---|---|---|
| F-indirect-01 | Encoded target with its lowest two bits cleared | Accepts the opaque 32-bit value and signed 32-bit candidate; supplies a signed 32-bit comparison result | Concrete identity, target validity, memory effects, ownership, global-state effects, failure behavior, and additional calling requirements |

The identifier is a contract-local name for the unresolved target. Different encoded inputs may select different concrete dependencies. No other explicit function dependency is visible.

## Acceptance examples

These are conditional expectations derived from static evidence, not executed tests. Unless an example specifically addresses arithmetic limits, it assumes normally returning comparisons and applicable arithmetic behavior.

### Empty range

For count zero, the result is zero and F-indirect-01 is not invoked. Starting position, spacing, and target validity do not affect this routine’s empty-range result.

### Single candidate

For starting position 100 and count one:

- Exactly one comparison receives candidate 100.
- Equality returns 100.
- Either sign of nonzero result returns zero.

Spacing is not used in selecting or advancing this candidate.

### Preceding and following portions

For starting position 100, count three, and spacing four, the first candidate is 104.

- If its result is negative, the next candidate is 100.
- If its result is positive, the next candidate is 108.
- Equality at the first candidate returns 104 immediately.
- At either remaining single candidate, equality returns that position and any nonzero result returns zero.

The unselected portion is not examined.

### Two-candidate boundary

For starting position 100, count two, and spacing four, candidate 104 is examined first.

- Equality returns 104.
- A negative result causes candidate 100 to be examined.
- A positive result advances the starting position to 108, leaves count zero, and then returns zero without another comparison.

This case establishes both upper-midpoint selection and advancement before empty-range termination.

### Comparison magnitudes

At a midpoint with more than one candidate, −7 selects the preceding portion and +23 selects the following portion. Replacing either value with another value of the same sign does not change that branch.

### Zero and negative spacing

For starting position 100, count three, and spacing zero, a negative result followed by equality produces two comparisons at numeric position 100 and returns 100.

For starting position 108, count three, and spacing −4, the first candidate is 104. A positive result advances to 100; equality there returns 100. Logical “following” therefore does not necessarily mean numerically greater.

### Defined unsigned midpoint wrap

For starting position two, count four, and spacing 2,147,483,647:

- The midpoint offset is two.
- The unsigned product is 4,294,967,294.
- Adding the unsigned starting position produces zero modulo 4,294,967,296.
- Zero converts exactly to the signed candidate representation.

The first comparison therefore receives candidate zero. If it reports equality, the routine returns zero as a successful position. This calculation does not incur signed multiplication or signed midpoint-addition overflow.

### Out-of-range candidate conversion

For starting position zero, count two, and spacing −1, the unsigned midpoint value is 4,294,967,295.

Its conversion to a signed candidate is implementation-defined. This contract does not prescribe candidate −1. Any acceptance expectation about the exact comparison argument requires the execution environment’s conversion behavior.

### Signed advancement overflow

For starting position 2,147,483,646, count two, and spacing one, the first candidate is exactly 2,147,483,647.

- Equality returns that candidate without advancement.
- A positive result attempts signed addition of one to it.

That addition overflows. Although the following-portion count would be zero, a portable expectation of returning zero is not justified: advancement occurs first.

### Target normalization and zero-result ambiguity

Encoded targets `0x1000`, `0x1001`, `0x1002`, and `0x1003` all select target `0x1000`. This does not establish that the target is executable or compatible.

Separately, starting position zero with count one and an equality result returns zero. That output is indistinguishable from unsuccessful search without additional observation.

## Uncertainties

The following remain **documented_only**:

- The meaning and memory interpretation of the opaque search value.
- Whether positions are addresses, offsets, handles, or another representation.
- Candidate object layout, alignment, storage extent, and lifetime.
- The concrete identity and full contract of F-indirect-01.
- The purpose of the encoded target’s lowest two bits.
- The environment’s behavior for unsigned-to-signed candidate conversions outside signed range.
- Exact execution consequences of signed starting-position overflow.
- Caller guarantees about ordering, comparison stability, target validity, or permissible positions.
- Dependency ownership, memory preservation, concurrency, reentrancy, and global-state behavior.
- Whether callers intentionally permit successful position zero.

Unsigned midpoint modular arithmetic is established, not an unresolved overflow policy. The unresolved conversion and signed-advancement cases are separate stages.

The ordered-collection interpretation remains inferred intent.

## Evidence

The supplied static representation establishes:

- Five inputs with the widths and signedness described above.
- An empty-count exit before any indirect invocation.
- Unsigned halving for midpoint selection.
- Unsigned 32-bit midpoint multiplication and addition, followed by conversion to a signed candidate.
- Clearing of the target’s lowest two bits before invocation.
- Immediate return of the selected candidate on equality.
- Preceding-range selection for negative midpoint results.
- Signed starting-position addition and following-range count reduction for positive midpoint results.
- A distinct single-candidate comparison with no spacing-based advancement.
- No visible candidate-content access, caller-storage mutation, allocation, release, or explicit error recovery.

The arithmetic distinctions follow from the operand types and operation order. The call-count bound and acceptance examples are deductions from that static behavior.

No runtime tests have been performed.
