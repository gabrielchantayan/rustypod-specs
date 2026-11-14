<!-- Trusted provenance: F-ece3a128eb1b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ece3a128eb1b

## Purpose

F-ece3a128eb1b repeatedly invokes indirect dependencies and performs interleaved byte and 32-bit memory accesses. The supplied representation contains an unconditional iteration cycle and no normal return.

No application-level purpose is established. Computed addresses are not established as objects, allocated records, queues, or ordinary application storage. They may be small, unaligned, overlapping, or invalid.

This contract describes represented behavior, conditional on preceding calls returning and accesses completing. It does not guarantee completion of an iteration.

## Interface

### Explicit inputs

| Input | Width | Established use |
|---|---:|---|
| First input | 32 bits | Opening-call argument and initial word-store payload; replaced at iteration renewal. |
| Second input | 32 bits | Opening-call argument; replaced at iteration renewal. |
| Third input | 32 bits | Opening-call argument; replaced at iteration renewal. |
| Fourth input | 8 bits | Published through two ordered pairs of byte stores; replaced with zero at iteration renewal. |

The first three inputs have signed 32-bit types in the supplied representation. Their relevant uses are forwarding, arithmetic, and bit-pattern storage; no signed comparison controls execution.

Additional execution inputs include incoming link state, dependency results, a 32-bit global value, bytes read from computed addresses, and an additional register value forwarded at F-115 whose producer is unresolved.

### Outputs, preconditions, and failures

There is no represented return value, completion status, recovery branch, or rollback. Outputs consist of dependency invocations, memory accesses, and state changes.

Execution requires usable indirect targets and memory supporting each represented access width. No target validation, address-range check, or alignment adjustment is shown. If a dependency does not return or an access fails, subsequent events are not guaranteed. Earlier effects are not undone.

### Width conventions

Unless stated otherwise, expressions below describe 32-bit bit patterns. Addition, multiplication, and left shifts use a modulo-\(2^{32}\) interpretation; byte stores retain the low eight bits. This is a fixed-width model of the represented dataflow, not a claim that all signed-looking source expressions have portable language semantics.

Define:

\[
E(x)=((x\mathbin{\&}0x3FFFFFFF)\gg20).
\]

This selects original bits 20–29. Publishing its low byte retains original bits 20–27. Right shifts of explicitly unsigned values are logical.

A wide result contains 64 represented bits, split into low and high 32-bit halves. Width alone establishes neither semantic type nor ownership.

## Behavior

### Ordering conventions

Each iteration contains **146 represented dependency invocations**. Opaque IDs F-001 through F-146 identify invocation sites, not necessarily distinct implementations.

In the tables:

- “None shown” means no represented explicit arguments; implicit machine-state inputs remain unresolved.
- A range denotes separate calls in ascending order.
- \(\rho_n\) denotes the result of F-\(n\), where consumed.
- Memory events follow the row’s call or call range and precede the next row.
- Every specified byte read is a separate read unless its retained value is explicitly reused.

Dependencies may change memory or globals between events. Repeated arguments, addresses, or payloads do not justify combining or reordering calls.

### Opening publication and extraction

Let \(A=\rho_5\), \(Q=\rho_{10}\), \(S=\rho_{15}\), and \(X=\rho_{20}\). Retain the quarter base \(B=Q\times2^{30}\).

| Invocation | Explicit arguments or retained result | Following memory events |
|---|---|---|
| F-001 | Current first, second, third inputs | — |
| F-002–F-004 | None shown | — |
| F-005 | None shown; retain \(A\) | Store the first input as a word at \(20+4A\). |
| F-006 | \(4A\) | — |
| F-007–F-009 | None shown | — |
| F-010 | None shown; retain \(Q\) | Store \(4Q\) as a word at \(20+16Q\); retain \(B\). |
| F-011 | \(4Q,\ 4Q,\ 4Q\) | — |
| F-012–F-014 | None shown | — |
| F-015 | None shown; retain \(S\) | Store \(4Q\) as a word at \(20+4S\). |
| F-016 | \(4S\) | — |
| F-017–F-019 | None shown | — |
| F-020 | None shown; retain \(X\) | Store the fourth input as a byte at \(B+17\), then at \(B+1\). |
| F-021 | \(X\ll2\) | — |
| F-022 | None shown; retain result | — |
| F-023 | \(\rho_{22}\) | — |
| F-024–F-026 | None shown | — |
| F-027 | None shown; retain result | Again store the fourth input at \(B+17\), then at \(B+1\). |
| F-028 | \(\rho_{27}\ll2\) | — |
| F-029–F-031 | None shown | — |
| F-032 | None shown; retain selector \(D\) | Store \(S\ll31\) as a word at \(20+16D\); then read word \(W\) at \(20+E(X)\). |
| F-033 | \(4D\) | — |
| F-034 | None shown; retain selector \(H\) | — |
| F-035 | \((W\gg22)\ll28\) | — |
| F-036–F-038 | None shown | — |
| F-039 | None shown; retain result | Clear the word at \(20+4\rho_{39}\). |
| F-040 | \(4\rho_{39}\) | — |

