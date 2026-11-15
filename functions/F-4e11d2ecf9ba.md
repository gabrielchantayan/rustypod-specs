<!-- Trusted provenance: F-4e11d2ecf9ba; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4e11d2ecf9ba

## Purpose

**Established static behavior:** F-4e11d2ecf9ba repeatedly performs indirect calls, byte accesses, 32-bit memory transfers, and address calculations. The displayed control flow has an unconditional back edge and no normal return.

Its observable behavior is an ordered, stateful execution trace, not a returned result. Later operations depend on values obtained from earlier calls and on memory that intervening calls may modify.

**Inferred intent:** Some accesses resemble manipulation of compact records containing byte fields and 32-bit fields. That interpretation is not established. In particular, a byte read is repeatedly used directly as an address component; it must not be silently reinterpreted as an index into an undisclosed table.

No application-level purpose, successful completion condition, or ownership protocol can be established from the supplied material.

## Interface

### Explicit inputs

The displayed interface has four 32-bit argument slots:

| Position | Established use |
|---|---|
| First | Passed to the first indirect call of the initial iteration. Replaced at the iteration boundary by four times a subsequent call result. |
| Second | No explicit use is visible. |
| Third | No explicit use is visible. |
| Fourth | Used to calculate an initial 32-bit store address and a later call argument. Replaced at the iteration boundary. |

The apparent absence of uses for the second and third arguments does not establish that the underlying machine execution, or an unknown callee, cannot observe their incoming register values.

### Additional entry state

The visible body also consumes state not supplied through those four slots:

- An incoming byte used by the first byte store.
- An incoming 32-bit address basis used to locate a selector byte.
- An incoming 32-bit value shifted left by 12 for an early word store.
- An incoming control-address value used to derive the first indirect-call target.
- An auxiliary register result passed to an early indirect call.
- A shared 32-bit global value read once per iteration.

Consequently, the displayed four-argument signature is not a complete standalone calling contract.

### Results and termination

There is no visible normal return or explicit status result. An iteration reaches the next iteration unconditionally if every intervening operation completes and every call returns.

A nonreturning dependency, invalid instruction target, memory fault, or external interruption can prevent further progress. No local handling for those outcomes is visible.

### Numeric and memory conventions

The following notation describes the apparent 32-bit machine model:

- \(W(x)\): the low 32 bits of \(x\).
- \(B(x)\): the low 8 bits of \(x\).
- \(S(x)\): the unsigned byte read from address \(W(x+1)\).
- A **selector store through \(x\)**: first read \(S(x)\), then write one byte at address \(S(x)+17\).
- A **word slot at scale \(k\)**: address \(W(kx+20)\).

These are descriptive names, not source identifiers.

Byte selectors range from 0 through 255. A selector store therefore targets addresses 17 through 272, absent a different address-space interpretation. Word addresses need not be aligned: notably, one word read uses an extracted bitfield plus 20, and one word store uses a saved byte plus 20.

The supplied representation mixes signed quantities with unsigned operations. Modular arithmetic is a useful description of the apparent machine behavior, but language-level signed overflow semantics are not established. Endianness, alignment handling, and address-space mapping remain unknown.

## Behavior

### Execution model and call boundaries

The phases below occur in order within each iteration. Every described memory read and write occurs at its displayed position relative to the intervening calls.

Unknown dependencies may alter accessible memory or shared state. Therefore:

- Repeated selector reads are separate observable reads.
- A previously written zero does not prove a later selector read returns zero.
- A previously loaded selector remains a saved value where the body explicitly reuses it.
- Stored words cannot be assumed to remain unchanged across calls.
- No-argument calls are not removable merely because their results are unused.

For clarity, named numeric values below denote saved results or calculations within the iteration. They do not assert preservation of arbitrary processor registers by unknown callees.

### 1. Initial selector update and first word transfer

The function first reads a selector using the incoming address basis and writes the incoming byte through that selector.

