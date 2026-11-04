<!-- Trusted provenance: F-3a43adaaeff9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3a43adaaeff9

## Purpose

F-3a43adaaeff9 copies a two-word state into a destination, computes an adjustment using a dependency result and a global baseline, and applies a sign-dependent correction to both destination components.

The arithmetic resembles normalization of a two-component numeric representation. That is **inferred intent**, not an established guarantee: the supplied fragment does not establish units, a normalization interval, a divisor, or the values of the global arithmetic operands.

The established behavior is the ordered sequence of reference evaluations, memory accesses, dependency calls, arithmetic, and destination writes described below. In particular, the function evaluates a global data-region reference twice before its first dependency call. The first evaluation supplies a saved address for a later baseline read; the second supplies the address for an immediate byte read. These addresses are not established to be equal.

## Interface

The function has five explicit inputs and no explicit return value.

| Input | Established representation | Use |
|---|---|---|
| First input | One 32-bit word | Not referenced by the visible body. |
| Second input | One 32-bit word | Passed unchanged to F-35e19007c08f. Its semantic meaning and signedness are unknown. |
| Source state | Address of two consecutive 32-bit words | Supplies the initial destination components. |
| Destination state | Address of two consecutive 32-bit words | Receives the initial copy and both corrected components. |
| Adjustment factor | Signed 32-bit integer | Multiplies the difference between the first dependency result and the baseline. |

Both state addresses use this layout:

| Byte offset | Width | Established role |
|---|---|---|
| 0 | 32 bits | First component, initially copied and later increased by a correction. |
| 4 | 32 bits | Second component, initially copied and incorporated into an adjusted signed sum. |

These are word accesses, not an explicitly serialized format. Byte order and alignment requirements are not established.

Implicit inputs include:

- A global data-region reference, evaluated twice before the first dependency call.
- One byte at offset 0 of the address obtained by the second evaluation.
- A signed 32-bit baseline at offset 4 of the address saved by the first evaluation.
- Two branch-specific global operands supplied to F-870c4fcf7295.
- A global multiplier used in the final second-component calculation.
- Four additional incoming 32-bit machine-state values displayed as arguments to F-870c4fcf7295.

The declarations of the branch-specific global operands and final multiplier are absent. Their storage widths and signedness therefore remain unresolved, although their uses occur in the displayed call and integer-arithmetic expressions. The pointer width is also not established.

### Preconditions and ownership

Ordinary completion requires valid memory for each access at the time it occurs:

- The source must support the two 32-bit reads.
- The destination must support the specified 32-bit reads and writes.
- The address obtained by the second global-reference evaluation must support the byte read at offset 0.
- The address saved by the first evaluation must still support the baseline read at offset 4 after F-35e19007c08f returns.

The two global-reference evaluations may yield different addresses; validity of one does not establish validity of the other.

The visible body performs no allocation, release, or ownership transfer. Neither dependency is established to preserve memory ownership, storage contents, global values, or the lifetime of referenced regions.

There are no explicit null checks, bounds checks, error returns, or local recovery paths.

## Behavior

### 1. Save one global-reference value, then evaluate the reference again

The function first reads the global data-region reference and saves the resulting address.

It then evaluates that global reference separately and reads one byte at offset 0 of the address obtained by this second evaluation. It calls F-35e19007c08f with:

1. The second explicit input, unchanged.
2. The value from that one-byte read.

The dependency result is subsequently consumed as a signed 32-bit integer. The precise extension or argument-passing treatment of the byte is not established.

The distinction between the two reference evaluations is part of the contract:

- The **saved address** comes from the first evaluation and supplies the later baseline location.
- The **byte-read address** comes from the second evaluation and supplies the immediate byte location.

No visible dependency call occurs between these evaluations, but the saved-address data flow does not establish their equality. Equality requires the reference value to remain stable across the evaluations. The fragment provides no synchronization guarantee or other rule establishing that stability.

If F-35e19007c08f subsequently changes the global reference, the later baseline access still uses the saved address. However, the dependency may modify or invalidate the region at that saved address; preservation is not established.

### 2. Retain the source second component and calculate the adjustment

After F-35e19007c08f returns, the function reads and retains the source word at offset 4.

It then reads the signed 32-bit baseline at offset 4 of the saved address. Thus, the baseline is obtained after the first dependency call, not when either global-reference evaluation occurs.

For representable arithmetic, define:

- \(R\): the first dependency result.
- \(B\): the baseline read through the saved address.
- \(K\): the signed adjustment factor.
- \(S\): the retained source second component.
- \(\Delta = K(R-B)\): the calculated adjustment.

