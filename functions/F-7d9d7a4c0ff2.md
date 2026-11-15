<!-- Trusted provenance: F-7d9d7a4c0ff2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7d9d7a4c0ff2

## Purpose

F-7d9d7a4c0ff2 is an apparently non-returning sequence of indirect dependency calls, computed-address memory reads, and byte and 32-bit memory writes. The visible body repeats indefinitely. At each iteration boundary, it replaces several working inputs with values derived from a dependency result and forces the working fourth input to zero.

No application-level purpose is established. In particular, the evidence does not establish that the computed addresses identify ordinary objects, arrays, device registers, or allocated storage. Descriptions such as “initialize,” “release,” or “publish” would imply semantics not supported by the supplied material.

This contract describes the visible operation sequence. It does not establish that this sequence is reachable or executable on a particular machine.

## Interface

### Explicit inputs

The visible interface contains four arguments:

| Argument | Visible width | Established use |
|---|---:|---|
| First | 32 bits | No explicit use in the visible body. |
| Second | 32 bits | Passed twice to the first dependency invocation; subsequently written to a computed address. Replaced at the iteration boundary. |
| Third | 32 bits | Passed as the third argument to the first dependency invocation. Replaced at the iteration boundary. |
| Fourth | 32 bits | Its low eight bits supply four direct byte writes early in an iteration. The working argument becomes zero before the next iteration. |

The second and third arguments have signed integer representations in the supplied listing, but their externally meaningful signedness is not established. No range checks are visible.

“No explicit use” of the first argument does not prove that its incoming register contents are unobservable to dependencies. The recovered dependency interfaces are incomplete.

### Additional entry state

Execution also depends on two incoming machine-state values not represented as explicit arguments:

- An incoming register value used to construct an address base by multiplication by \(2^{28}\).
- An incoming link/control value used to calculate the first indirect call target.

Neither has a visible initialization in the function. They are entry prerequisites, not constants or implicit zeroes.

A further register-derived value is passed to a dependency later in the iteration. Its defining operation is not explicit in the listing.

### Output and termination

There is no visible normal return, return value, loop exit, or error result. Every completed iteration starts another iteration.

A dependency that does not return, an invalid instruction target, or a memory fault could prevent further execution. The supplied evidence defines no recovery behavior for those cases.

### Width and address conventions

The descriptions below use:

- **Byte**: eight bits.
- **Word**: 32 bits.
- **Wide result**: a recovered 64-bit dependency result, subsequently separated into low and high 32-bit halves.
- **Byte load**: zero-extension of an eight-bit memory value before its use in address arithmetic.
- **Byte store**: retention of only the low eight bits of the supplied value.

For conditional arithmetic examples, word arithmetic retains the low 32 bits, and right shifts of unsigned quantities are logical. This is the apparent machine-word interpretation. Exact behavior of signed overflow and signed left shifts is not established independently of the supplied representation.

Addresses below are computed addresses, not offsets from an assumed allocation.

## Behavior

### General ordering rule

The visible sequence consists of one unconditional loop with no data-dependent branch. Dependency results affect addresses, stored values, and subsequent arguments, but not a visible branch condition.

All listed memory operations occur in the stated order, with dependency invocations interspersed between them. No dependency may be assumed to preserve memory, globals, or unlisted register state.

Several call groups contain a leading invocation with explicit arguments, followed by four invocations with no explicit arguments. The last of those four often supplies the next result used below. “No explicit arguments” describes only the supplied representation; it is not proof that the actual dependency has a zero-argument interface.

### 1. Establish the iteration’s initial state

Capture the low byte of the current fourth argument. Construct the iteration base from the current incoming-register working value multiplied by \(2^{28}\).

The first dependency invocation receives, in order:

1. The current second argument.
2. The current second argument again.
3. The current third argument.

Its target depends on the incoming link/control state. Four further calls follow. Let **initial index** denote the last call’s word result.