The extraction address is a direct byte address, not a word index multiplied by four. It ranges from 20 through 1043. The F-035 argument retains only original bits 22–25 of \(W\). The payload \(S\ll31\) depends only on the low bit of \(S\).

### Byte-indirected publication and global capture

Let \(U=\rho_{44}\). Retain the byte read after F-051 as \(b\).

| Invocation | Explicit arguments or retained result | Following memory events |
|---|---|---|
| F-041–F-043 | None shown | — |
| F-044 | None shown; retain \(U\) | Read a byte at \(B+1\); store the low byte of \(U\ll2\) at 17 plus that byte. |
| F-045 | \(U\ll2\) | — |
| F-046 | None shown; retain result | — |
| F-047 | \(\rho_{46},\ U\ll30\) | — |
| F-048–F-050 | None shown | — |
| F-051 | None shown; retain result | Read \(b\) at \(B+1\); store the low byte of \(E(U)\) at \(17+b\). |
| F-052 | \(\rho_{51}\ll2\) | — |
| F-053–F-056 | None shown | Store \(H\ll2\) as a word at \(20+b\); then read the global into 32-bit snapshot \(G\). |
| F-057 | \(4D\) | — |
| F-058–F-060 | None shown | — |
| F-061 | None shown; retain wide result | Retain \(M\), its low half shifted left by two. Read a fresh byte at \(B+1\); store the low byte of \(G\gg22\) at 17 plus that byte. |
| F-062 | High half of F-061’s result | — |
| F-063 | None shown; retain result | — |
| F-064 | \(\rho_{63},\ M\ll28\) | — |
| F-065–F-067 | None shown | — |
| F-068 | None shown; retain result | Read a fresh byte at \(B+1\); clear the byte at 17 plus that byte. |
| F-069 | \(\rho_{68}\ll2\) | — |
| F-070–F-072 | None shown | — |
| F-073 | None shown; retain result | — |
| F-074 | \(\rho_{73}\ll2,\ \rho_{73}\ll2\) | — |

There is one explicit global read per iteration. Later dependency changes to the global do not replace the retained snapshot \(G\).

The retained byte \(b\) selects both the extraction-byte destination and the later word destination, despite intervening calls. Other reads of \(B+1\) are fresh and cannot be replaced with \(b\).

### Middle publication and changing byte sources

Define payload \(P=\rho_{78}\ll2\). Let \(L\) be the low half of F-088’s wide result shifted left by two.

| Invocation | Explicit arguments or retained result | Following memory events |
|---|---|---|
| F-075–F-077 | None shown | — |
| F-078 | None shown; retain result | Store \(P\) as a word at \(20+4096M\). |
| F-079 | \(M\ll14\) | — |
| F-080–F-082 | None shown | — |
| F-083 | None shown; retain result | Store \(P\) as a word at \(20+4\rho_{83}\). |
| F-084 | \(4\rho_{83}\) | — |
| F-085–F-087 | None shown | — |
| F-088 | None shown; retain wide result and \(L\) | Read a byte at \(1+16384M\); store the low byte of \(E(P)\) at 17 plus that byte. |
| F-089 | High half of F-088’s result | — |
| F-090 | None shown; retain result | — |
| F-091 | \(\rho_{90}\) | — |
| F-092 | None shown; retain wide result | — |
| F-093 | Low half, then high half, of F-092’s result | — |
| F-094 | None shown | — |
| F-095 | None shown; retain result | Read a byte at \(1+L\times2^{28}\); clear the byte at 17 plus that byte. |
| F-096 | \(\rho_{95}\ll2\) | — |
| F-097–F-099 | None shown | — |
| F-100 | None shown; retain selector \(T\) | Retain base \(C=4L\). Read a byte at \(C+1\); clear the byte at 17 plus that byte. |
| F-101 | \(L\ll10\) | — |
| F-102 | None shown; retain result | — |
| F-103 | \(\rho_{102},\ H\ll28\) | — |
| F-104–F-106 | None shown | — |
| F-107 | None shown; retain result | Read a fresh byte at \(C+1\); clear the byte at 17 plus that byte. |
| F-108 | \(\rho_{107}\ll2\) | — |

