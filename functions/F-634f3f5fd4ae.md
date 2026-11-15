<!-- Trusted provenance: F-634f3f5fd4ae; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-634f3f5fd4ae

## Purpose

F-634f3f5fd4ae repeatedly invokes indirect dependencies, transforms their results, and performs byte and 32-bit memory accesses. Results from one iteration determine arguments, addresses, and stored values in later iterations.

The static representation contains an unconditional loop and no normal exit. “Nonterminating” is conditional: an invalid access, exceptional transfer, or dependency that does not return can prevent further progress.

No application-level purpose is established. The evidence does not identify an allocator, serialization format, collection operation, or device protocol. Small numeric addresses are observable address calculations, not evidence of a particular address space or object layout.

## Interface

### Explicit arguments and output

The represented interface accepts four 32-bit arguments and has no return value.

| Argument | Established use |
|---|---|
| First | Passed to the first dependency in each iteration; replaced at the iteration boundary by a dependency result shifted left by two bits. |
| Second | No explicit use is represented. This does not establish that dependencies cannot observe its incoming machine representation. |
| Third | Initially supplies the low four bits of a shifted dependency argument. Replaced during the iteration by a dependency result, subsequently used in a word write and retained for the following iteration. |
| Fourth | Determines an early byte-read address and a shifted dependency argument. Replaced near the iteration boundary. |

There is no represented status output, error result, or normal completion.

### Other consumed state

Before their first visible initialization, the representation consumes:

- A 32-bit base for the initial byte read.
- An incoming byte payload.
- A link-like value used to derive the first indirect target.

These are additional entry-state requirements of the represented dataflow, not additional declared arguments.

A separate, register-derived argument is consumed by F-D020. Its value and provenance are unresolved. The representation does **not** establish that it is supplied at entry, nor that it survives preceding dependencies unchanged.

The function also samples one shared 32-bit location during each iteration.

### Widths and notation

The following notation describes a 32-bit bit-vector interpretation:

- \(L_k(x)\): shift \(x\) left by \(k\), retaining the low 32 bits.
- \(\operatorname{byte}(x)\): retain the low eight bits.
- \(B(x)\): read one unsigned byte at address \(x\).
- \(W(x)\): read one 32-bit word at address \(x\).
- \(H(x)=\operatorname{byte}((x\mathbin{\&}\texttt{0x3fffffff})\gg20)\).

All address additions and scaling in this account retain the low 32 bits. Multiplication by four, sixteen, or other powers of two is expressed as the corresponding \(L_k\).

Right shifts of explicitly unsigned quantities are logical. A represented 64-bit result is divided into its low and high 32-bit halves; passing those halves does not establish their in-memory order.

This arithmetic model captures the apparent machine-width operations. It does not assert that signed overflowing expressions or shifts are valid portable source-language operations.

### Preconditions

Continued execution requires each reached address to support the specified access width, each indirect target to be executable under the actual calling convention, and each dependency to return to the represented continuation. No visible checks enforce these requirements.

Word accesses are not universally aligned. No supplied object, buffer length, ownership token, or address-space descriptor bounds the accesses.

## Behavior

### Call notation and ordering

F-D001 through F-D146 identify successive indirect-call sites within one complete iteration. They do not establish distinct target implementations.

A **five-call run** names five consecutive sites: the first receives the stated explicit arguments, and the remaining four have no explicit arguments shown. A named result belongs to the fifth call unless otherwise stated. “No explicit arguments” does not mean that ambient registers are cleared or unobservable.

The stages below specify the represented order. Every later action is conditional on earlier calls returning and earlier accesses completing. Mathematical result names identify data dependencies, not allocated objects or validated pointers.

### 1. Initial byte write and first result

Read a byte at the incoming hidden base plus one. Write the incoming hidden payload byte to address 17 plus that selector.

Then execute the five-call run F-D001–F-D005, beginning with the current first argument. F-D001’s target depends on the incoming link-like state. Denote F-D005’s 32-bit result by \(A\).

The initial byte read and write precede all dependency calls in the iteration.

### 2. Fourth-argument selectors and first word write

Let \(P=L_2(\text{fourth argument})\).

Read \(B(P+1)\), then write zero to address 17 plus that byte. Next execute:

1. F-D006 with \(L_{10}(\text{fourth argument})\).
2. F-D007, producing a 32-bit result.
3. F-D008 with that result and \(L_{28}(\text{third argument})\).
4. F-D009–F-D012 with no explicit arguments.

Denote F-D012’s result by \(B_0\). Read \(B(P+1)\) again and write zero through this newly obtained selector. The first selector is not reused.

Execute the five-call run F-D013–F-D017, beginning with \(L_2(B_0)\). No final result is explicitly retained.

Write the 32-bit value \(L_{14}(A)\) to address \(L_{14}(A)+20\).

### 3. First mirrored word and extracted byte

Execute:

- F-D018 with \(L_{16}(A)\).
- F-D019 with no explicit arguments.
- F-D020 with zero as its first argument and the unresolved register-derived value as its second.
- F-D021–F-D024 with no explicit arguments.

The zero argument follows from applying a further 28-bit left shift to \(L_{14}(A)\), under 32-bit truncation. This establishes nothing about F-D020’s second argument.

Let \(C\) be F-D024’s result. Write \(L_{14}(A)\) to \(L_2(C)+20\).

Execute F-D025–F-D029 as a five-call run beginning with \(L_2(C)\), producing \(D\).

Freshly read \(B(P+1)\), then write \(H(L_{14}(A))\) to address 17 plus that selector. The stored byte contains original bits 6 through 13 of \(A\).

### 4. Split-result forwarding and second mirrored value

Execute:

1. F-D030 with \(L_{26}(A)\).
2. F-D031, producing a 32-bit result.
3. F-D032 with that result and \(L_{30}(D)\).
4. F-D033, producing a represented 64-bit result.
5. F-D034 with its low half followed by its high half.
6. F-D035 with no explicit arguments.
7. F-D036, producing \(E\).

Freshly read \(B(P+1)\) and write zero through that selector.

Execute the five-call run F-D037–F-D041 beginning with \(L_2(E)\), producing \(F\). Write \(L_2(F)\) to \(L_{14}(D)+20\).

F-D042 receives four arguments, in order:

1. \(L_{12}(D)\).
2. \(L_{12}(D)\).
3. \(L_{14}(D)\).
4. \(L_2(F)\).

F-D043–F-D046 follow without explicit arguments. Let their final result be \(G\). Write \(L_2(F)\) to \(L_2(G)+20\).

### 5. Two scaled-word stages

Execute F-D047–F-D051 as a five-call run beginning with \(L_2(G)\), producing \(J\).

Write \(L_2(J)\) to \(L_4(J)+20\). F-D052 receives three identical arguments, each \(L_2(J)\). F-D053–F-D056 follow without explicit arguments, producing \(K\). Write \(L_2(J)\) to \(L_2(K)+20\).

Execute F-D057–F-D061 as a five-call run beginning with \(L_2(K)\), producing \(N\).

Write \(L_2(N)\) to \(L_4(N)+20\), and retain the base \(Q=L_{30}(N)\). F-D062 receives three identical arguments, each \(L_2(N)\). F-D063–F-D066 follow without explicit arguments, producing \(R\). Write \(L_2(N)\) to \(L_2(R)+20\).

Execute F-D067–F-D071 as a five-call run beginning with \(L_2(R)\), producing \(S\).

The later high-bit payload depends on the retained result \(R\), not on a fixed address-derived constant.

### 6. Repeated direct byte clearing

Write zero to \(Q+17\), then zero to \(Q+1\). These are direct byte writes, without selector reads.

Execute:

- F-D072 with \(L_2(S)\).
- F-D073, producing a result.
- F-D074 with that result.
- F-D075–F-D078 with no explicit arguments.

Let \(T\) be F-D078’s result.

Again write zero to \(Q+17\), then zero to \(Q+1\). Both repetitions are represented even though the same addresses were cleared earlier. Intervening dependencies may change memory.

### 7. Parity-dependent word and selected word read

Execute F-D079–F-D083 as a five-call run beginning with \(L_2(T)\), producing \(U\).

Write the 32-bit payload \(L_{31}(R)\) to \(L_4(U)+20\). Thus:

- Even \(R\) produces zero.
- Odd \(R\) produces `0x80000000`.

Only bit zero of \(R\) survives. The payload is neither fixed nor independent of preceding dependency results.