Write the current second argument as a word to:

\[
4 \times \text{initial index} + 20.
\]

A dependency receives four times the initial index. Four further calls follow. Preserve the last result as **initial packed value**.

Write the captured input byte to the iteration base plus 17, then to the iteration base plus 1. These are separate byte writes, in that order.

A dependency receives the initial packed value shifted left by two. A following dependency produces a word that is passed to another dependency. Four further calls follow, producing a second index.

Repeat the two direct byte writes: iteration base plus 17 first, then iteration base plus 1.

A dependency receives the second index shifted left by two. Four further calls produce **wide-stride index**.

Write a word to:

\[
16 \times \text{wide-stride index} + 20.
\]

The written value is the initial index shifted left by 31. Under 32-bit truncation, only the initial index’s least significant bit contributes.

Next, read one word from:

\[
\left((\text{initial packed value}\mathbin{\&}0x3fffffff)\gg20\right)+20.
\]

The addition is not scaled by four. The address can therefore be unaligned. The mask and shift produce an offset from 0 through 1023, so this read addresses the range 20 through 1043 under the stated word interpretation.

Retain the loaded word as **loaded packed value**.

### 2. Establish the early indirect stores

A dependency receives four times the wide-stride index. The following call produces **retained control value**.

Another dependency receives:

\[
(\text{loaded packed value}\gg22)\ll28.
\]

Four further calls produce **clear index**.

Write a zero word at four times the clear index plus 20.

A dependency then receives four times the clear index. Four further calls produce **byte-source value**.

Read a byte from iteration base plus 1. Use that zero-extended byte directly as an address base, and write one byte at that base plus 17. The written byte is the low byte of byte-source value shifted left by two.

This is not a pointer-sized load: the destination is in the numeric range 17 through 272.

A dependency receives byte-source value shifted left by two. The next call produces a word that is passed to another dependency together with byte-source value shifted left by 30. Four further calls produce another index.

Read iteration base plus 1 again and retain that byte as **saved byte base**. Write a byte at saved byte base plus 17 containing the low byte of:

\[
(\text{byte-source value}\mathbin{\&}0x3fffffff)\gg20.
\]

A dependency receives the latest index shifted left by two. Four more calls occur; their final return is not explicitly used.

Write a word at saved byte base plus 20 containing retained control value shifted left by two.

The saved byte base is reused for this word store. It is not reloaded after the intervening calls. The word address lies between 20 and 275 and is not necessarily aligned.

### 3. Consume the global word and a wide result

Read the visible global word once and retain its value locally. This read occurs after the preceding word store and before the next call group.

A dependency receives four times the wide-stride index. Four further calls produce the first wide result.

Define **first shifted low** as the low word of this result shifted left by two.

Read iteration base plus 1 again. Write a byte at the loaded byte plus 17 containing the low byte of the captured global word shifted right by 22.

A dependency receives the high word of the wide result. A following call produces a word, which is passed to another dependency together with first shifted low shifted left by 28. Four further calls produce an index.

Reload the byte at iteration base plus 1 and write zero at that byte plus 17.

A dependency receives the latest index shifted left by two. Four further calls produce another index. A dependency then receives that new index shifted left by two twice, as two equal arguments.

Four further calls produce **first stored result**. Define **first stored word** as that result shifted left by two.

Write first stored word to:

\[
4096 \times \text{first shifted low}+20.
\]

A dependency receives first shifted low shifted left by 14. Four further calls produce an index.

Write the same first stored word to four times that index plus 20.

### 4. Perform the second wide-result sequence

A dependency receives four times the preceding index. Four further calls produce the second wide result.

Define **second shifted low** as its low word shifted left by two.

Read a byte from:

\[
16384 \times \text{first shifted low}+1.
\]

At the loaded byte plus 17, write the low byte of:

\[
(\text{first stored word}\mathbin{\&}0x3fffffff)\gg20.
\]