The source-address calculations \(B+1\), \(1+16384M\), \(1+L\times2^{28}\), and \(C+1\) are distinct. Their equality or separation in a particular execution is not guaranteed.

### Late publication and iteration renewal

Define \(V=T\ll14\), final selector \(J=\rho_{124}\), scaled selector \(K=J\ll2\), and final payload \(Y=\rho_{136}\ll2\).

| Invocation | Explicit arguments or retained result | Following memory or state events |
|---|---|---|
| F-109–F-112 | None shown | Store \(V\) as a word at \(20+4096(T\ll2)\). |
| F-113 | \((T\ll2)\ll14\), equivalently \(T\ll16\) | — |
| F-114 | None shown | — |
| F-115 | \(V\ll28\), then an additional register value of unresolved provenance | — |
| F-116–F-118 | None shown | — |
| F-119 | None shown; retain result | Store \(V\) as a word at \(20+4\rho_{119}\). |
| F-120 | \(4\rho_{119}\) | — |
| F-121–F-123 | None shown | — |
| F-124 | None shown; retain \(J\) | Read a fresh byte at \(C+1\); store the low byte of \(E(V)\) at 17 plus that byte. |
| F-125 | \(T\ll26\) | — |
| F-126 | None shown; retain result | — |
| F-127 | \(\rho_{126},\ K\ll28\) | — |
| F-128 | None shown; retain wide result | — |
| F-129 | Low half, then high half, of F-128’s result | — |
| F-130 | None shown | — |
| F-131 | None shown; retain result | Read a fresh byte at \(C+1\); clear the byte at 17 plus that byte. |
| F-132 | \(\rho_{131}\ll2\) | — |
| F-133–F-135 | None shown | — |
| F-136 | None shown; retain result | Store \(Y\) as a word at \(20+4096K\). |
| F-137 | \(1024K,\ 1024K,\ J\ll14,\ Y\) | — |
| F-138–F-140 | None shown | — |
| F-141 | None shown; retain result | Store \(Y\) as a word at \(20+4\rho_{141}\). |
| F-142 | \(4\rho_{141}\) | — |
| F-143–F-145 | None shown | After F-145 returns, replace the incoming-link-state variable with the fixed continuation-derived value, immediately before F-146. |
| F-146 | None shown; retain renewal result | Perform the post-call renewal described below. |

Under the fixed-width model, F-115’s first argument is zero: \(V\) already has at least fourteen low zero bits before the additional shift by 28. F-115 remains a required invocation, and its second argument remains unresolved.

**The continuation-state replacement precedes F-146.** It is not part of the post-call input renewal. F-146’s represented target is constructed from the continuation established at F-145; the link-state replacement is a separate state event. Whether F-146 also consumes that replaced state implicitly is unresolved, so moving the replacement across the call is not justified.

After F-146 returns, renewal proceeds in this order:

1. Replace the first input with \(4\rho_{146}\).
2. Replace the fourth input with zero.
3. Store the new first input as a word at \(20+16\rho_{146}\).
4. Replace the second input with the new first input.
5. Replace the third input with the new first input.

Execution then repeats at F-001. Every reached iteration after the first opens with three equal explicit arguments and uses zero for its paired fourth-input publications.

If F-146 does not return, the pre-call continuation-state replacement has already occurred, but none of these post-call renewal operations is established. If the boundary store fails, the first and fourth inputs have been replaced, while the second and third replacements have not yet occurred.

## Side effects

Established explicit effects comprise the ordered calls, byte and word writes, computed reads, one global snapshot, continuation-state replacement before F-146, and input renewal after F-146.

Byte address components are unsigned. A read byte of 255 selects address 272 when combined with offset 17. Word addresses such as \(20+b\) need not be aligned.