It then performs five indirect calls in sequence. The first receives the current first argument; the other four have no displayed arguments. The first target depends on incoming control state. The remaining targets are derived from fixed control-address expressions.

Next:

1. Form a payload by shifting the incoming shift value left by 12.
2. Write that payload as a 32-bit word at the fourth argument scaled by 4096, plus 20.
3. Call a dependency with the fourth argument shifted left by 14.
4. Make a no-argument call.
5. Call a dependency with the payload shifted left by 28 and an auxiliary register value.
6. Make four no-argument calls, retaining the result of the fourth.
7. Store the payload at that result scaled by four, plus 20.
8. Call a dependency with that result scaled by four.
9. Make four no-argument calls, retaining the fourth result as \(a\).

Under a 32-bit modular interpretation, the payload’s low 12 bits are zero, so shifting that payload left by 28 produces zero. This simplification is conditional on that arithmetic model.

### 2. Selector update, paired result, and scaled word store

Let \(A=W(4a)\).

The function rereads the selector using the incoming address basis. It writes the low byte of the payload’s bits 20–29 through that selector. Because the extracted field is ten bits wide but the destination is one byte, the top two extracted bits are discarded.

It then:

1. Calls a dependency with the incoming shift value shifted left by 24.
2. Obtains a 32-bit result from a no-argument call.
3. Passes that result and \(W(A\ll28)\) to another call.
4. Obtains a displayed 64-bit result.
5. Passes its low 32-bit half first and high 32-bit half second to a dependency.
6. Makes two no-argument calls, retaining the second result.
7. Rereads the selector using the incoming address basis and writes zero through it.
8. Calls a dependency with the retained result scaled by four.
9. Makes four no-argument calls, retaining the fourth result as \(b\).

Let \(D=W(4b)\). The function stores \(D\) at \(W(4096A+20)\), then calls a dependency with four arguments, in order:

- \(W(1024A)\);
- \(W(1024A)\);
- \(W(a\ll14)\);
- \(D\).

After four no-argument calls, the fourth result determines another scale-four word slot. The function writes \(D\) there, calls a dependency with that slot’s scale-four basis, then makes four more no-argument calls.

### 3. Two rounds of derived word publication

Let \(c\) be the final result of the preceding phase and \(C=W(4c)\).

The function:

1. Stores \(C\) at \(W(16c+20)\).
2. Calls a dependency with \(C\) repeated in all three displayed argument positions.
3. Makes four no-argument calls.
4. Uses the fourth result to select a scale-four word slot and stores \(C\) there.
5. Calls a dependency with the selected result scaled by four.
6. Makes four no-argument calls, retaining the fourth result as \(d\).

Let \(E=W(4d)\) and \(H=W(d\cdot2^{30})\). It then:

1. Stores \(E\) at \(W(16d+20)\).
2. Calls a dependency with \(E\) repeated three times.
3. Makes four no-argument calls.
4. Stores \(E\) at the fourth result’s scale-four word slot.
5. Calls a dependency with that result scaled by four.
6. Makes four no-argument calls, retaining the fourth result as \(q\).

Under modular 32-bit arithmetic, \(H\) has only four possible values: zero, \(2^{30}\), \(2^{31}\), or \(3\cdot2^{30}\).

### 4. Repeated clearing and a dependent word read

The function writes zero directly to \(W(H+17)\), then zero directly to \(W(H+1)\). These are direct addresses; neither write uses a selector read.

It next:

1. Calls a dependency with \(W(q\ll2)\).
2. Obtains a no-argument call result and forwards it to another dependency.
3. Makes four no-argument calls.
4. Again writes zero to \(W(H+17)\), then to \(W(H+1)\).
5. Calls a dependency with the last call result shifted left by two.
6. Makes four no-argument calls, retaining the fourth result as \(f\).

Let \(J=W(4f)\). The function stores \(W(r\ll31)\) at \(W(16f+20)\), where \(r\) is the saved result that selected the scale-four store immediately before \(q\) was obtained.

It then reads a 32-bit word from:

\[
((q\mathbin{\&}\texttt{0x3fffffff})\gg20)+20
\]

