<!-- Trusted provenance: F-7c0e8dced50a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7c0e8dced50a

## Purpose

F-7c0e8dced50a performs a destructive, comparison-directed transformation of two word-based objects and, on some paths, an object selected through a context.

**Inferred intent:** the alignment, conditional subtraction-like operations, and low-bit accumulation resemble unsigned binary division, with the first operand acting as a remainder workspace and the context-selected object accumulating a quotient. This interpretation is not sufficient to promise division semantics: the dependencies, numeric encoding, and caller invariants are unknown.

**Established behavior:** the function selects paths using dependency results and signed header products, issues ordered dependency calls, directly modifies selected object fields, and returns a 32-bit status-like value. Failures do not roll back preceding mutations.

## Interface

### Inputs and directly accessed layouts

The function accepts three inputs:

| Input | Visible representation | Use |
|---|---|---|
| **A**, first operand | Pointer to a word-based object | Passed to dependencies; header and backing-word pointer read directly |
| **B**, second operand | Pointer to a word-based object | Passed to dependencies; header and backing-word pointer read directly |
| **C**, context | 32-bit address-like value | Zero selects an immediate return; otherwise contains a selected-object pointer |

The directly accessed operand layout is:

| Byte offset | Field |
|---|---|
| `0x00` | 32-bit header |
| `0x04` | 32-bit address of backing words |

Backing words are 32 bits, with four-byte spacing.

At byte offset `0x34`, C contains a 32-bit pointer to another object, called **Q** in this contract. Q is a role, not a pointer captured once for the whole invocation. Each explicit context lookup selects the object present at that time.

No explicit capacity, object size, ownership indicator, or aliasing restriction is supplied. Numeric headers are not independently established to be lengths, signs, or allocation sizes.

### Return value

The return width is 32 bits.

| Condition | Result |
|---|---|
| C is zero | Current null-context result global |
| Header eligibility fails | Current ineligible-operands result global |
| A dependency whose result is checked for success returns nonzero | `1` |
| A locally recognized completion path succeeds | `0` |

The two global result values are unknown and may themselves equal `0` or `1`. Comparison-like and factor-producing dependency results are interpreted separately; they are not uniformly treated as statuses.

### Global values and capture rules

The visible global words have these roles:

- **Null-context result** and **ineligible-operands result** supply branch-specific returns.
- **Header multiplier** participates in eligibility and later count calculations.
- **Limb multiplier** participates in highest-word analysis and later low-word updates.
- **Store multiplier** participates in direct low-word writes.

The access distinctions are part of the contract:

| Value | Capture or read point | Subsequent use |
|---|---|---|
| Saved header multiplier | Immediately after the preliminary call on A, before its status is rejected or accepted | Both alignment counts |
| Header multiplier for eligibility | Read in the header-gate expressions | Signed eligibility products |
| Saved limb multiplier | Immediately after the second comparison-like call, before testing whether its result is exactly `1` | Loop low-word updates |
| B-scan limb multiplier | Fresh global read when initializing B’s highest-word scan | B’s transformed highest word |
| A-scan limb multiplier | Another fresh global read when initializing A’s highest-word scan, after B’s scan | A’s transformed highest word |
| Store multiplier | Read for each direct low-word update | That update’s stored value |

These reads must not be collapsed into one immutable multiplier. In particular, the saved limb multiplier is read even when the second comparison result is exactly `1` and the function never performs alignment.

### Widths and arithmetic limits

Headers, backing words, visible pointers, and dependency results are 32 bits. Eligibility and loop-distance decisions use signed 32-bit interpretations; scan words and low-bit operations use 32-bit unsigned bit patterns.

The first-subtraction state is held in a wider word but explicitly masked to eight bits when incremented.

Products, count arithmetic, four-byte indexing, bit-length arithmetic, and the final distance subtraction are not unbounded mathematical operations. Exact signed-overflow behavior and supported address ranges are not established by the supplied representation. Cases requiring assumptions about those properties remain `documented_only`.

## Behavior

### Entry and preliminary acceptance

A zero C causes an immediate return of the null-context result global. This path makes no visible dependency calls and does not dereference A or B.

For nonzero C, preliminary acceptance is attempted on B through F-41aa86c4f954, followed by A through the same dependency. Only a result of exactly zero is accepted. A failure on B returns `1` without calling the dependency on A.

After the call on A, the function saves the header multiplier before deciding whether A’s preliminary result requires return `1`. These dependencies are not established to be read-only or to validate every later direct access.

### Header eligibility and zero-A handling

Continuation requires both of the following:

- The signed 32-bit product of B’s header and the header multiplier read for that expression is strictly positive.
- If B passes, the corresponding signed product for A is nonnegative.

A’s eligibility product is not evaluated when B fails. Rejection returns the ineligible-operands result global.

The saved header multiplier used later for counts is distinct from the global reads used by these eligibility expressions. Even if stable globals are expected operationally, stability is not established here.