The difference is formed before multiplication by the adjustment factor. Both operations appear as signed 32-bit arithmetic. The fragment does not establish overflow handling, so these mathematical equations specify exact values only when the relevant intermediate results are representable.

A dependency mutation of the baseline at the saved address affects this calculation. A mutation of the global reference alone does not redirect it. If the byte-read address differs from the saved address, a baseline-like value in the byte-read region is not used merely because the byte came from that region.

### 3. Copy the state using ordered word accesses

After calculating the adjustment, the function:

1. Reads source offset 0.
2. Writes that value to destination offset 0.
3. Writes the retained source second component to destination offset 4.
4. Reads destination offset 4.

Both source components have therefore been read before the first destination write. The second source component was retained before the adjustment calculation; it is not reread for the copy.

This sequence is not an atomic copy and does not establish general block-copy semantics. Exact source/destination equality is compatible with the visible accesses. Other overlap is governed by the same access order, memory validity, and any dependency effects.

### 4. Form the adjusted sum and choose a correction

The function adds \(\Delta\) to the value read back from destination offset 4. Call the resulting signed 32-bit value \(T\).

Without intervening changes to that destination word, \(T=S+\Delta\). The actual operation uses a destination read, however, rather than directly reusing \(S\).

The branch condition is strictly \(T<0\). Zero selects the nonnegative branch.

On the negative branch, F-870c4fcf7295 receives:

1. \(T\).
2. The negative-branch global operand.
3. \(\Delta\).
4. The retained source second component \(S\).
5. Four additional incoming machine-state values.

If its result is \(H\), the correction is:

\[
C=-(H+1)
\]

The addition of one precedes negation. This is not equivalent to simply negating the result.

On the nonnegative branch, the dependency receives the same argument roles, but with the nonnegative-branch global operand. Its result is used directly:

\[
C=H
\]

Exactly one correction-dependency call is reached on an ordinary execution that reaches this stage. Its branch-specific global operand is obtained after the two initial destination writes.

No quotient, divisor, rounding rule, or relationship between the two global operands is established. The negative-result transformation alone does not establish floor division or any particular normalization rule.

### 5. Update the first component, then replace the second

After F-870c4fcf7295 returns and the correction is determined, the function reads destination offset 0, adds \(C\), and writes the result back to that location.

It then calculates and writes the final second component.

Let \(A\) be the destination first component read after the correction dependency returns, and let \(M\) be the global multiplier used by the final expression. For representable arithmetic:

\[
\text{final first component}=A+C
\]

\[
\text{final second component}=T+(MC)\times128
\]

The displayed grouping multiplies \(M\) by \(C\), then multiplies that result by 128, then adds \(T\). Intermediate widths and conversions may affect overflowing cases; unlimited-precision arithmetic is not an established substitute.

The first-component update uses the destination’s current value, not necessarily the originally copied first component. A dependency mutation of destination offset 0 can therefore affect the result.

The final second-component calculation instead uses retained \(T\). It does not reread destination offset 4 after the correction dependency. A dependency mutation of that destination word is overwritten on normal completion, unless it also changes another relevant input through aliasing.

The final multiplier is used after the corrected first-component write. It is not an established entry-time snapshot. If its storage aliases destination storage, earlier writes can affect the value used.

## Side effects

The visible body performs four destination writes in this order:

1. Copy the first component.
2. Copy the second component.
3. Write the corrected first component.
4. Write the final second component.

These writes are not atomic as a group. F-870c4fcf7295 is called between the initial copy and the final updates, so its execution occurs while the destination contains the copied state, subject to other mutations.

The body also evaluates global references, reads global data, and calls two dependencies. Their side effects are not bounded by the supplied evidence.

There is no explicit direct global write in the body. Destination/global aliasing can nevertheless make a destination write change global storage, including storage subsequently read by the function.

The function provides no status return or rollback. Failure before a later write does not undo an earlier write. If a dependency fails to return normally, subsequent calculations and writes are not guaranteed.

No synchronization, thread-safety, or concurrent-observer guarantees are established. The listed access order describes the visible operation, not a separate cross-thread memory-ordering contract.

## Dependencies

### F-35e19007c08f

**Established inputs:** the second explicit input and a byte obtained through the second evaluation of the global data-region reference.

**Result use:** consumed as a signed 32-bit integer in the baseline difference.

**Position:** after both global-reference evaluations and the byte read; before either source-component read or any destination write.

**Unknowns:** semantic operation, accepted ranges, byte-extension convention, error behavior, memory effects, global preservation, and ownership or lifetime effects.

The baseline address is saved before this call. The baseline contents are read after it.

### F-870c4fcf7295

**Established displayed inputs:** the adjusted sum, the sign-selected global operand, the adjustment, the retained source second component, and four additional incoming 32-bit machine-state values.