Next read the 32-bit word

\[
V=W\!\left(20+((S\mathbin{\&}\texttt{0x3fffffff})\gg20)\right).
\]

The extracted offset ranges from 0 through 1023, so the starting address ranges from 20 through 1043 and need not be aligned.

F-D084 receives \(L_2(U)\). F-D085 produces \(Y\), replacing the current third argument.

### 8. Word clearing and new byte payload

F-D086 receives \(L_{28}(V\gg22)\). Only original bits 22 through 25 of \(V\) survive.

F-D087–F-D090 follow without explicit arguments, producing \(Z\). Write a zero word to \(L_2(Z)+20\).

Execute F-D091–F-D095 as a five-call run beginning with \(L_2(Z)\), producing \(I\).

Read \(B(Q+1)\), then write \(\operatorname{byte}(L_2(I))\) to address 17 plus that selector.

### 9. Fresh selector, then cached-selector word write

Execute:

- F-D096 with \(L_2(I)\).
- F-D097, producing a result.
- F-D098 with that result and \(L_{30}(I)\).
- F-D099–F-D102 with no explicit arguments.

Let \(O\) be F-D102’s result.

Freshly read \(B(Q+1)\), retaining this byte as selector \(b\). Write \(H(I)\) to \(b+17\).

Execute F-D103–F-D107 as a five-call run beginning with \(L_2(O)\). Then write the 32-bit value \(L_2(Y)\) to \(b+20\).

This word write uses the cached selector. It does not reread \(Q+1\), even if the intervening calls changed that location or the preceding byte write aliased it.

### 10. Shared-word snapshot and another byte sequence

Read the shared 32-bit location, retaining the sampled value as \(G_{\mathrm{shared}}\). This read occurs after the cached-selector word write and before F-D108.

Execute F-D108–F-D112 as a five-call run beginning with \(L_2(U)\). F-D112 produces a represented 64-bit result; denote its low half by \(a\) and high half by \(h\).

Freshly read \(B(Q+1)\), then write \(\operatorname{byte}(G_{\mathrm{shared}}\gg22)\) through that selector plus 17. The payload uses the earlier shared-word sample, not a new shared-location read.

Execute F-D113 with \(h\), F-D114 producing a result, and F-D115 with that result and \(L_{30}(a)\). F-D116–F-D119 follow without explicit arguments, producing \(c\).

Freshly read \(B(Q+1)\) and write zero through that selector plus 17.

### 11. Final mirrored words and fourth-argument replacement

Execute F-D120–F-D124 as a five-call run beginning with \(L_2(c)\), producing \(d\).

F-D125 receives \(L_2(d)\) twice. F-D126–F-D129 follow without explicit arguments, producing \(e\).

Write \(L_2(e)\) to \(L_{14}(a)+20\).

Execute F-D130–F-D134 as a five-call run beginning with \(L_{16}(a)\), producing \(f\). Write \(L_2(e)\) to \(L_2(f)+20\).

Execute F-D135–F-D139 as a five-call run beginning with \(L_2(f)\). F-D139 produces a represented 64-bit result with low half \(g\) and high half \(j\).

Replace the fourth argument with \(L_2(g)\).

Read a byte at \(L_{16}(a)+1\), then write \(H(L_2(e))\) to address 17 plus that selector.

### 12. Iteration feedback

Execute:

1. F-D140 with \(j\).
2. F-D141, producing a result.
3. F-D142 with that result.
4. F-D143, producing a represented 64-bit result.
5. F-D144 with its low and high halves, in that order.
6. F-D145 with no explicit arguments.
7. F-D146, producing a 32-bit result.

Replace the first argument with \(L_2(\text{F-D146 result})\).

At the next iteration:

- The hidden payload byte is zero.
- The hidden byte-read base is \(L_{30}(g)\).
- The third argument is \(Y\).
- The fourth argument is \(L_2(g)\).
- The first indirect target uses link-like state established by the preceding iteration rather than the original entry state.

Control returns unconditionally to the initial byte read. No return branch is represented.

## Side effects

A complete represented iteration directly performs:

- Ten byte reads.
- Fourteen byte writes.
- Two 32-bit reads: the selected word and the shared word.
- Thirteen 32-bit writes.
- 146 indirect calls.