A dependency receives the high word of the second wide result. A following call produces a word that is passed to another dependency.

A subsequent call produces another wide result. Its low and high words are passed separately, in that order, to a dependency. Two more calls follow; the latter supplies an index.

Read a byte from \(2^{28}\) times second shifted low plus 1. Write zero at the loaded byte plus 17.

A dependency receives the latest index shifted left by two. Four further calls produce **middle index**.

Read a byte from four times second shifted low plus 1. Write zero at the loaded byte plus 17.

A dependency receives second shifted low shifted left by 10. A following call produces a word that is passed to another dependency together with retained control value shifted left by 28. Four further calls produce another index.

Again read a byte from four times second shifted low plus 1, and write zero at the loaded byte plus 17.

A dependency receives the latest index shifted left by two. Four further calls occur.

Define **middle stored word** as middle index shifted left by 14. Write it to:

\[
4096 \times (4 \times \text{middle index})+20.
\]

A dependency receives four times middle index shifted left by 14. A further call occurs. Another dependency receives middle stored word shifted left by 28 and an incompletely recovered register-derived second argument. Four further calls produce an index.

Write middle stored word to four times that index plus 20.

### 5. Perform the final wide-result and word-store sequences

A dependency receives four times the preceding index. Four further calls produce **final addressing index**.

Read a byte from four times second shifted low plus 1. At that byte plus 17, write the low byte of:

\[
(\text{middle stored word}\mathbin{\&}0x3fffffff)\gg20.
\]

A dependency receives middle index shifted left by 26. The next call produces a word. Pass that word and four times final addressing index shifted left by 28 to another dependency.

The following call produces a wide result. Pass its low and high words separately to another dependency. Two further calls produce an index.

Reload the byte from four times second shifted low plus 1 and write zero at the loaded byte plus 17.

A dependency receives the latest index shifted left by two. Four further calls produce **final payload index**. Define **final payload word** as that index shifted left by two.

Write final payload word to:

\[
4096 \times (4 \times \text{final addressing index})+20.
\]

The next dependency receives four explicit arguments:

1. \(1024 \times (4 \times \text{final addressing index})\).
2. The same value again.
3. Final addressing index shifted left by 14.
4. Final payload word.

Four further calls produce an index. Write final payload word to four times that index plus 20.

A dependency receives four times that index. Four further calls produce **self-derived index**.

Define **self-derived word** as four times self-derived index. Write self-derived word to 16 times self-derived index plus 20.

Set the working fourth argument to zero.

A dependency receives self-derived word three times, as three equal arguments. Four further calls produce an index. Write self-derived word to four times that index plus 20.

A dependency receives four times the latest index. Four further calls produce **next-iteration index**.

Set the working second argument to four times next-iteration index. Write that value to 16 times next-iteration index plus 20. Set the working third argument and incoming-register working value to the same value.

Update the link/control working value to the fixed continuation value represented at the end of the body, then repeat.

## Side effects

### Memory

The function visibly performs byte and word stores without validating addresses, alignment, capacity, or accessibility.

Byte-indirect destinations use a loaded byte as a numeric base, not as an offset into an established object. Multiple such stores may alias each other, the byte supplying a later address, or other visible storage.

Repeated stores cannot safely be collapsed. Dependencies intervene, destinations can be reloaded, and memory may have externally observable behavior.

The global word is read once per reached iteration. Its value is captured before intervening calls and later used for a byte store. The function does not visibly write that global by name; computed-address stores and dependencies could nevertheless alias or modify it.

### Ownership and lifetime

No allocation, deallocation, ownership transfer, reference retention, or object lifetime rule is established. Numeric address construction does not imply valid or owned memory.

### Failure handling

There are no visible null checks, bounds checks, dependency-status checks, rollback steps, or error returns. A partially executed iteration may already have performed many writes before a later operation fails.

No atomicity, synchronization, or memory-ordering guarantee is established.

## Dependencies

