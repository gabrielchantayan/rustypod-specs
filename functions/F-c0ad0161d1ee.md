<!-- Trusted provenance: F-c0ad0161d1ee; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c0ad0161d1ee

## Purpose

F-c0ad0161d1ee describes a side-effecting operation whose supplied static representation contains 100 indirect invocations, computed memory accesses, and a terminal invalid-instruction boundary. No normal return is represented.

An application-level purpose is not established. The evidence does not support identifying this operation as allocation, initialization, serialization, decoding, or resource management.

This contract specifies the represented observable behavior, conditional on execution reaching each event. Invalid-instruction warnings, unresolved dependencies, and inputs drawn from undeclared execution state limit its authority as a description of actual machine execution.

## Interface

### Declared inputs and result

The declared interface accepts three 32-bit words, designated **first**, **second**, and **third**.

- The first invocation receives the explicit argument tuple \((first, first, third, first)\).
- Second has no explicit use in the represented body. This does not establish that dependencies cannot observe its incoming location or related machine state.
- No explicit return value or successful return path is represented.
- The terminal invalid-instruction marker is not a demonstrated normal return, status result, or recoverable error.

### Additional execution-state inputs

The three declared words do not fully determine behavior. Additional inputs include:

- Incoming link state, used to derive the first invocation target.
- An ambient address and an ambient byte used by the first direct byte transfer.
- An auxiliary 32-bit value exposed through recovered register state.
- Memory contents at computed addresses.
- Dependency results and dependency effects on machine state, memory, resources, and control flow.

The auxiliary value observed before one invocation is not established to equal the value observed after intervening invocations.

### Widths and notation

Contract symbols below identify values, not implementation-local names.

- A **byte** contains eight bits. Byte reads used in arithmetic are zero-extended.
- A **word** contains 32 bits.
- A byte write retains the low eight bits of its value.
- A word access transfers 32 bits.
- The one represented 64-bit result is divided into its low word \(L\) and high word \(U\). This numerical split does not establish memory byte order.
- \(Q(x)=4x\), \(S(x)=16x\), and \(H(x)=2^{28}x\).
- \(\operatorname{low8}(x)\) denotes the low eight bits.
- Right shifts of unsigned quantities below are logical.

Address expressions use byte addresses. No object layout is established beyond access widths and explicit offsets.

For conditional examples, scaling, addition, and left shifts use 32-bit bit-vector arithmetic, retaining the low 32 bits. The supplied representation mixes signed-looking and unsigned-looking operations; it does not independently establish a general signed-overflow policy or the target’s address-wrap behavior.

## Behavior

### Ordering and dependency notation

Opaque identifiers **F-d001** through **F-d100** designate invocation occurrences in represented execution order. They do not assert 100 distinct target functions.

A range denotes successive invocations in ascending order. An invocation without listed arguments has no explicit arguments in the supplied representation; this does not establish its actual calling convention or which live machine values it consumes. Likewise, an unlisted result may exist even though the represented body does not explicitly consume it.

Every event is conditional on all preceding accesses and invocations permitting execution to continue. No branch, loop, retry, recovery path, or rollback is represented.

### Entry and initial byte transfers

F-d001 receives \((first, first, third, first)\); F-d002 follows.

After F-d002, a byte is read at the ambient address plus 1. If its zero-extended value is \(a\), the ambient byte is written at address \(a+17\). This destination is between 17 and 272 inclusive; it is not generally relative to the ambient source address.

F-d003 receives an auxiliary word twice, followed by that word scaled by 16. F-d004–F-d006 follow, and F-d007 supplies a word \(A\).

After F-d007, let \(X\) denote the auxiliary value observed at this later point. A byte is read at \(S(X)+1\), producing selector \(b\). The operation writes \(\operatorname{low8}(X\gg22)\) at address \(b+17\). Intervening dependencies prevent assuming that \(X\) equals the earlier auxiliary observation.

F-d008 receives \(Q(A)\). F-d009–F-d011 follow, and F-d012 supplies \(B\). F-d013 receives \((b,Q(B))\), followed by F-d014.

F-d015 receives two zero words. The represented shift of an already-zero intermediate does not change either explicit argument.

