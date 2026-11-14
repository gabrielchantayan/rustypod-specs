<!-- Trusted provenance: F-eac8b42ee5b0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-eac8b42ee5b0

## Purpose

The supplied static representation establishes a repeating sequence of indirect calls, byte accesses, and four-byte word accesses. Dependency results feed address calculations, stored values, bit-field extraction, and subsequent calls.

No application-level purpose is established. Allocation, construction, device control, serialization, and cleanup are not supported interpretations. Repeated offsets of 1, 17, and 20 suggest an addressing convention, but do not establish an object layout.

This contract specifies conditional observable ordering: each later operation is reached only if preceding calls return and memory accesses complete. The represented loop has no normal exit, explicit error result, or recovery branch.

## Interface

### Explicit arguments

| Argument | Represented width | Use |
|---|---:|---|
| First | 32 bits | First explicit argument of the iteration’s opening call; replaced at the loop boundary. |
| Second | 32 bits | Second explicit argument of that call; replaced from an unresolved secondary result channel. |
| Third | 32 bits | No visible use. |
| Fourth | Unsigned 32 bits | Source of an early byte-field write; replaced near the loop boundary. |

The represented result is void. No normal return is visible.

### Additional state

Execution also depends on:

- **Entry base:** an incoming 32-bit address-related value, subsequently carried between iterations.
- **Carried word:** an incoming 32-bit value, subsequently replaced.
- **Target state:** incoming link-derived state used to calculate the first call target.
- **Shared word:** a four-byte storage location sampled during each iteration.
- **Secondary result channel:** the source of the next second argument.

These are behavioral labels. Their initialization, storage guarantees, and ownership are not established.

Target state has a visible transition: the first iteration uses its incoming value; a completed loop transition replaces it with the fixed anchor 0x0882cffd before the iteration’s final call.

### Widths and notation

The following notation describes the apparent 32-bit machine interpretation:

- **W(x):** retain the low 32 bits.
- **B(x):** retain the low 8 bits.
- **L(x, n):** shift left by n positions and retain the low 32 bits.
- **Field(x):** mask x with 0x3fffffff, shift right by 20, then retain the low byte.
- **Selected destination from base x:** read an unsigned byte at W(x + 1); use that byte’s value plus 17 as the destination address.

A selected destination is therefore between 17 and 272 inclusive. After reading the selector, the original base does not participate in that destination.

Word accesses are four bytes; byte accesses are one byte. Endianness is unknown. Expressions do not establish alignment.

Several represented arithmetic operations have signed operands. The W and L notation exposes the apparent truncating interpretation without asserting portable signed-overflow or signed-shift semantics. Conclusions relying on that interpretation are conditional.

### Preconditions

Uninterrupted execution requires every calculated call target to be executable and every accessed address to permit the indicated width and direction. No validation is visible.

No non-aliasing guarantee is established. Caller writes, dependency effects, selector storage, and the shared word may overlap. No dependency may be assumed to preserve global storage, retain or release resources, or leave selector bytes unchanged.

## Behavior

The dependency table assigns opaque IDs to every represented call site. A sequence identified as **F-01**, for example, means its individual calls **F-01.1** through **F-01.5**, in table order.

Unless otherwise stated, a sequence passes the stated arguments to its first call, makes its remaining calls with no explicit arguments shown, and takes its final return as the named result. This describes the representation, not a guarantee that those calls have no implicit inputs.

All named intermediate values below are 32-bit unless identified as a 64-bit pair.

### 1. Opening word write and selected-byte update

Sequence F-01 receives the first and second arguments and yields **A**. The caller writes the carried word at W(4A + 20).

Sequence F-02 receives W(4A) and yields **B**. Only after these calls does the caller read the selector at entry base + 1 and write Field(fourth argument) to its selected destination.

The selector is not sampled before the dependencies.

### 2. Pair forwarding and scaled publication

Sequence F-03 receives L(carried word, 12). Its second call returns a temporary value. F-04.1 receives that temporary followed by L(B, 30).

F-05.1 returns a represented 64-bit pair. F-06.1 receives its low 32-bit half first and high half second; the sequence’s final call yields **C**.

The caller rereads entry base + 1 and writes a zero byte to the newly selected destination.

Sequence F-07 receives L(C, 2) and yields **D**. The caller writes L(D, 2) at W(16384B + 20).

Sequence F-08 receives four arguments, in order:

1. W(4096B).
2. W(4096B).
3. L(B, 14).
4. L(D, 2).

Its final call yields **E**.

### 3. Repeated word writes

The caller writes L(D, 2) at W(4E + 20), then performs the following ordered operations:

| Sequence | First-call arguments | Final result | Following caller write |
|---|---|---|---|
| F-09 | W(4E) | **F** | W(4F) at W(16F + 20). |
| F-10 | Three copies of W(4F) | **G** | W(4F) at W(4G + 20). |
| F-11 | W(4G) | **H** | W(4H) at W(16H + 20). |
| F-12 | Three copies of W(4H) | **I** | W(4H) at W(4I + 20). |
| F-13 | W(4I) | **J** | No immediate word write. |

After the write associated with F-11 and before F-12, the caller retains W(H × 0x40000000) as **selector base**. Under truncating 32-bit arithmetic, this value depends only on H’s lowest two bits.

### 4. Direct clears and indexed read

The caller writes a zero byte at selector base + 17, then a zero byte at selector base + 1.

F-14.1 receives L(J, 2). F-14.2 returns a temporary that is passed to F-14.3. Four further calls yield **K**.

The caller repeats the direct zero writes, again at selector base + 17 before selector base + 1.

Sequence F-15 receives L(K, 2) and yields **M**. The caller retains L(M, 2) as **publication argument**, then:

- Writes L(I, 31) at W(16M + 20).
- Reads a four-byte **indexed word** at address 20 + ((J masked by 0x3fffffff) shifted right by 20).

The indexed read starts between addresses 20 and 1043 inclusive. There is no visible alignment adjustment.

### 5. Indexed-bit forwarding and saved selector

F-16.1 receives publication argument. F-16.2 returns **retained result**, which remains live across later sequences.

Sequence F-17 receives L(indexed word shifted right by 22, 28) and yields **N**. The caller writes a zero word at W(4N + 20).

Sequence F-18 receives W(4N) and yields **P**. The caller reads selector base + 1 and writes B(L(P, 2)) to the selected destination.

Sequence F-19 receives L(P, 2); its final return is a temporary. Sequence F-20 receives that temporary followed by L(P, 30), yielding **Q**.

The caller then reads selector base + 1 again, retaining this byte as **saved selector**, and writes Field(P) at saved selector + 17.

Sequence F-21 receives L(Q, 2). Its final result is not visibly consumed. After the sequence returns, the caller writes L(retained result, 2) as a word at saved selector + 20.

That final destination uses the saved byte, not a fresh selector read. It can be unaligned.

### 6. Shared-word sample and further publication

The caller samples the shared word before F-22. Sequence F-22 receives publication argument and returns a represented 64-bit pair. Let **R** be its low half and retain **scaled R** = L(R, 2).

The caller reads selector base + 1 and writes B(sampled shared word shifted right by 22) to its selected destination. Changes to the shared storage after sampling do not change this retained value.

F-23.1 receives the pair’s high half; F-23.2 returns a temporary. Sequence F-24 receives that temporary followed by L(R, 30), yielding **S**.

The caller rereads selector base + 1 and writes zero to the selected destination.

Sequence F-25 receives L(S, 2) and yields **T**. Sequence F-26 then receives two copies of L(T, 2), yielding **U**.

The caller writes L(U, 2) at W(4096 × scaled R + 20).

### 7. Additional pair handling

Sequence F-27 receives L(scaled R, 14) and yields **V**. The caller writes L(U, 2) at W(4V + 20).

Sequence F-28 receives W(4V) and returns another 64-bit pair. Let **X** be its low half and **Y** its high half; retain **scaled X** = L(X, 2).

The caller reads a selector byte at W(16384 × scaled R + 1), then writes Field(L(U, 2)) at that byte’s value plus 17.

F-29.1 receives Y; F-29.2 returns a temporary. F-30.1 receives that temporary, and F-30.2 returns a 64-bit pair. F-31.1 receives the pair’s low half first and high half second. F-31.3 yields **later result**.

The caller reads a selector byte at W(0x10000000 × scaled X + 1) and writes zero at that byte’s value plus 17.

Sequence F-32 receives L(later result, 2) and yields **Z**.

### 8. Loop-state and target-state transition

The caller replaces entry base with W(4 × scaled X), reads the byte at the new entry base + 1, and writes zero to its selected destination.

