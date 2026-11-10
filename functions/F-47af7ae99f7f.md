<!-- Trusted provenance: F-47af7ae99f7f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-47af7ae99f7f

## Purpose

The available static evidence describes an ordered sequence of indirect dependency calls, coprocessor-transfer operations, and a final 16-bit memory write. The sequence ends at an instruction boundary that the supplied representation cannot interpret. No normal return is established.

No application-level purpose is established. In particular, the evidence does not justify describing this function as initialization, decoding, device configuration, or memory allocation.

This contract distinguishes:

- **Established static behavior:** operations and data dependencies explicitly present in the supplied representation.
- **Inferred intent:** possible interpretations that are not contractual requirements.
- **Unknown behavior:** information unavailable without dependency implementations, architecture details, or valid instruction coverage.

All behavior below is conditional on execution reaching the relevant operation. An earlier dependency call, coprocessor operation, or memory access may prevent later operations from executing.

## Interface

### Explicit inputs

The represented interface has three explicit inputs:

| Input | Width | Established use |
|---|---:|---|
| First input | 32 bits | No explicit use in the visible body. |
| Second input | 32 bits | No explicit use in the visible body. |
| Third input, called **P** below | 32 bits, unsigned | Passed unchanged to the first dependency; also used in shifted and masked arguments. |

Lack of an explicit use does not prove that the first two inputs are irrelevant to the actual machine interface. The recovered signatures of the indirect calls are not independently verified.

### Implicit inputs

The visible computation also depends on:

- An entry register value, treated here as a 32-bit word **E**, whose low 11 bits contribute to the first dependency’s first argument.
- An entry link-register value used to construct the first indirect call target.
- Five captured coprocessor-related operands, designated **C1**, **C5**, **C9**, **C10**, and **C15**.
- Results returned by several dependencies.
- Any ambient state that the dependencies or coprocessor operations access.

The entry link-register value plus a fixed displacement determines the first call target. The remaining visible call targets are constructed from fixed code-location anchors and fixed displacements. This contract identifies those calls opaquely rather than reproducing private addresses.

### Output and completion

The represented function has no explicit return value.

The visible body does not establish normal completion. After the final memory write, the representation terminates at an unrecognized-instruction boundary. That boundary is not evidence of a deliberate application-level halt, a defined exception, or a return.

### Arithmetic notation

For the word computations below:

- **Wₖ(x)** means a left shift by *k* positions, retaining only the low 32 bits.
- **low(x)** and **high(x)** mean the low and high 32-bit halves of a represented 64-bit result.
- Right shifts of represented unsigned 64-bit results are logical shifts.
- Values passed through uninterpreted 32-bit results remain opaque words unless arithmetic is explicitly applied.

These are machine-word descriptions, not a claim that the supplied signed expressions define portable source-language arithmetic. Exact instruction semantics remain architecture-dependent.

## Behavior

### 1. Initial dependency sequence

The function first invokes **F-U01** with four represented arguments, in this order:

1. W₂₁(E).
2. Zero.
3. P.
4. Zero.

Under 32-bit word arithmetic, W₂₁(E) depends only on the low 11 bits of E.

It then invokes the following dependencies in order, each with no arguments explicitly represented:

1. **F-U02**.
2. **F-U03**.
3. **F-U04**.
4. **F-U05**.
5. **F-U06**.
6. **F-U07**.

The 32-bit result of F-U07 supplies the payload of the first coprocessor transfer.

“No arguments explicitly represented” does not establish that a dependency has no machine-level inputs. It only describes the visible call expression.

### 2. First coprocessor transfers and P-derived arguments

Immediately after F-U07 returns, the function performs a coprocessor-transfer operation with:

- Fixed leading fields: **0, 1, 7**.
- Payload: the 32-bit result of F-U07.
- Remaining captured operands: C1 and C15.

It then invokes **F-U08**, **F-U09**, and **F-U10**, in that order, with no explicit arguments.

Next it performs a second coprocessor transfer with:

- Fixed leading fields: **0, 5, 7**.
- Payload: W₂₄(P).
- Remaining captured operands: C5 and C15.

W₂₄(P) places P’s low byte in bits 31–24 and clears bits 23–0.

The function then invokes:

1. **F-U11**, with W₂₄(P) and zero.
2. **F-U12**, with no explicit arguments.
3. **F-U13**, with no explicit arguments, producing an opaque 32-bit result.

That result supplies the third coprocessor transfer:

- Fixed leading fields: **0, 6, 7**.
- Payload: the result of F-U13.
- Remaining captured operands: C9 and C15.

These field descriptions preserve the represented operation. They do not establish the hardware meaning of the selectors or captured operands.

### 3. Zero-input call and derived word

The function invokes **F-U14** with no explicit arguments.

It then invokes **F-U15** with three zero arguments. The first two zeros arise from shifting a value initialized to zero; no visible assignment changes that value before this call.

Let **R** be F-U15’s represented 32-bit result. The subsequent calculations use:

- **A = W₁(R)**.
- **B = W₄(R)**.
- **M = W₄(P masked to its low 23 bits)**.

The fourth coprocessor transfer has:

- Fixed leading fields: **0, 5, 7**.
- Payload: A.
- Remaining captured operands: C10 and C15.

The function then invokes **F-U16** with four arguments:

1. A.
2. M.
3. B.
4. Zero.

M occupies bits 26–4. Its low four bits and high five bits are zero. P’s bits 31–23 do not contribute to M.

The function next invokes **F-U17** and **F-U18**, with no explicit arguments.

### 4. Zero transfers and shifted arguments

After F-U18, the function performs two coprocessor transfers in order:

| Transfer | Fixed leading fields | Payload | Remaining operands |
|---|---|---:|---|
| Fifth | 0, 7, 7 | Zero | C9, C15 |
| Sixth | 0, 0, 7 | Zero | C15, C15 |

Between these transfers, the visible arithmetic also derives a value from B’s low two bits, shifted left by 16. Under the 32-bit word model, this value is necessarily zero: B was formed by shifting R left by four, so B’s low four bits are zero.

The function invokes **F-U19** with one zero argument. Let its represented 32-bit result be **S**.

It then performs the seventh coprocessor transfer:

- Fixed leading fields: **0, 2, 7**.
- Payload: W₈(B).
- Remaining captured operands: C9 and C15.

Next, **F-U20** receives:

1. W₈(B), equivalently W₁₂(R).
2. W₁₂(B), equivalently W₁₆(R).
3. W₉(S).

No eighth coprocessor transfer is visible.

### 5. First 64-bit result and chained word results

The function invokes **F-U21** with no explicit arguments. Its result is represented as 64 bits; call it **T**.

**F-U22** then receives:

1. low(T).
2. high(T).
3. W₁(low(T)).

This specifies the use of the two halves, not an established memory layout or calling convention for a 64-bit result.

The function next invokes **F-U23**. Its sole argument is derived by shifting the previously established zero value left by eight, so the represented argument is zero.

The following calls form a direct result chain:

1. **F-U23** returns an opaque 32-bit word.
2. **F-U24** receives that word and returns another opaque 32-bit word.
3. **F-U25** receives F-U24’s result.

The visible body does not dereference either chained result. They may represent numbers, handles, pointers, or other values; no particular interpretation is established.

The function then invokes **F-U26** with no explicit arguments.

### 6. Second 64-bit chain and final dependencies

The function invokes **F-U27** with no explicit arguments. Let its represented 64-bit result be **V**.

**F-U28** receives:

1. low(V).
2. high(V).
3. W₁₁(high(V)).

Let F-U28’s represented 64-bit result be **Z**.

Two uses of Z are visible:

- low(Z) is passed to **F-U29**.
- Z shifted right logically by 41 is retained for the eventual store-address calculation.

The function next invokes **F-U30** with no explicit arguments, obtaining an opaque 32-bit result.

**F-U31** receives:

1. The result of F-U30.
2. Zero.

The second argument is derived from the earlier zero value by another left shift; it is not derived from Z.

Finally, **F-U32** is invoked with no explicit arguments.

### 7. Final memory write

After F-U32 returns, the function computes a destination address from Z.

Define **Q** as the unsigned integer represented by Z’s top 12 bits. Equivalently, Q is Z shifted right logically by 52. Its range is 0 through 4095.

The destination’s numeric address is:

**Q + floor(Q / 4)**.

The function writes the 16-bit value **0x00CD** to that address.

The represented calculation first extracts a 23-bit value from Z and then shifts it further. Although one intermediate shift is expressed as signed, that intermediate lies between zero and 2²³ − 1, so it is nonnegative. The signedness does not change this address calculation.

Consequences:

- The computed address ranges from **0** through **5118**, inclusive.
- Not every integer in that interval is reachable.
- Address zero is reachable.
- Odd destination addresses are reachable.
- No base pointer, allocation result, or explicit input pointer is added.
- No bounds, mapping, null-address, or alignment check is visible.
- The store width is 16 bits, not 8 bits.
- Byte order is not established.

The supplied representation stops at an unrecognized instruction immediately after this store.

## Side effects

### Established direct effects

The visible sequence contains:

- Thirty-two indirect dependency calls.
- Seven coprocessor-transfer operations.
- One final 16-bit memory write.

The visible caller contains no ordinary data-memory read and no other ordinary data-memory write. Dependencies, instruction fetching, stack activity, and coprocessor behavior are outside that narrow observation.

The ordering above is the represented execution order. No concurrency, memory-barrier semantics, or externally observable completion guarantees are established.

### Ownership and lifetime

There is no established allocation, retention, release, or ownership-transfer contract.

Opaque dependency results are forwarded, split into words, shifted, or used to derive an address. Such use does not prove that they refer to allocated memory.

Nothing in the supplied material establishes that any dependency:

- Preserves globals, registers, memory, or coprocessor state.
- Allocates or releases storage.
- Transfers ownership.
- Returns normally.
- Avoids callbacks or re-entry.

