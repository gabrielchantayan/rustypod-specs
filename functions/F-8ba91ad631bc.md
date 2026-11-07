<!-- Trusted provenance: F-8ba91ad631bc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8ba91ad631bc

## Purpose

F-8ba91ad631bc updates a globally selected state record using a current 32-bit sample and a previously recorded sample. Its behavior depends on whether bit 30 is set in each sample.

When both samples have bit 30 set, the function adjusts their signed difference, classifies that difference against a record-specific threshold, and may update one of two accumulators or invoke dependencies. When bit 30 changes state, it captures an accumulator into a separate field and may invoke a dependency. On every normally returning path, it stores the entry-time current sample as the previous sample.

The apparent intent is to track changes in a cyclic quantity while distinguishing small accumulated changes from larger accepted changes. That interpretation is inferred from the constants and field relationships; the physical meaning, units, and lifecycle of the record are not established.

## Interface

**Explicit inputs:** None.

**Implicit input:** A global reference to a mutable record. The function captures that reference at entry and uses both the captured reference and accesses expressed through the global reference.

**Output:** No return value. Observable results consist of record mutations and calls to dependencies.

The supplied representation uses 32-bit integer fields and one-byte accesses. The relevant record layout is:

| Byte offset | Width | Role in this contract |
|---|---:|---|
| `0x00` | 1 byte | Mode byte; zero and nonzero are distinguished |
| `0x04` | 32 bits | Signed threshold |
| `0x08` | 32 bits | Signed small-change accumulator |
| `0x0C` | 32 bits | Signed accepted-change accumulator |
| `0x10` | 32 bits | Captured accepted-change value |
| `0x18` | 32 bits | Previous sample, initially read as unsigned |
| `0x20` | 32 bits | Current sample, read as unsigned |

Other bytes are not directly accessed by this function. They may still be read or modified by dependencies.

Bit 30 has mask `0x40000000`. No other sample bits are explicitly masked out before subtraction.

The direct-access preconditions are:

- The global reference identifies readable storage for all fields that the selected path reads.
- Storage written by the selected path is writable.
- The 32-bit fields satisfy the target’s access and alignment requirements.
- The captured record remains valid through the final previous-sample store, including across dependency calls.
- Execution is not interrupted by unsynchronized mutation that invalidates the described access sequence.

No allocation, ownership transfer, or release operation is directly visible. Dependency ownership behavior is unknown.

## Behavior

### Entry observations and difference

The function first captures the global record reference. It then reads, in this order:

1. The current sample at offset `0x20`.
2. The previous sample at offset `0x18`.
3. The mode byte at offset `0x00`.

These entry-time sample values determine the bit-30 branch and the difference used in that branch. The saved entry-time mode byte is used only on the branch where the samples disagree on bit 30.

The subtraction is expressed using unsigned 32-bit operands. Its result is then assigned to a signed 32-bit value. Thus the subtraction itself is modulo \(2^{32}\). Interpreting an out-of-range unsigned result as signed depends on the target conversion semantics; the ordinary two’s-complement interpretation is the basis for the signed examples below.

The function does not compute a difference between masked sample payloads. All 32 sample bits participate in subtraction.

### Both bit-30 values clear

If bit 30 is clear in both samples, the function performs no accumulator update, reset, or dependency call.

It proceeds directly to storing the entry-time current sample into the previous-sample field.

### Exactly one bit-30 value set

If bit 30 differs between the samples, the function performs a transition action. This applies in either direction: clear to set or set to clear.

In order, it:

1. Reads the accepted-change accumulator at offset `0x0C`.
2. Writes that value to the captured-value field at offset `0x10`.
3. Clears the small-change accumulator by writing zero to each byte at offsets `0x08`, `0x09`, `0x0A`, and `0x0B`, in that order.
4. Calls F-df5371e0c520 if the **entry-time** mode byte was zero.

The threshold is neither read nor cleared on this path. The accepted-change accumulator is copied, not directly cleared.

Any changes made to the mode byte after its entry-time read do not change this branch’s call decision.

After the optional dependency returns, the function performs the final previous-sample store through the captured record reference.

### Both bit-30 values set: difference adjustment

If bit 30 is set in both samples, the signed difference receives at most one adjustment:

| Difference before adjustment | Adjustment |
|---|---|
| Less than −72 | Add 96 |
| From −72 through 72, inclusive | None |
| At least 73 | Subtract 96 |

This is a single correction, not repeated normalization. For example, a difference of 200 becomes 104, not 8.