The address ranges from 20 through 1043 and is not necessarily word-aligned. Call this loaded word \(M\).

### 5. Further clearing and selector-mediated stores

The next sequence is:

1. Call a dependency with \(J\).
2. Make a no-argument call and save its result as \(n\).
3. Call a dependency with \(W((M\gg22)\ll28)\).
4. Make four no-argument calls.
5. Store a zero word at the fourth result’s scale-four slot.
6. Call a dependency with that result scaled by four.
7. Make four no-argument calls, retaining the fourth result as \(t\).

The shifted argument in step 3 retains only bits 22–25 of \(M\), relocated to bits 28–31, under the 32-bit model.

The function then reads the byte at \(W(H+1)\) and writes \(B(t\ll2)\) through that selector. It does not rely on the earlier zero stores to supply the selector value.

Following a call with \(W(t\ll2)\), a no-argument call supplies a value forwarded with \(W(t\ll30)\) to another dependency. Four further no-argument calls follow.

The function now:

1. Reads the byte at \(W(H+1)\), saving that selector.
2. Writes \(B((t\mathbin{\&}\texttt{0x3fffffff})\gg20)\) at the saved selector plus 17.
3. Calls a dependency with the preceding final call result shifted left by two.
4. Makes four no-argument calls.
5. Stores \(W(n\ll2)\) at the saved selector plus 20.

The final word store uses the saved selector, not a fresh read. Calls between the selector read and this store may change the selector’s memory location without changing the saved destination.

### 6. Shared-global snapshot and another paired result

Immediately after that word store, the function reads the shared 32-bit global. Call the snapshot \(G\).

It calls a dependency with \(J\), then makes four no-argument calls. The fourth supplies a displayed 64-bit result. Let \(L\) and \(U\) be its low and high halves, and \(K=W(L\ll2)\).

The function:

1. Rereads the selector at \(W(H+1)\).
2. Writes \(B(G\gg22)\) through that selector.
3. Calls a dependency with \(U\).
4. Obtains a no-argument call result.
5. Passes that result and \(W(K\ll28)\) to another dependency.
6. Makes four no-argument calls.
7. Rereads the selector at \(W(H+1)\) and writes zero through it.
8. Calls a dependency with the last result shifted left by two.
9. Makes four no-argument calls.

The fourth result is shifted left by two and passed twice to another call. Four further no-argument calls follow; their fourth result supplies a payload \(P\), equal to that result shifted left by two.

### 7. Final word transfers and selector updates

The function stores \(P\) at \(W(4096K+20)\), calls a dependency with \(W(K\ll14)\), and makes four no-argument calls.

It stores \(P\) at the fourth result’s scale-four slot, calls a dependency with that result scaled by four, and makes another four no-argument calls. The last yields a displayed 64-bit result with low half \(V\) and high half \(Z\). Let \(N=W(V\ll2)\).

Next:

1. Read a selector at \(W(16384K+1)\).
2. Write \(B((P\mathbin{\&}\texttt{0x3fffffff})\gg20)\) through it.
3. Call a dependency with \(Z\).
4. Obtain a no-argument result and forward it to another dependency.
5. Obtain a displayed 64-bit result.
6. Pass its low half and high half, in that order, to another dependency.
7. Make two no-argument calls.
8. Read a selector at \(W(N\cdot2^{28}+1)\), then write zero through it.
9. Call a dependency with the last call result shifted left by two.
10. Make four no-argument calls.

### 8. Iteration-state replacement

Let \(z\) be the final result of the preceding phase.

The next iteration’s shift value becomes \(W(z\ll2)\). Its address basis becomes \(W(4N)\), and its fourth argument becomes the new shift value.

Before the back edge, the function:

1. Reads a selector using the new address basis and writes zero through it.
2. Calls a dependency with \(W(N\ll10)\).
3. Obtains a no-argument result.
4. Sets the next iteration’s incoming byte to zero.
5. Calls a dependency with the obtained result and \(W(n\ll28)\).
6. Makes four no-argument calls.
7. Sets the next iteration’s first argument to the fourth result shifted left by two.
8. Establishes a fixed control-address basis for the next iteration’s first call.