### Initial word effects

F-d016 supplies the paired result with low word \(L\) and high word \(U\). The first four direct word writes are ordered as follows:

| Position | Destination address | Stored word | Subsequent dependency sequence |
|---|---:|---:|---|
| After F-d016 | \(Q(U)+20\) | \(0\) | F-d017 receives \(U\); F-d018–F-d020 follow; F-d021 supplies \(C\). |
| After F-d021 | \(Q(C)+20\) | \(U\) | F-d022 receives \(Q(C)\); F-d023–F-d026 follow. |
| After F-d026 | \(H(L)+20\) | \(2^{26}L\) | F-d027 receives \(2^{26}L\); F-d028–F-d030 follow; F-d031 supplies \(D\). |
| After F-d031 | \(Q(D)+20\) | \((L\mathbin{\&}\mathtt{0x1fffffff})\gg9\) | F-d032 receives \(Q(D)\); F-d033–F-d035 follow; F-d036 supplies \(E\). |

The stored words and argument expressions remain distinct from the validity or meaning of their computed addresses.

### Scaled-address word effects

The next ten direct word writes are listed below. Define:

\[
e=Q(E),\qquad g=Q(G),\qquad j=Q(J),\qquad
m=Q(M),\qquad n=Q(N),\qquad p=Q(P).
\]

These symbols retain the represented values used by later expressions. They do not assert preservation of unrelated globals or ambient registers.

| Position | Destination address | Stored word | Events before the next row |
|---|---:|---:|---|
| After F-d036 supplies \(E\) | \(S(E)+20\) | \(e\) | F-d037 receives arguments formed by two word reads, as specified below. F-d038–F-d040 follow; F-d041 supplies \(F\). |
| After F-d041 | \(Q(F)+20\) | \(e\) | F-d042 receives \(Q(F)\); F-d043–F-d045 follow; F-d046 supplies \(G\). |
| After F-d046 | \(S(G)+20\) | \(g\) | F-d047 receives \((g,g,L\ll26)\); F-d048–F-d050 follow; F-d051 supplies \(I\). |
| After F-d051 | \(Q(I)+20\) | \(g\) | F-d052 receives \(Q(I)\); F-d053–F-d055 follow; F-d056 supplies \(J\). |
| After F-d056 | \(S(J)+20\) | \(j\) | F-d057 receives \(j\); F-d058–F-d060 follow; F-d061 supplies \(K\). |
| After F-d061 | \(Q(K)+20\) | \(j\) | F-d062 receives \(Q(K)\); F-d063–F-d065 follow; F-d066 supplies \(M\). |
| After F-d066 | \(S(M)+20\) | \(m\) | A word is read at \(m+20\), retaining value \(V\). F-d067 receives \(m\); F-d068–F-d070 follow; F-d071 supplies \(N\). |
| After F-d071 | \(n+20\) | \(V\gg12\) | F-d072 receives \(n\); F-d073–F-d075 follow; F-d076 supplies \(P\). |
| After F-d076 | \(S(P)+20\) | \(p\) | F-d077 receives the arguments specified below. F-d078–F-d080 follow; F-d081 supplies \(R\). |
| After F-d081 | \(Q(R)+20\) | \(p\) | F-d082 receives \(Q(R)\); F-d083–F-d085 follow; F-d086 supplies \(T\). |

There are two distinct address scales in this sequence. A write at \(S(E)+20\), for example, does not generally supply a later read at \(Q(E)+20\). Neither a common offset nor a related scale establishes a shared field or object.

**F-d037’s reads and arguments.** After the write following F-d036, two explicit word reads occur at \(e+20\). One supplies the first argument after a logical right shift by 12; the other supplies the third argument after a left shift by 2. The middle argument is \(e\).

The representation does not establish the relative evaluation order of those two reads. Both precede the invocation. They must not be replaced by a requirement that one cached observation supply both arguments: concurrently changing or device-backed memory could produce different observations.

**Retained read \(V\).** The word read at \(m+20\) follows the write at \(S(M)+20\) and precedes F-d067. The two addresses generally differ. Later, the write following F-d071 uses the retained observation \(V\), logically shifted right by 12.