All visible calls are indirect. The supplied dependency map identifies only F-7d9d7a4c0ff2; it provides no opaque identities for its callees.

Accordingly, individual dependency identities are **documented_only** rather than invented. Different call sites must not be assumed to identify the same dependency, and different computed targets must not be assumed to have distinct behavior.

Established interface observations include:

- Calls with zero through four explicit word arguments.
- Word results used as indices, payloads, or forwarded arguments.
- Recovered wide results split into low and high words.
- A call receiving an additional register-derived value whose origin is incomplete.
- One initial target derived from incoming control state; later targets calculated from fixed code-relative values.

Dependency memory effects, register effects, failure modes, ownership behavior, and actual calling conventions remain unknown. A dependency may modify a byte that the function subsequently reloads. Nothing establishes that dependencies preserve the global word.

## Acceptance examples

These are conditional static examples, not runtime results.

1. **Fourth-argument truncation.**  
   If the initial fourth argument is `0x123456ab` and execution reaches the four early direct byte writes, each writes `0xab`. If execution reaches the next iteration normally, its corresponding writes use zero.

2. **Parity-derived word.**  
   Under 32-bit shift semantics, an even initial index produces zero for the word stored at 16 times wide-stride index plus 20. An odd initial index produces `0x80000000`.

3. **Packed-load address.**  
   An initial packed value of `0x3fffffff` produces the word-read address 1043. A value of `0xc0000000` produces address 20. Neither case establishes that the read is valid or aligned.

4. **Byte-indirect destination.**  
   If a relevant byte load returns `0xfe`, the subsequent byte destination is address 271. The corresponding saved-base word destination, when applicable, is address 274.

5. **Reload versus saved value.**  
   If a dependency changes iteration base plus 1 between two explicit reloads, the later byte store uses the newly loaded byte. In contrast, the early saved-base word store uses the previously captured byte despite intervening calls.

6. **Additional truncation.**  
   For byte-source value `0x12345678`, shifting left by two and retaining one byte writes `0xe0`. Masking that value with `0x3fffffff`, shifting right by 20, and retaining one byte writes `0x23`.

7. **Iteration transition.**  
   If next-iteration index is 5, the final store writes 20 at address 100. The next iteration’s second and third working arguments are both 20. Its first three explicit dependency arguments are therefore 20, 20, and 20.

8. **Interrupted execution.**  
   If a dependency never returns, later stores and the iteration transition do not occur. No visible cleanup or compensating operation is required by the supplied body.

## Uncertainties

The following remain **documented_only**:

- Intended application behavior and meaningful names for memory regions.
- Valid entry conditions for the incoming register and control state.
- Actual dependency identities, signatures, and side effects.
- Provenance of the incompletely recovered register argument.
- Exact target architecture, endianness, alignment rules, and fault behavior.
- Whether low-address accesses refer to mapped memory or specialized hardware.
- Whether apparently separate memory locations alias.
- Whether the recovered wide results reflect true 64-bit returns or a multi-register convention.
- Whether discarded returns carry status that the actual machine sequence handles differently.
- Whether the apparent infinite loop and function boundary accurately represent executable control flow.

The supplied material also reports unreachable-block removal and overlapping global symbols. Those reports limit confidence in completeness and memory interpretation; they do not establish extra behavior.

## Evidence

**Established static evidence:** four explicit arguments; additional incoming machine state; an unconditional repeating body; indirect calls; byte, word, and recovered wide-value operations; computed-address reads and writes; a global-word read; and explicit end-of-iteration state replacement.

**Inferred interpretation:** 32-bit modular machine arithmetic is the basis for the conditional arithmetic examples. Packed-field manipulation is suggested by masks and shifts, but no field meaning is assigned.

**Unknown:** executable validity, intended domain, dependency contracts, memory ownership, and runtime outcomes.

No runtime tests have been performed. No execution trace, observed output, or successful memory access is claimed.