The correction boundaries are asymmetric in their written comparisons but equivalent to retaining the inclusive interval −72 through 72.

### Both bit-30 values set: small-change classification

The adjusted difference is first compared against the signed threshold.

The small-change branch is selected only when both strict inequalities hold:

\[
-\text{threshold} < \text{difference} < \text{threshold}
\]

The positive-bound comparison occurs first. The negative-bound comparison is reached only if the first comparison succeeds. The supplied expression reads the threshold separately for these comparisons; it does not establish a single captured threshold value.

On this branch, the function reads the small-change accumulator, adds the adjusted difference, and writes the result back to that field.

The boundaries are excluded. A difference equal to the positive or negative threshold does not qualify.

For a stable, nonnegative threshold:

- A threshold of zero admits no difference.
- A positive threshold admits only differences strictly inside its bounds.

For an ordinary negative threshold, the inequalities cannot both hold. Negation of the minimum signed 32-bit value is a separate arithmetic uncertainty described below.

### Both bit-30 values set: accepted-change classification

If the small-change condition fails, the function evaluates an unsigned range test equivalent, for a signed 32-bit difference, to:

\[
-35 \leq \text{difference} \leq 35
\]

The test adds 35 using unsigned arithmetic and requires the result to be less than 71. Consequently, −35 and 35 are included; −36 and 36 are excluded.

If this test succeeds, the function performs the following sequence:

1. Reads the accepted-change accumulator.
2. Adds the adjusted difference.
3. Writes the result to the accepted-change accumulator.
4. Clears bytes `0x08` through `0x0B`, in increasing offset order.
5. Calls F-8b6dfdd8bb2d.
6. After that call returns, rereads the mode byte through the captured record reference.
7. Calls F-df5371e0c520 if that newly read byte is zero.

This branch has lower priority than the small-change branch. A difference satisfying both numerical ranges takes only the small-change branch.

The decision to call F-df5371e0c520 here uses the **post-call** mode byte, not the entry-time mode byte. F-8b6dfdd8bb2d can therefore affect whether the second call occurs by modifying that byte, provided the captured record remains valid.

If both classification tests fail, neither accumulator is directly updated at this stage, and neither dependency is called.

### Both bit-30 values set: accumulator cleanup

After classification and any dependency calls, the function reads the small-change accumulator through the captured record reference.

It uses an unsigned addition-and-comparison test that is equivalent to asking whether the signed accumulator is outside the inclusive interval −3 through 3.

If the value is outside that interval, it clears:

1. Threshold bytes `0x04` through `0x07`, in increasing offset order.
2. Small-change accumulator bytes `0x08` through `0x0B`, in increasing offset order.

If the value is −3, −2, −1, 0, 1, 2, or 3, neither field is cleared by this cleanup.

This check occurs on every both-set path, including when neither classification branch accepted the difference. It also occurs after dependency calls. The check therefore observes any dependency changes to the small-change accumulator rather than assuming the earlier zeroing remains intact.

### Final previous-sample store

Every normally returning path ends by writing the entry-time current sample to offset `0x18` through the captured record reference.

This final write:

- Does not reread the current sample.
- Does not use a newly selected record if a dependency changes the global reference.
- Overwrites any dependency-written value at offset `0x18` of the captured record.
- Does not occur if execution fails or a dependency does not return.

## Side effects

Direct writes are limited to:

- The small-change accumulator.
- The threshold, on cleanup.
- The accepted-change accumulator, on accepted changes.
- The captured-value field, on a bit-30 transition.
- The previous-sample field, on every normal return.

The function directly reads but does not directly write the mode byte or current sample.

Accumulator clearing and threshold clearing are explicitly bytewise operations, not single 32-bit stores. This establishes their order in the supplied representation. It does not establish atomicity, synchronization guarantees, or safe concurrent observation.

A dependency may modify the captured record, the global reference, unrelated state, or memory ownership. None of those effects can be excluded from the supplied evidence. In particular, the caller cannot rely on the small-change accumulator remaining zero after a dependency call merely because it was cleared before that call.

No direct error status, logging, exception handling, allocation, or deallocation is shown. Invalid memory access and dependency failures have no visible recovery path.

## Dependencies

### F-8b6dfdd8bb2d

Called only when:

- Both samples have bit 30 set.
- The small-change condition fails.
- The adjusted difference is within −35 through 35 inclusive.