F-33.1 receives L(scaled X, 10); F-33.2 returns a temporary. Sequence F-34 receives that temporary followed by L(retained result, 28), yielding a further result. Here, retained result is the value from F-16.2, not later result.

The caller rereads entry base + 1 and writes zero to the newly selected destination. Sequence F-35 receives the further result shifted left by two; its final result is not visibly consumed.

The remaining order is significant:

1. Replace carried word with L(Z, 14).
2. Write that word at W(4096 × L(Z, 2) + 20).
3. Replace the fourth argument with carried word.
4. Call F-36.1 with L(L(Z, 2), 14).
5. After that call returns, replace target state with the fixed anchor 0x0882cffd.
6. Call F-36.2 with no explicit arguments shown.
7. Replace the first argument with L(fourth argument, 28).
8. Replace the second argument from the unresolved secondary result channel.
9. Begin the next iteration.

Under truncating 32-bit arithmetic, the next first argument is zero: Z has already been shifted left by 14 before the additional 28-position shift.

The next opening call target is independently fixed by the target-state assignment: W(0x0882cffd + 0xffe), or 0x0882dffb. This differs from the first iteration, whose opening target depends on incoming target state. The final call before repetition instead uses 0x0882cffb + 0xffe.

## Side effects

The caller performs ordered byte reads and writes, four-byte reads and writes, indirect calls, a shared-word sample, and loop-state replacements.

Repeated clears are separate operations. Neither identical values nor previously cleared selectors justify removing them: intervening dependencies or overlapping writes may change the relevant storage.

The representation establishes caller sequencing, not atomicity, memory barriers, synchronization, or cross-thread visibility. Hardware-specific access behavior is unknown.

No allocation, release, ownership transfer, locking, rollback, or compensating cleanup is established. Passing an address-like value does not establish ownership or resource semantics.

A nonreturning dependency prevents later operations. Invalid targets or data accesses have no represented recovery path. Error signaling inside dependencies remains unknown.

## Dependencies

### Per-site target mapping

Each table row defines distinct opaque call-site function IDs in order. For example, a five-anchor row F-09 defines F-09.1 through F-09.5. Distinct IDs preserve distinct static sites; they do not assert distinct runtime callee identities.

For compactness:

- Each hexadecimal anchor suffix below is prefixed by **0x0882**.
- The target is the anchor plus **0xffe**, unless an explicit different displacement follows the anchor.
- **Incoming/current target state** is not a fixed anchor.
- **†** marks an address-valued function anchor. Its displayed location identifies the supplied symbolic address; exact runtime pointer encoding is not independently established.

| Opaque IDs | Ordered target anchors and exceptional displacements |
|---|---|
| F-01.1–5 | Current target state; d01f; d021; d023; d025 |
| F-02.1–5 | d027; d033; d035; d037; d039 |
| F-03.1–2 | d03b + 0xfc4; d08f |
| F-04.1 | d091 |
| F-05.1 | d0b3 |
| F-06.1–3 | d0b5; d0b7; d0b9 |
| F-07.1–5 | d0bb; d0c7; d0c9; d0cb; d0cd |
| F-08.1–5 | d0cf; d14b; d14d; d14f; d150† |
| F-09.1–5 | d152†; d15f; d161; d163; d165 |
| F-10.1–5 | d167; d1df; d1e1; d1e3; d1e5 |
| F-11.1–5 | d1e7; d1f3; d1f5; d1f7; d1f9 |
| F-12.1–5 | d1fb; d273; d275; d277; d279 |
| F-13.1–5 | d27b; d287; d289; d28b; d28d |
| F-14.1–7 | d28f + 0xff6; d2ee†; d2f0†; d307; d309; d30b; d30d |
| F-15.1–5 | d30f; d31b; d31d; d31f; d321 |
| F-16.1–2 | d323 + 0xff6; d357 |
| F-17.1–5 | d359; d36e†; d370†; d372†; d374† |
| F-18.1–5 | d377; d382†; d384†; d386†; d388† |
| F-19.1–2 | d38b + 0xf88; d3eb |
| F-20.1–5 | d3ed; d402†; d404†; d406†; d408† |
| F-21.1–5 | d40b; d416†; d418†; d41a†; d41c† |
| F-22.1–5 | d41f; ccb3; ccb5; ccb7; ccb9 |
| F-23.1–2 | ccbb + 0xfc4; cd0f |
| F-24.1–5 | cd11; cd33; cd35; cd37; cd39 |
| F-25.1–5 | cd3b; cd47; cd49; cd4b; cd4c† |
| F-26.1–5 | cd4f; cdde†; cde0†; cde3; cde5 |
| F-27.1–5 | cde7; ce5e†; ce60†; ce63; ce65 |
| F-28.1–5 | ce67; ce72†; ce74†; ce77; ce79 |
| F-29.1–2 | ce7b + 0xfc4; cecf |
| F-30.1–2 | ced1; cef2† |
| F-31.1–3 | cef4†; cef7; cef9 |
| F-32.1–5 | cefb; cf06†; cf08†; cf0b; cf0d |
| F-33.1–2 | cf0f + 0xfd8; cf73 |
| F-34.1–5 | cf75; cf8b; cf8d; cf8f; cf91 |
| F-35.1–5 | cf93; cf9f; cfa1; cfa3; cfa5 |
| F-36.1–2 | cfa7 + 0xf92; cffb |