No ownership behavior should be inferred from a result’s width or position in the call sequence.

### Errors and exceptional execution

No explicit status check, error branch, recovery path, or rollback operation is visible.

Successful progress requires each preceding operation to complete sufficiently for execution to continue. Potential failures include invalid indirect targets, dependency failures, unsupported or privileged coprocessor operations, and an invalid or misaligned final store.

The evidence does not establish which failures trap, terminate, return an error, or have some other effect. The final unrecognized-instruction boundary adds a separate uncertainty about post-store execution.

## Dependencies

The supplied dependency map provides no resolved identities for the called targets. **F-U01** through **F-U32** are opaque identifiers for call occurrences in execution order.

They are not assertions that there are 32 distinct implementations. Target aliasing and overlapping instruction interpretations remain unknown.

| Opaque IDs | Visible result use |
|---|---|
| F-U01–F-U06 | No result explicitly consumed. |
| F-U07 | 32-bit payload of the first coprocessor transfer. |
| F-U08–F-U12 | No result explicitly consumed. |
| F-U13 | 32-bit payload of the third coprocessor transfer. |
| F-U14 | No result explicitly consumed. |
| F-U15 | 32-bit word used in shifts and later arguments. |
| F-U16–F-U18 | No result explicitly consumed. |
| F-U19 | 32-bit result shifted for F-U20. |
| F-U20 | No result explicitly consumed. |
| F-U21 | Represented 64-bit result split for F-U22. |
| F-U22 | No result explicitly consumed. |
| F-U23, F-U24 | Chained opaque 32-bit results. |
| F-U25, F-U26 | No result explicitly consumed. |
| F-U27 | Represented 64-bit result split for F-U28. |
| F-U28 | Represented 64-bit result used by F-U29 and the final address calculation. |
| F-U29 | No result explicitly consumed. |
| F-U30 | Opaque 32-bit result passed to F-U31. |
| F-U31, F-U32 | No result explicitly consumed. |

“No result explicitly consumed” does not mean the call is side-effect-free or that its return value is absent at the machine level.

## Acceptance examples

These are conditional static expectations, not executed tests.

### P-derived fields

If P is **0x12345678**, then:

- F-U01’s third argument is **0x12345678**.
- The second coprocessor payload and F-U11’s first argument are **0x78000000**.
- F-U16’s second argument is **0x03456780**.

If only P’s bits 31–23 change, F-U16’s second argument remains unchanged. F-U01 still receives the changed full word, so downstream behavior is not necessarily unchanged.

### Shifted dependency result

If F-U15 returns **0x10000001**, under the stated word model:

- A is **0x20000002**.
- B is **0x00000010**.
- F-U20’s first two arguments are **0x00001000** and **0x00010000**.
- F-U23’s argument and F-U31’s second argument remain zero.

### 64-bit splitting

If F-U21 produces **0x1122334480000001**, F-U22 receives:

1. **0x80000001**.
2. **0x11223344**.
3. **0x00000002**.

This example checks truncation of the shifted low word without assuming byte order.

### Final destination

Conditional on reaching the store:

| Top 12 bits of Z | Numeric destination address | Stored value |
|---:|---:|---:|
| 0 | 0 | 16-bit 0x00CD |
| 1 | 1 | 16-bit 0x00CD |
| 4 | 5 | 16-bit 0x00CD |
| 4095 | 5118 | 16-bit 0x00CD |

A destination of 1 or 5 exposes the unresolved alignment requirement. These examples do not assert that those accesses succeed.

For equal top 12 bits of Z, changes to its lower 52 bits do not change the final destination. They can still change the argument to F-U29 and therefore dependency side effects.

## Uncertainties

The following are **documented_only**:

- **Instruction validity:** the representation reports invalid instruction data and truncated control flow. The actual executed path is not independently established.
- **ABI and signatures:** explicit parameter counts, implicit inputs, result widths, and 64-bit result conventions are not verified against an ABI.
- **Dependency behavior:** implementations, target aliasing, side effects, failure modes, global preservation, and ownership behavior are unknown.
- **Coprocessor semantics:** privilege requirements, selector meanings, operand roles, and hardware effects are unknown.
- **Machine arithmetic:** the word-based interpretation is appropriate to the visible widths but is not a verified reconstruction of every underlying instruction.
- **Memory model:** address-space mapping, address zero, alignment behavior, byte order, and the final destination’s validity are unknown.
- **Completion:** behavior beyond the final represented store is unavailable.

An interpretation involving hardware configuration is possible because coprocessor operations appear, but that intent is not established and is not part of this contract.

## Evidence

This contract is based solely on the supplied static representation of F-47af7ae99f7f.

Direct evidence supports the listed operation order, explicit arguments, represented result widths, coprocessor-transfer fields, and final 16-bit store. The zero-valued arguments and top-12-bit destination formula are arithmetic deductions from those operations.

No dependency implementations, instruction bytes, runtime traces, architecture reference, or independently verified interface declarations were supplied. No runtime tests have been performed.