It is invoked after updating the accepted-change accumulator and clearing the small-change accumulator. No explicit arguments are supplied, and no returned value is consumed.

Its implementation, global effects, ownership behavior, failure behavior, and return guarantee are unknown.

### F-df5371e0c520

May be called in either of two contexts:

- On a bit-30 transition, when the entry-time mode byte is zero.
- After F-8b6dfdd8bb2d returns, when a fresh read of the captured record’s mode byte is zero.

No explicit arguments are supplied, and no returned value is consumed.

The function does not itself directly call this dependency more than once per invocation. Additional calls initiated inside dependencies are outside the established contract.

Its implementation, global effects, ownership behavior, failure behavior, and return guarantee are unknown.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests. Unless noted otherwise, assume stable valid storage, ordinary two’s-complement signed interpretation, no signed arithmetic overflow, and dependencies that return without modifying relevant state.

1. **Both samples unmarked.**  
   Previous sample 7 and current sample 12 both have bit 30 clear. No dependency is called; no accumulator or threshold is changed. The previous sample becomes 12.

2. **Transition with zero mode.**  
   Previous sample 5, current sample `0x40000005`, mode zero, and accepted-change accumulator 19. The captured-value field becomes 19, the small-change accumulator becomes zero, F-df5371e0c520 is called, and the previous sample becomes `0x40000005`.

3. **Reverse transition with nonzero mode.**  
   Previous sample `0x40000005`, current sample 5, and nonzero mode. The accepted-change accumulator is copied and the small-change accumulator is cleared, but no dependency is called.

4. **Small change retained at the cleanup boundary.**  
   Both samples are marked, their difference is 2, the threshold is 5, and the small-change accumulator starts at 1. It becomes 3. Cleanup preserves both that value and the threshold.

5. **Small change triggers cleanup.**  
   Under the same conditions, an initial small-change accumulator of 2 becomes 4. Cleanup then clears both the threshold and small-change accumulator.

6. **Strict threshold boundary takes the accepted-change branch.**  
   Both samples are marked, their adjusted difference is 5, and the threshold is 5. The strict small-change test fails. The accepted-change accumulator increases by 5, the small-change accumulator is cleared, and F-8b6dfdd8bb2d is called.

7. **Accepted-change range boundaries.**  
   With threshold zero, adjusted differences −35 and 35 enter the accepted-change branch. Adjusted differences −36 and 36 do not. The latter cases still undergo accumulator cleanup.

8. **Adjustment boundary.**  
   Raw difference 72 remains 72; raw difference 73 becomes −23. Raw difference −72 remains −72; raw difference −73 becomes 23.

9. **No repeated adjustment.**  
   A raw difference of 200 becomes 104. With threshold zero, it satisfies neither classification branch. An existing small-change accumulator of −4 is nevertheless cleared along with the threshold.

10. **Dependency-controlled mode decision.**  
    On the accepted-change branch, if F-8b6dfdd8bb2d changes the captured record’s mode byte from nonzero to zero, F-df5371e0c520 is called. A change from zero to nonzero suppresses that second call.

11. **Entry sample survives a dependency update.**  
    If a dependency changes the current-sample field, the final previous-sample write still uses the sample read before the dependency call.

## Uncertainties

The following are **documented_only**, not guessed guarantees:

- The record’s application-level identity, units, initialization rules, and lifetime.
- Whether 96 represents a cycle length and whether bit 30 represents validity, activity, or another state.
- Exact target behavior for converting an out-of-range unsigned difference to signed.
- Overflow behavior for signed accumulator additions and negation of the minimum signed threshold. The supplied representation does not justify a portable signed-overflow contract.
- Whether the global record reference can change asynchronously.
- Any concurrency, volatile-access, atomicity, or memory-order guarantees.
- Dependency side effects, ownership changes, failure modes, and whether they return.
- Whether dependencies preserve the validity of the captured record.
- Any external invariant restricting sample values, thresholds, or accumulator ranges.

A zero threshold is not an overall disable switch: it prevents the small-change branch, but differences within −35 through 35 can still take the accepted-change branch.

## Evidence

Established static evidence consists of the supplied function representation: field offsets, access widths, branch conditions, arithmetic constants, bytewise clearing order, dependency call positions, and final store placement.

The distinction between entry-time and post-dependency observations follows directly from where reads occur. The dependency identities are expressed only through the supplied opaque IDs.

Cyclic tracking is inferred intent, not an established domain description. No runtime tests, execution traces, dependency implementations, or independent layout declarations were available or used.
