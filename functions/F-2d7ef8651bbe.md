<!-- Trusted provenance: F-2d7ef8651bbe; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2d7ef8651bbe

## Purpose

F-2d7ef8651bbe performs an in-place, 80-round transformation of a five-word state using a sixteen-word input block. It first expands the input block into an 80-word schedule, then processes that schedule, and finally adds the resulting working state into the five state words.

The established structure is SHA-1-like: the schedule recurrence, rotations, round functions, and feed-forward arrangement match that compression design. Identification as a standard SHA-1 compression operation is conditional, however, because the values of four externally stored round constants are not supplied.

This operation does not constitute a complete hash interface. It does not visibly perform padding, track message length, initialize a standard state, convert byte order, or serialize a digest.

## Interface

### Argument and result

The function accepts one pointer to a contiguous, writable region of 32-bit unsigned words. It has no explicit length argument and no return value.

The region must contain at least 85 words, totaling 340 bytes under the established 32-bit word model.

| Word indices | Byte offsets | Role |
|---|---:|---|
| 0–4 | 0–19 | Five input state words, replaced by feed-forward results |
| 5–20 | 20–83 | Sixteen input schedule words, preserved |
| 21–84 | 84–339 | Sixty-four generated schedule words, overwritten |

Offsets are relative to the supplied pointer. No behavior depends on the initial contents of words 21–84: every such word is generated before being consumed.

The function also accesses four externally stored, 32-bit round operands. Their values, mutability, and placement relative to the supplied region are not established.

### Preconditions

For defined operation:

- The pointer must designate accessible storage for all 85 words.
- The storage must support the required 32-bit reads and writes with the target platform’s alignment requirements.
- Words 0–20 must contain valid input values.
- The four external round operands must be readable when used.
- The storage must remain valid for the entire invocation.

There is no visible pointer validation, capacity check, alignment check, or recoverable error path.

The caller retains ownership of the region. The function neither allocates nor releases memory.

### Arithmetic model

All state values, schedule values, bitwise operations, rotations, and arithmetic results are 32 bits wide.

Addition and subtraction wrap modulo \(2^{32}\). Right shifts are logical. A left rotation by \(r\) moves the high \(r\) bits into the low positions within the same 32-bit word.

No byte-order conversion is visible. The function consumes words as represented by the target’s word loads. A caller supplying bytes must arrange their interpretation separately.

## Behavior

### 1. Expand the schedule before reading the working state

Let \(W_0\) through \(W_{79}\) denote the schedule words. Schedule word \(W_t\) occupies region word \(5+t\).

The first sixteen schedule words are supplied by the caller. For each \(t\) from 16 through 79, in increasing order, the function stores:

\[
W_t =
\operatorname{ROL}_{32}
\left(
W_{t-3}\oplus W_{t-8}\oplus W_{t-14}\oplus W_{t-16},
1
\right)
\]

Each generated word is written before the next schedule word is generated. Consequently, later schedule entries consume earlier generated entries rather than their pre-call contents.

The first generated word uses \(W_{13}, W_8, W_2,\) and \(W_0\). The last generated word uses \(W_{76}, W_{71}, W_{65},\) and \(W_{63}\).

All 64 schedule writes occur before initialization of the five working state values. This ordering is part of the established behavior; it is not an interleaving of schedule generation and compression.

Within an individual schedule expression, the supplied representation does not establish a machine-level ordering among the four source reads. It does establish that the destination is written after its contributing values are obtained.

### 2. Initialize the working state

After schedule expansion, the five state words are loaded in this visible order:

1. Region word 1.
2. Region word 2.
3. Region word 3.
4. Region word 0.
5. Region word 4.

For the mathematical description below, name the resulting working values \(A,B,C,D,E\), corresponding respectively to region words 0, 1, 2, 3, and 4.

These symbols describe the algorithm’s working state, not additional caller-visible storage.

### 3. Execute 80 rounds

Rounds are numbered 0 through 79. Each round consumes the corresponding schedule word \(W_t\).

For each round, a new leading value is formed from:

- \(A\) rotated left by 5;
- a phase-specific Boolean function of \(B,C,D\);
- \(E\);
- \(W_t\);
- the applicable external round operand.

The remaining working values advance according to the same transformation in every phase:

\[
(A,B,C,D,E)_{\text{next}}
=
\left(
T,\,
A,\,
\operatorname{ROL}_{32}(B,30),\,
C,\,
D
\right)
\]

All values on the right side refer to the state entering that round. In particular, the rotation applies to the previous \(B\), not to a newly advanced value.

The four phases are as follows.

| Rounds | Boolean function | External operand treatment |
|---|---|---|
| 0–19 | \((B\land C)\lor(\neg B\land D)\) | Add \(K_0\) |
| 20–39 | \(B\oplus C\oplus D\) | Add \(K_1\) |
| 40–59 | \((B\land C)\lor(B\land D)\lor(C\land D)\) | Add \(K_2\) |
| 60–79 | \(B\oplus C\oplus D\) | Subtract \(S_3\) |

Here \(K_0,K_1,K_2,S_3\) name the four external operand locations, not established numeric constants.

For rounds 0–59:

\[
T =
\operatorname{ROL}_{32}(A,5)
+ f_t(B,C,D)
+ E
+ W_t
+ K_{\text{phase}}
\pmod{2^{32}}
\]

For rounds 60–79:

\[
T =
\operatorname{ROL}_{32}(A,5)
+ (B\oplus C\oplus D)
+ E
+ W_t
- S_3
\pmod{2^{32}}
\]

The subtraction in the last phase is significant. Treating its operand as an additive round constant requires taking its additive inverse modulo \(2^{32}\).

Each phase performs exactly 20 rounds. There is no visible input-dependent branch, early completion, or alternate round count.

The external operand is referenced within each round’s expression. The contract does not assume that its value is frozen at function entry. Likewise, no machine-level ordering is established among the schedule read, external operand read, and other operands within one arithmetic expression.

### 4. Feed the working state back into the region

After round 79, the final working values are added into the state region. The visible update order is:

1. Add final \(B\) to region word 1.
2. Add final \(C\) to region word 2.
3. Add final \(D\) to region word 3.
4. Add final \(E\) to region word 4.
5. Add final \(A\) to region word 0.

Every addition wraps modulo \(2^{32}\).

These operations read the corresponding region words at the time of feed-forward. They are not expressed as additions to a separately retained copy of the initial five words.

With ordinary, non-overlapping storage and no outside modification, the distinction does not affect the result: the compression rounds do not write the five state words before feed-forward. It does matter when describing observation order or unsupported concurrent interference.

The function then returns without a result value.

## Side effects

### Caller-provided storage

The function performs:

- 64 schedule writes, to region words 21–84 in ascending order;
- five feed-forward writes, to region words 1, 2, 3, 4, and then 0.

Words 5–20 are read but not written. The expanded schedule remains in words 21–84 after return. Neither the input schedule nor the generated schedule is erased.

No write outside words 0–84 is visible in the supplied representation.

### External state

The four external round operand locations are read. No direct write to those locations is shown.

Their disjointness from the caller’s region is not established. If an external operand overlaps a region location written during schedule expansion, the write precedes the rounds and can affect the operand subsequently consumed. No non-aliasing guarantee should be inferred.

There are no visible function calls, allocation operations, release operations, I/O operations, callbacks, or explicit synchronization.

### Failure and interruption

No normal error result is provided.

Invalid or inaccessible storage can cause execution failure according to the target environment. The supplied evidence does not define the fault mechanism, transactional behavior, rollback, or contents remaining after interruption.

Because schedule writes precede compression and final state writes occur individually, callers must not assume that an interrupted invocation leaves the region unchanged.

No concurrency or atomicity guarantee is established.

## Dependencies

No called-function dependencies are visible. Accordingly, there are no additional opaque function IDs to list.

The operation depends on four external 32-bit data operands:

- \(K_0\), used in rounds 0–19;
- \(K_1\), used in rounds 20–39;
- \(K_2\), used in rounds 40–59;
- \(S_3\), subtracted in rounds 60–79.