These counts exclude all dependency-internal effects.

Unsigned byte selectors range from 0 through 255. Selector-plus-17 byte writes therefore target addresses 17 through 272. The cached-selector word write starts between addresses 20 and 275 inclusive and spans four bytes.

The representation establishes access widths and starting addresses, but not endianness, atomicity, bus transactions, or volatile semantics. Dependencies and direct accesses may alias each other or the shared location.

No allocation, release, ownership transfer, reference counting, or lifetime guarantee is established. A result used in an address calculation is not thereby a valid pointer.

There is no visible rollback, transaction boundary, bounds check, null check, alignment correction, error translation, or recovery path. A failure can leave preceding writes and dependency effects in place.

## Dependencies

F-D001–F-D146 are opaque call-site IDs. Their actual identities, signatures, and relationships are unresolved.

Targets are formed from address-like intermediates and fixed displacements. F-D001 additionally depends on link-like state. No resolved dependency contract establishes:

- Memory or global-state preservation.
- Ownership acquisition or release.
- Normal return, exception, or fault behavior.
- Validity of returned address-like values.
- Which ambient machine state is consumed by calls without explicit arguments.

The forwarding of a represented 64-bit result’s halves is established at the specified sites. Its application-level meaning is unknown.

## Acceptance examples

These are conditional static criteria, not executed tests. Each assumes the preceding path is reached successfully.

1. **Entry selector boundary.** If the initial byte read returns 255, the first write stores the incoming payload at address 272 before F-D001.

2. **Fresh early selector.** If the first read at \(P+1\) returns 3 and the second returns 9, the corresponding zero writes target addresses 20 and 26. Reusing the first selector is inconsistent with the representation.

3. **Cached selector across dependencies.** If stage 9 samples selector 7, its byte write targets 24 and its later word write starts at 27. A subsequent change to \(Q+1\) does not redirect that word write.

4. **Truncated shift.** F-D020’s first argument is zero for every \(A\) under the stated width model. Its second argument remains unconstrained by this calculation.

5. **Parity-dependent payload.** If F-D066 returns 2, stage 7 writes zero. If it returns 3, stage 7 writes `0x80000000`. Both writes start at \(L_4(U)+20\). Changing higher bits of \(R\) without changing its parity does not change this payload.

6. **Unaligned selected read.** If the extraction from \(S\) equals one, stage 7 reads a word at address 21. No rejection or alignment adjustment is represented; whether the environment permits the access is unknown.

7. **Shared-word snapshot.** If the shared-word sample is `0x00400000`, stage 10’s payload is one. A dependency changing the shared location after that sample does not change the retained payload.

8. **No normal completion.** If all accesses succeed and every dependency returns, the function begins another iteration rather than returning a success value.

## Uncertainties

### Established static evidence

The representation establishes the ordered accesses, indirect-call occurrences, unconditional iteration, width-dependent transformations, split-result forwarding, and feedback described above. In particular, stage 7 shifts the retained F-D066 result, so its payload depends on that result’s parity.

### Inferred intent

Repeated scaled-address writes and packed-bit transformations suggest a structured state transformation. No particular protocol, data structure, or operational purpose is established.

### documented_only unknowns

- Accuracy of the represented function boundary and control transfers.
- Address mapping, alignment rules, endianness, and access atomicity.
- Initialization and validity of the hidden entry base, byte, and link-like state.
- F-D020’s second argument, including when and how it becomes available.
- Dependency calling conventions, effects, ownership rules, and return guarantees.
- Shared-location identity and interpretation where symbols overlap.
- External concurrency, memory-mapped effects, and expected fault behavior.

Safe execution for arbitrary four-argument inputs is not established.

## Evidence

This contract is based solely on the supplied static representation. The parity-dependent payload follows from retaining F-D066’s result until its 31-bit left shift; no intervening represented assignment replaces that retained value. F-D020’s second argument has no established initialization or provenance in the supplied representation.

The supplied annotations report removed unreachable material and overlapping global symbols. These qualify confidence in recovered structure and shared-location interpretation; they do not establish additional runtime behavior.

No implementation inspection beyond the supplied material, dependency inspection, execution, instrumentation, or runtime testing has been performed. No provenance digest is asserted.