Aliasing is permitted. Byte destinations may overlap later byte sources; word stores may overlap relevant bytes or the global’s storage. The global is not established as independent of smaller overlapping storage. Endianness and the consequences of overlapping word and byte accesses remain unresolved.

No explicit allocation, release, ownership transfer, or lifetime-management operation is established. Dependencies cannot be assigned these responsibilities without additional evidence.

The ordered account specifies represented sequencing, not atomicity, synchronization, or hardware memory-ordering guarantees.

## Dependencies

The invocation IDs establish sites and order only. They do not establish distinct target identities, complete signatures, purity, global preservation, ownership, or error conventions.

F-001 constructs its target from incoming link state plus 4094. On subsequent iterations it uses the continuation-state value installed immediately before the preceding F-146, subject to the limitations of the supplied representation.

Other targets are constructed from represented continuation-like values plus displacements. Most use 4094; exceptions are:

| Sites | Displacement |
|---|---:|
| F-021, F-033 | 4086 |
| F-045 | 3976 |
| F-062, F-089, F-125 | 4036 |
| F-101 | 4056 |
| F-113 | 3986 |

These calculations establish neither target validity nor implementation identity. Calls with no shown arguments may consume implicit machine state. Calls whose results are unused remain required events.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Each assumes earlier required events complete.

1. **Opening payload.** With first input `0x12345678` and \(A=3\), the store after F-005 writes that word at address 32.

2. **Repeated byte publication.** With fourth input `0xA5`, both publication pairs write `0xA5` at \(B+17\) before \(B+1\). The second pair precedes F-028.

3. **Quarter-base wraparound.** Values \(Q=0,1,2,3,4\) produce bases `0x00000000`, `0x40000000`, `0x80000000`, `0xC0000000`, and zero. Address validity does not follow.

4. **Unaligned extraction.** If \(E(X)=1\), the word read after F-032 targets address 21 without rounding. It follows that row’s word store and precedes F-033.

5. **Clear before forwarding.** If F-039 returns 6, address 44 is cleared as a word before F-040 receives 24.

6. **Retained byte.** If \(b=4\), the extraction-byte destination is 21 and the later saved-selector word destination is 24, even if intervening dependencies change the original byte source.

7. **Global snapshot.** If \(G=\texttt{0xFFC00000}\), the snapshot-derived byte payload after F-061 is `0xFF`, regardless of later changes to the global.

8. **Final destination.** If F-141 returns 9, \(Y\) is stored at address 56 before F-142 receives 36.

9. **Pre-call continuation replacement.** After F-145 returns, the incoming-link-state variable is replaced before F-146 begins. If F-146 never returns, no boundary store or input renewal follows. Any implicit consumption of the replaced state by F-146 remains documented_only.

10. **Successful renewal.** If F-146 returns 7, the first input becomes 28 and the fourth becomes zero before 28 is stored at address 132. Only afterward do the second and third inputs become 28. The next F-001 receives 28 in all three explicit positions.

11. **Unbounded execution.** If every call returns and every access succeeds indefinitely, execution continues iterating rather than returning a status.

## Uncertainties

The following remain **documented_only**:

- Application-level purpose and intended protocol.
- Accuracy of the supplied control-flow and calling-convention reconstruction.
- Valid incoming link state and actual dependency identities or signatures.
- Implicit machine-state inputs, including possible consumption of the replaced continuation state by F-146.
- Provenance and validity of F-115’s additional register argument.
- Mapped addresses, alignment requirements, endianness, access-fault behavior, concurrency, and memory-ordering guarantees.
- Dependency effects on memory, globals, ownership, and lifetimes.
- External mechanisms that might terminate or redirect execution.
- Behavior of the separately reported omitted unreachable region.

These unknowns do not justify changing the established ordering of continuation-state replacement, F-146, and post-call renewal.

## Evidence

**Established static evidence:** four explicit inputs; 146 ordered invocation sites; represented argument expressions and consumed results; byte and 32-bit memory accesses; split 64-bit results; one explicit global read; continuation-state replacement immediately before F-146; input replacements and boundary store after F-146; unconditional repetition.

**Interpretive modeling:** modulo-\(2^{32}\) arithmetic provides a fixed-width account where signed-looking expressions do not independently establish portable arithmetic semantics. Phase names and mathematical value roles are explanatory, not evidence of application data structures.

**Inferred intent:** none asserted.

No dependency implementation or executable trace was supplied. No runtime tests have been performed.