**F-d077’s arguments.** Its explicit tuple is:

\[
\bigl((n\mathbin{\&}63)\ll14,\ p,\ N\ll30\bigr).
\]

Under the conditional 32-bit model, the first argument depends only on bits 0–3 of \(N\), and the third only on bits 0–1. Neither is derived from \(V\).

### Final byte transfers and endpoint

After F-d086 supplies \(T\), a byte is read at \(H(N)+1\). If it yields selector \(r\), the operation writes \(\operatorname{low8}(Q(T))\) at address \(r+17\).

F-d087 then receives \((Q(T),Q(T),S(T))\). Its represented target displacement differs from the predominant displacement; that difference is part of the evidence.

F-d088 supplies a word \(W\), which F-d089 receives. F-d090–F-d092 follow, and F-d093 supplies \(Y\).

After F-d093, the operation performs a **fresh byte read** at \(H(N)+1\). Let this observation be \(c\). It writes:

\[
\operatorname{low8}\bigl((T\mathbin{\&}\mathtt{0x3fffffff})\gg20\bigr)
\]

at address \(c+17\). The earlier read of that address does not determine \(c\), especially because multiple dependencies intervene.

F-d094 receives \(Q(Y)\). F-d095–F-d097 follow, and F-d098 supplies \(Z\). F-d099 receives \((Q(Z),c)\), followed by F-d100.

The representation then ends at an invalid-instruction marker. No subsequent cleanup, recovery, continuation, or normal return is established.

## Side effects

### Direct memory effects

If the entire represented sequence is reached, it contains exactly:

| Access kind | Count | Basis |
|---|---:|---|
| Byte writes | 4 | Two initial transfers and two final transfers |
| 32-bit writes | **14** | Four initial word writes and ten scaled-address word writes |
| Byte reads | 4 | One selector read for each byte write |
| 32-bit reads | **3** | Two reads forming F-d037’s arguments and one read retaining \(V\) |

These counts exclude dependency-internal accesses, instruction fetches, and unrepresented calling-convention machinery. There is no fourth explicit word read.

All four byte-write destinations are formed by adding 17 to a zero-extended byte, giving the numeric range 17–272. Accessibility of those addresses is not established.

Aliasing remains possible:

- Distinct formulas may resolve to the same address, including under conditional wraparound.
- A direct write may affect a later read.
- A byte write may overlap storage accessed by a word operation.
- Dependencies may change memory between direct accesses.

The recurring offset 20 does not establish a structure layout, allocation size, array stride, or ownership boundary. Byte order, alignment requirements, and access atomicity remain unresolved.

### Ownership and lifetime

No direct allocation, deallocation, ownership transfer, reference counting, or lifetime extension is established.

Computed addresses and invocation arguments must not be classified as owned or borrowed merely from their use here. Dependencies may acquire, retain, or release resources; their contracts are unavailable.

### Preconditions and failures

The representation contains no explicit checks for invalid addresses, alignment, arithmetic overflow, invalid call targets, dependency failure, resource exhaustion, or unsuitable ambient state.

Reaching an event requires every preceding invocation and access to complete sufficiently for execution to continue. No transactional behavior or rollback is represented. Fault delivery, partial effects, exception propagation, and persistence of earlier writes are platform-dependent unknowns.

## Dependencies

F-d001–F-d100 are editorial opaque IDs for invocation occurrences, not verified dependency identities.

Their established properties are limited to represented ordering, explicit arguments, consumed results, and target calculations:

- F-d001 derives its target from incoming link state plus \(\mathtt{0xffe}\).
- The remaining targets combine embedded location-like values with a displacement.
- The predominant displacement is \(\mathtt{0xffe}\).
- F-d087 instead uses \(\mathtt{0xfec}\).

These calculations do not establish executable targets, distinct implementations, valid calling conventions, normal return, or preservation of registers, globals, memory, or resources.

F-d016 is represented as supplying a 64-bit result. Its low/high decomposition is explicit, but the actual return mechanism is calling-convention-sensitive and unresolved.

## Acceptance examples