If A’s eligible product is zero, the function selects Q through C and directly writes zero to Q’s header. It then calls F-c64ad9e9ad7c with A followed by B. A nonzero result returns `1`.

A successful call does not finish the operation. Execution continues into comparison-directed behavior. The direct header clear does not itself clear Q’s backing pointer or backing words, and does not demonstrate storage release.

### Comparison-directed exits

The first comparison-like call is F-61e238108dfa with B followed by A.

If this result is zero, F-d72f047fba8b is called on B and then on A. The product of those results is supplied as the scalar argument to F-1661479163f4, with A as destination. A zero result from that final dependency returns `0`; any nonzero result returns `1`.

This branch does not replace the destination with Q. It also does not read the limb multiplier for alignment or low-word accumulation.

If the first comparison result is nonzero, F-61e238108dfa is called again with B followed by A. The function then immediately saves the limb-multiplier global, before testing the second comparison result.

A second result of exactly `1` selects Q through C and calls F-1661479163f4 with scalar zero and Q. Its zero result produces return `0`; otherwise the function returns `1`. The saved limb-multiplier read has already occurred, although its value is unused on this exit.

Every other second result enters alignment, including zero. The second call is observable and cannot be replaced by reuse of the first result.

### Alignment analysis

Two signed 32-bit counts are formed from the saved header multiplier and the operands’ current headers: B’s count first, then A’s. Both counts and their tentative lengths—32 times each count—are formed before the scans.

For B, the selected backing word is at index B-count minus one. B’s transformed scan word uses a **fresh read** of the limb-multiplier global, not the previously saved limb multiplier.

B’s transformed word is shifted left until bit 31 is set. Its tentative length decreases by one for each shift. Once B’s scan finishes, A undergoes the corresponding analysis using its own selected backing word and **another fresh read** of the limb-multiplier global.

Thus, three limb-multiplier observations are distinct:

1. The saved value immediately following the second comparison.
2. The value used to initialize B’s scan.
3. The later value used to initialize A’s scan.

The representation does not establish a relative evaluation order between the multiplier read and backing-word read within an individual multiplication expression.

The alignment distance is A’s resulting length minus B’s resulting length. The calculation requires valid counts, readable selected words, and arithmetic within the supported execution model.

A transformed scan word of zero cannot acquire bit 31 through repeated left shifts. The visible scan has no zero-word exit. Whether the surrounding signed decrement eventually overflows, traps, or behaves otherwise is not established; normal completion is not guaranteed for that case.

### Alignment call and Q initialization

F-9c01f3513f11 receives B and the computed alignment distance. A nonzero result returns `1`.

After a successful call, the function selects Q through C and calls F-1661479163f4 with scalar zero and Q. A nonzero result returns `1`.

The first-subtraction state starts at zero. Only after the preceding calls succeed does the function test whether the signed alignment distance is negative. A negative distance therefore does not bypass the initial B operation or Q initialization; it causes return `0` after them.

The direction and numeric meaning of F-9c01f3513f11 are inferred, not independently established.

### Iterative transformation

For each nonnegative distance, the function first selects Q through C and calls F-9c01f3513f11 with Q and `1`. A nonzero result returns `1` before that iteration’s comparison.

The iteration then calls F-61e238108dfa with B followed by A. A result of exactly `1` skips the subtraction-associated work. Every other result selects that work; it is not treated as a comparison error status.

After the comparison and any subtraction-associated work, a nonzero current distance causes F-347405e096bf to be called with B and `1`. A nonzero result returns `1`. At distance zero this dependency is omitted.

The distance then decreases. Under supported arithmetic and successful dependencies, an initial nonnegative distance produces one iteration for each position down through zero, followed by return `0`.

### Subtraction-associated updates

When selected by an iteration’s comparison, F-8a9589a9b71f receives A followed by B. A nonzero result returns `1` without the following direct low-word write.

On success, Q is selected again through C. The function reads Q’s backing pointer and first 32-bit word. That word is multiplied by the **saved limb multiplier captured immediately after the second pre-alignment comparison**. Bit zero of the resulting 32-bit pattern is forced to one. Multiplication by the store multiplier supplies the replacement value written to the same backing-word address.

Neither scan’s fresh multiplier is substituted for this saved value.

On the first such event only, the function subsequently selects Q again and calls F-1661479163f4 with scalar `1` and Q. The first-subtraction state advances from zero to one, with an eight-bit mask, even if that call reports failure. A nonzero result then returns `1`.

The direct low-word write precedes this first-event call. The call may replace that data or target a different Q if the context selection has changed. Later subtraction-associated events omit the scalar-one call.

## Side effects

### Direct mutations

The function directly performs only these object writes:

- Zeroing Q’s header on the eligible-zero-A path.
- Replacing Q’s first backing word during subtraction-associated iterations.

Dependencies may mutate A, B, Q, their backing storage, globals, or context state. Their mutation boundaries are unknown.