**Position:** after the initial destination copy and before both final updates.

**Result use:** direct on the nonnegative branch; addition of one followed by negation on the negative branch.

**Unknowns:** actual calling convention, arithmetic meaning, accepted ranges, error behavior, side effects, and significance of the four additional values.

Those additional values must not be assumed irrelevant. Conversely, the fragment does not establish that they are intentional high-level parameters rather than consequences of an incompletely established call interface.

## Acceptance examples

These are conditional static acceptance examples, not runtime observations. Unless stated otherwise, relevant storage remains unchanged except as described, all accesses are valid, and arithmetic is representable.

### Distinct addresses from the two reference evaluations

Suppose the first global-reference evaluation yields a region whose baseline at offset 4 is 5. The second evaluation yields another region whose byte at offset 0 is 9 and whose word at offset 4 is 100.

F-35e19007c08f must receive the byte value 9. If it returns 8 and the adjustment factor is 2, the adjustment is:

\[
\Delta=2(8-5)=6
\]

The baseline is 5 from the first region, not 100 from the byte-read region. This example describes behavior conditional on distinct evaluation results; it does not assert a mechanism that makes the reference change.

### Stable reference

If the global reference yields the same address on both evaluations, the byte and baseline come from offsets 0 and 4 of the same region. They are still read at different times, with F-35e19007c08f between the byte read and baseline read.

### Nonnegative adjusted sum

Suppose the source is \((12,7)\), the first dependency returns 9, the baseline is 5, and the adjustment factor is 3.

Then \(\Delta=12\) and \(T=19\). The nonnegative operand is supplied to F-870c4fcf7295. If it returns 2 and the final multiplier is \(-1\), the destination passes through \((12,7)\) and finishes as:

\[
(14,-237)
\]

### Negative adjusted sum

Suppose the source is \((12,7)\), the first dependency returns 3, the baseline is 5, and the adjustment factor is 5.

Then \(\Delta=-10\) and \(T=-3\). The negative operand is selected. If the correction dependency returns 0, then \(C=-(0+1)=-1\). With a final multiplier of \(-1\), the final state is:

\[
(11,125)
\]

This does not assert how the dependency obtains its result.

### Zero boundary

If \(T=0\), the nonnegative operand is selected. If the correction dependency returns 0, the first-component update adds zero and the final second component is zero.

An adjustment factor of zero does not bypass either dependency or the baseline read.

### Baseline mutation and reference replacement

Suppose F-35e19007c08f changes the baseline in the saved region from 10 to 14 and also replaces the global reference with an address of another region.

The later baseline read uses 14 from the saved region, provided that region remains valid. Replacing the global reference does not redirect the read.

### Destination mutation by the correction dependency

Suppose the nonnegative correction dependency changes destination offset 0 to 100 and returns 3. The final first component is 103.

If the dependency also changes destination offset 4, that change does not replace retained \(T\). The final second-component write uses \(T\) and the final multiplier.

### Identical source and destination

With identical valid source and destination addresses and no interfering effects, both source words are captured before the initial writes. The copy preserves those values, and the subsequent updates proceed from them. The visible body does not require a separate source buffer.

## Uncertainties

The following remain **documented_only**:

- Whether the global reference is stable across its two initial evaluations.
- The mechanism, if any, by which those evaluations could yield different addresses.
- Global operand values, undeclared storage widths, and relevant conversion rules.
- Pointer width, byte order, alignment requirements, and fault behavior.
- Intended units and permissible ranges of the state components.
- The correction dependency’s mathematical operation and actual call interface.
- Overflow behavior for subtraction, multiplication, addition, and negative-branch correction.
- Behavior when adding one to the negative-branch helper result, or negating that intermediate result, is not representable.
- Dependency effects on globals, source, destination, ownership, and memory lifetime.
- External caller preconditions concerning aliasing, overflow, or valid memory.
- Concurrency and synchronization semantics.

These unknowns prevent a stronger promise of canonical normalization, modulo behavior, a particular quotient, or safety for arbitrary inputs.

## Evidence

The supplied static fragment establishes two separate initial evaluations of the global data-region reference: the first is saved for the later baseline read, while the second supplies the byte-read address for F-35e19007c08f. Equality of those addresses is not established by the saved-reference data flow.

The fragment also establishes the five explicit inputs, two-word state layout, delayed baseline read, ordered source and destination accesses, dependency-call positions, strict negative branch condition, negative-result transformation, and final arithmetic structure.

Normalization is inferred intent only. No dependency bodies, global initializers, caller contracts, or target arithmetic specification were supplied.

No runtime tests have been performed.