These are conditional criteria, not executed tests. Unless otherwise stated, they assume accessible memory, returning dependencies, and 32-bit bit-vector arithmetic.

1. **Declared arguments.** With first equal to 7 and third equal to 11, F-d001 receives \((7,7,11,7)\). Changing second alone changes no explicit expression in the represented body; effects through unmodeled machine state remain unresolved.

2. **Selector destination boundary.** An initial selector of 255 and ambient byte of 171 produce a byte write of 171 at address 272. A selector of zero instead selects address 17.

3. **Shift followed by byte truncation.** If the later auxiliary observation is \(\mathtt{0xffffffff}\), its logical right shift by 22 is 1023. The corresponding byte write stores 255.

4. **Separated result halves.** If F-d016 supplies \(L=1\) and \(U=2\), the immediate word write stores zero at address 28, and F-d017 receives 2. The later write based on \(L\) stores \(\mathtt{0x04000000}\) at \(\mathtt{0x10000014}\).

5. **Masked extraction.** With \(L=\mathtt{0xffffffff}\), the write following F-d031 stores \(\mathtt{0x000fffff}\). The upper three bits are removed before the right shift.

6. **Distinct scaled addresses.** If F-d036 supplies 3, the next write stores 12 at address 68. F-d037’s argument reads use address 32. The write cannot be assumed to provide those observations.

7. **Independent argument observations.** If F-d037’s first-argument read observes \(\mathtt{0x00012000}\) and its third-argument read observes 5, its tuple is \((18,e,20)\). This criterion does not impose which read occurs first.

8. **Retained word observation.** If the read retaining \(V\) observes \(\mathtt{0x12345000}\), the write following F-d071 stores \(\mathtt{0x00012345}\). A later change to the read location does not itself replace the retained value.

9. **Low-bit argument extraction.** If F-d071 supplies \(N=3\), then \(n=12\). F-d077’s first argument is \(\mathtt{0x00030000}\), and its third is \(\mathtt{0xc0000000}\).

10. **Fresh final selector.** If the first read at \(H(N)+1\) observes 4 and the later read observes 9, the corresponding writes target addresses 21 and 26. Reusing the first selector would not satisfy the represented behavior.

11. **Reachability-limited effects.** If F-d047 does not return, no later invocation or direct write is required to occur. The fourteen-word-write count describes the full represented sequence, not guaranteed effects of every attempted execution.

## Uncertainties

### Established static evidence

The supplied representation contains a candidate linear sequence of indirect invocations, explicit value transformations, fourteen direct word writes, three direct word reads, four byte reads, four byte writes, and an invalid-instruction endpoint. It contains no normal return.

### Inferred intent

Repeated scales and offsets may suggest structured storage or transformed data. No particular object model, application purpose, or resource protocol is sufficiently supported to become an acceptance requirement.

A 32-bit bit-vector model supports the conditional arithmetic examples. It is not independent proof of every target-machine arithmetic rule.

### documented_only unknowns

The following remain **documented_only**:

- Correct instruction decoding, control-flow completeness, and actual reachability.
- Dependency identities, signatures, hidden inputs, and return conventions.
- Provenance and stability of auxiliary execution-state values.
- Preservation of globals, registers, memory, and resources across dependencies.
- Valid address ranges, byte order, alignment rules, and atomicity.
- Concurrency and device-memory behavior.
- Signed arithmetic, overflow, and address-wrap semantics.
- Fault handling and survival of earlier effects.
- The actual machine event corresponding to the terminal invalid-instruction marker.

These unknowns must not be resolved by assuming conventional function behavior.

## Evidence

This contract is based only on the supplied static representation. The prior draft and feedback are not independent evidence of behavior.

The direct-access accounting follows the supplied body: four initial word writes plus ten subsequent word writes; two word reads forming F-d037’s arguments plus one read retaining \(V\).

The material includes both an initial invalid-instruction warning and a terminal invalid-instruction boundary. Those constrain confidence in the recovered interface and control flow.

Dependency IDs are editorial labels for invocation occurrences. Arithmetic examples are conditional deductions, not observed results.

No implementation access, independent instruction inspection, runtime execution, or tests have been performed.