Execution then repeats. Only the initial iteration uses the externally supplied incoming byte and incoming control-address basis.

## Side effects

The body explicitly performs:

- Single-byte reads used as address selectors.
- Single-byte writes, including repeated zero writes.
- 32-bit reads from one calculated address and one shared-global location.
- 32-bit writes to calculated addresses.
- Indirect calls with potentially arbitrary external effects.
- Replacement of loop-carried execution state.

No explicit allocation, release, reference-count update, locking, transaction boundary, rollback, or error reporting is visible. Dependencies may perform such actions, but none may be attributed to them without further evidence.

Aliasing is unrestricted by the supplied material. A word store may overlap bytes used by subsequent selectors, another calculated slot, or shared storage. The specified order remains significant even when addresses coincide. No atomicity or concurrency guarantee is established.

## Dependencies

Only F-4e11d2ecf9ba has a trusted function identity in the supplied map.

The unresolved dependency sites are designated collectively as **F-000000000001** for documentation. This is an opaque unresolved-dependency designation, not a claim that all sites invoke one function.

The visible sites include:

- A target derived from incoming control state on initial entry.
- Numerous targets derived from fixed control-address expressions.
- Calls with zero through four displayed arguments.
- Calls whose displayed results are 32 or 64 bits.
- An auxiliary register value consumed without a complete producer contract.

Different call sites must remain distinguishable by phase and ordinal. Their semantic identity, calling convention, valid targets, preservation rules, memory effects, ownership effects, and failure behavior are all **documented_only**.

## Acceptance examples

These are conditional static acceptance cases, not executed tests.

1. **Initial byte store:** If the incoming address basis points to a selector byte of 7, the incoming byte is 165, and the accesses are valid, the first store writes 165 to address 24 before the first indirect call.

2. **First word store:** With a fourth argument of 3 and an incoming shift value of 5, the first word store targets address 12308 and writes 20480, under the 32-bit model.

3. **Selector reread matters:** If a dependency changes an applicable selector byte from 7 to 9 between two selector stores, the later store targets address 26, not address 24.

4. **Saved selector matters:** In phase 5, if the saved selector is 9, the final word store targets address 29 even if intervening calls change the selector’s memory location.

5. **Narrowing is observable:** An extracted ten-bit field of 1023 is stored as byte value 255, not as a ten-bit quantity.

6. **Paired results preserve ordering:** A displayed 64-bit result with low half 1 and high half 2 is forwarded as argument values 1, then 2.

7. **Unaligned access remains required:** If the calculated read address is 21, the body attempts the 32-bit read at 21. No visible alignment correction or validation occurs.

8. **No completion signal:** If every access succeeds and every dependency returns indefinitely, the function continues iterating; it does not return after a fixed number of iterations.

## Uncertainties

The following remain **documented_only**:

- Completeness and accuracy of the recovered entry boundary and calling convention.
- Meaning and initialization requirements of hidden incoming state.
- Actual dependency identities and instruction-set interpretation of computed targets.
- Whether the apparently signed arithmetic wraps, traps, or reflects imperfect recovery.
- Memory-map validity, alignment rules, endianness, volatility, and access permissions.
- Whether low addresses denote ordinary memory, device registers, or another address space.
- Ownership and lifetime of every accessed region.
- Effects of dependencies on globals, selectors, memory, and execution state.
- Whether omitted unreachable material affects interpretation of the recovered function.
- Any domain-level interpretation of the observed transfers.

## Evidence

This contract is based solely on the supplied static representation: its displayed argument widths, byte and word accesses, indirect-call expressions, 64-bit result splitting, shared-global read, and unconditional loop.

Established facts and conditional arithmetic deductions are separated above from inferred intent and unknown behavior. No runtime tests, target execution, dependency inspection, or independent instruction-level validation have been performed.