Their actual values are a prerequisite for predicting concrete final state values.

If these operands are stable and have the values

\[
K_0=\text{0x5A827999},\quad
K_1=\text{0x6ED9EBA1},\quad
K_2=\text{0x8F1BBCDC},\quad
S_3=\text{0x359D3E2A},
\]

then the last phase’s subtraction is equivalent to addition of \(\text{0xCA62C1D6}\), and the described word-level transformation matches the standard SHA-1 compression round structure.

Those numeric values are conditional reference values, not values established for this function’s external storage.

## Acceptance examples

The following are static acceptance conditions, not results of executed tests.

### Zero input schedule

Given sixteen zero input schedule words, every generated schedule word must be zero.

This conclusion is independent of the initial five state words and the external round operands. It does not imply that the final state is zero.

### First generated word and rotation wrap

Given \(W_0=\text{0x80000000}\) and all other initial schedule words zero:

\[
W_{16}=\text{0x00000001}
\]

The high bit wraps into the low bit. A shift that discards the high bit would not satisfy the contract.

For the same input:

\[
W_{17}=0,\qquad W_{18}=0,\qquad
W_{19}=\text{0x00000002}.
\]

The \(W_{19}\) result demonstrates consumption of the newly generated \(W_{16}\).

### Independence from scratch contents

For two otherwise identical, isolated invocations with equal words 0–20 and equal external operand values throughout execution, differing initial contents in words 21–84 must not change the final state or expanded schedule.

This condition assumes the external operands do not alias those differing scratch locations.

### Preserved input block

For a valid isolated invocation, words 5–20 must retain their entry values after return, regardless of the state words and round operands.

### Modular feed-forward

If final working \(B\) is 1 and region word 1 is \(\text{0xFFFFFFFF}\) immediately before its feed-forward update, that update must store zero.

This is a conditional boundary example; it does not assert a particular input that produces final \(B=1\).

### Standard-compression compatibility

If the external operands have the reference values listed under Dependencies, remain stable, and do not alias the region, the final five words must equal standard SHA-1 compression applied to the supplied five-word state and sixteen already interpreted input words.

This condition does not require or imply padding, initialization, or byte conversion inside this function.

### Observable write sequence

An observer capable of distinguishing the visible stores must encounter the schedule stores in ascending word order, followed by state stores in the order 1, 2, 3, 4, 0.

This is a source-facing ordering requirement, not a claim about cross-thread visibility, memory barriers, or atomic observation on a particular processor.

## Uncertainties

The following items are **documented_only**:

- The actual values and mutability of the four external operands.
- The target’s byte order and precise alignment requirements.
- The caller’s intended interpretation of the input block.
- Whether callers always provide storage disjoint from the external operands.
- The target ABI details beyond the visible single-pointer interface and absence of a result value.
- Fault behavior for invalid storage.
- Machine-instruction ordering of reads within individual expressions.
- Concurrency semantics and visibility to unsynchronized observers.

**Inferred intent:** this is a SHA-1-style compression operation with caller-owned schedule workspace. Full identification as standard SHA-1 requires confirmation of the external operand values and caller-side word interpretation.

No unknown dependency behavior is presumed: no callees are visible, and no ownership transfer or hidden memory management is inferred.

## Evidence

Established static evidence consists of the supplied function representation:

- A schedule-generation loop covers indices 16 through 79 and writes the corresponding 64 schedule words.
- Its four-input XOR recurrence uses schedule offsets 3, 8, 14, and 16, followed by a one-bit rotation.
- Five state words are loaded only after schedule expansion.
- Four consecutive phases each execute 20 rounds.
- The phases use choice, parity, majority, and parity functions respectively.
- Working-state rotations are left 5 and left 30 over 32-bit values.
- Three phases add an external operand; the final phase subtracts one.
- Five modular feed-forward stores complete the operation in the stated order.
- No calls, validation branches, allocation, release, or explicit error reporting are visible.

The contract separates these established properties from the inferred algorithm identity and the unresolved external data values.

No runtime tests have been performed.