The operation is not transactional. For example, B’s alignment operation precedes Q initialization, and Q’s per-iteration operation precedes the comparison and possible subtraction failure. Earlier effects are not undone when a later checked result produces return `1`.

No explicit rollback, cleanup, diagnostic output, or exception translation is visible.

### Ownership and lifetime

No explicit allocation or release operation is established in this function. That does not imply dependencies allocate nothing, release nothing, or preserve supplied storage.

The returned value is status-like, not an object pointer. No ownership transfer is established.

Every reached direct access requires valid memory at that time. In particular, context rereads and post-dependency backing-word accesses require continued validity that cannot be inferred from earlier checks alone.

### Aliasing and mutable state

No checks prohibit overlap among A, B, Q, or their backing arrays. Aliasing may cause a write through one role to affect later reads through another. Conventional arithmetic behavior for overlapping objects is therefore not promised.

The function does not snapshot all inputs or globals. Saved multipliers remain saved; explicit later reads are separate observations. No synchronization, atomicity, or concurrent-mutation guarantee is visible.

## Dependencies

| Opaque ID | Established use |
|---|---|
| F-41aa86c4f954 | Preliminary B-then-A acceptance; exactly zero permits continuation |
| F-c64ad9e9ad7c | Receives A and B after the eligible-zero-A header clear |
| F-61e238108dfa | Receives B and A; results select comparison-directed paths |
| F-d72f047fba8b | Supplies B’s factor, then A’s factor, on the first-comparison-zero branch |
| F-1661479163f4 | Receives a scalar followed by a destination; checked success is exactly zero |
| F-9c01f3513f11 | Receives B and an alignment distance, or Q and `1`; checked success is exactly zero |
| F-8a9589a9b71f | Receives A and B before direct low-word accumulation |
| F-347405e096bf | Receives B and `1` between nonzero loop positions |

Names such as “comparison,” “assignment,” “shift,” and “subtraction” describe inferred roles, not verified dependency contracts. No dependency is assumed to preserve globals, context selection, ownership, or memory validity.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests.

1. **Null context.** With C equal to zero, return the null-context result global without operand checks or operand dereferences.

2. **Preliminary rejection.** If B’s preliminary dependency returns any nonzero 32-bit value, return `1` without checking A.

3. **Header rejection.** With preliminary checks successful and B’s signed eligibility product nonpositive, return the ineligible-operands global without evaluating A’s eligibility product or making comparison-stage calls.

4. **Zero-A failure after mutation.** If eligible A has product zero, Q’s header is cleared before F-c64ad9e9ad7c. A nonzero result returns `1` without restoring that header.

5. **First comparison zero.** Extract B’s factor before A’s, then pass their product to F-1661479163f4 with A as destination. Do not substitute Q.

6. **Second comparison exactly one.** After a nonzero first comparison and second result `1`, read and save the limb multiplier before selecting Q and requesting scalar-zero assignment. No highest-word scan occurs.

7. **Distinct multiplier observations.** If the saved limb-multiplier observation is \(s\), B’s scan observation is \(b\), and A’s later scan observation is \(a\), initialize the scan words using \(b\) and \(a\), respectively. Any later direct low-word update uses \(s\), regardless of whether these values differ.

8. **Negative alignment.** Given valid alignment analysis yielding a negative distance, successful B alignment and Q initialization still occur before return `0`. No loop-position operation occurs.

9. **Single-position iteration.** With initial distance zero and successful dependencies, Q receives one per-position F-9c01f3513f11 call. No F-347405e096bf call follows that position.

10. **First subtraction ordering.** After a successful F-8a9589a9b71f, the direct low-word write precedes the first scalar-one assignment call. Failure of that assignment returns `1` without rollback.

11. **Zero transformed highest word.** If a scan initializes to zero, repeated left shifts cannot satisfy its high-bit condition. Successful completion is not required for that unsupported case.

## Uncertainties

The following remain `documented_only`:

- Global values, initialization, and permitted mutation.
- Numeric encoding, normalization rules, and valid header/count ranges.
- Dependency implementations and comparison-result meanings.
- Signed-overflow behavior and supported address arithmetic.
- Allocation capacities and lifetime guarantees across calls.
- Supported aliasing arrangements.
- Dependency ownership effects and changes to context selection.
- Behavior on invalid memory, unsupported representations, or nonterminating scans.
- Whether the inferred quotient/remainder interpretation holds for any particular caller domain.

No conventional division contract is established without those additional prerequisites.

## Evidence

**Established static evidence:** the supplied representation supports the stated branch conditions, field offsets, access widths, dependency argument order, direct mutations, status handling, and loop ordering. It also places the saved limb-multiplier read immediately after the second comparison, before its exactly-one test, and shows separate fresh multiplier reads for B’s and A’s scan initialization.

**Inferred intent:** the highest-word alignment, conditional subtraction-like call, and accumulated low bits suggest destructive binary division.

**Unknown:** dependency semantics, caller invariants, global initialization, ownership rules, and behavior outside the supported visible operations.

No runtime tests have been performed.