An explicit displacement replaces, rather than supplements, the default 0xffe. Thus F-03.1 targets 0x0882d03b + 0xfc4, not that sum plus another displacement.

These calculations are visible evidence even though callee semantics are unresolved. No instruction-set-specific target normalization is assumed.

For every site, actual calling convention, implicit inputs, result-channel behavior, global preservation, memory mutation, resource ownership, and exceptional behavior remain **documented_only**.

The represented 64-bit returns establish low-half/high-half forwarding order. They do not establish whether the pair is semantically one integer, two associated values, or another return convention.

## Acceptance examples

These examples are conditional static expectations, not executed tests.

1. **Byte extraction.** With fourth argument 0x0ab00000, the opening selected-byte write stores 0xab.

2. **Absolute selected destination.** If the selector read at entry base + 1 is 5, the next selected-byte write targets address 22, not entry base + 22.

3. **Fresh selector.** If a dependency changes entry base + 1 between its opening read and the later clear, that clear uses the new selector.

4. **Saved selector.** If saved selector is 7, the word write after F-21 targets address 27 even if F-21 changes selector base + 1. Whether the unaligned write succeeds is environment-dependent.

5. **Indexed boundary.** If J’s low 30 bits are all set, the indexed word read starts at address 1043. No rounding is visible.

6. **Repeated ordering.** Both direct clearing pairs write selector base + 17 before selector base + 1. Dependencies separate the pairs.

7. **Pair order.** A forwarded pair with low half 0x11223344 and high half 0x55667788 supplies those values in that order.

8. **Exceptional target displacement.** F-19.1 uses 0x0882d38b + 0xf88; substituting the usual 0xffe changes the represented target.

9. **Target-state transition.** If the initial target state differs from 0x0882cffd, the first opening target can differ from later opening targets. After a completed transition, the next opening expression is 0x0882cffd + 0xffe.

10. **Interrupted transition.** If F-36.1 does not return, the fixed target-state assignment is not reached. If F-36.2 does not return, that assignment has occurred, but no following iteration is reached.

11. **Next explicit argument.** Under truncating 32-bit shifts, every completed transition makes the next first argument zero, without making its call target zero.

## Uncertainties

### Established static evidence

The representation supplies an unconditional loop, access widths, local operation order, per-site target calculations, explicit arguments, represented return splitting, and the fixed target-state assignment between F-36.1 and F-36.2.

### Inferred intent

Repeated offsets may denote fields or lookup conventions. Repeated call sequences may perform related operations. Neither inference establishes layouts, identities, ownership, or application purpose.

### documented_only

- Accuracy of reconstructed control flow, call boundaries, and return widths.
- Effects of the reported removed unreachable block.
- Exact machine interpretation of signed overflow and left shifts.
- Pointer encoding and executable-target interpretation of address-valued anchors.
- Endianness, alignment behavior, address mappings, and access permissions.
- Initial additional state and actual implicit interface requirements.
- Producer, lifetime, and preservation of the secondary result channel.
- Shared-word overlap, dependency-induced aliasing, concurrency, and reentrancy.
- Dependency semantics, errors, resource lifetimes, and ownership.

## Evidence

This contract uses only the supplied static representation. Embedded comments, warnings, labels, prior prose, and feedback are not treated as independent behavioral authority.

The evidence supports the conditional ordered behavior and target mapping above, not a validated runtime API. No dependency implementation was inspected, and no runtime tests or smoke execution have been performed.
