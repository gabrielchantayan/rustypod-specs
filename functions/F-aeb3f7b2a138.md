<!-- Trusted provenance: F-aeb3f7b2a138; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aeb3f7b2a138

## Purpose

F-aeb3f7b2a138 performs an ordered sequence of indirect calls, four-byte writes, and byte-selected writes, then enters an unconditional loop containing further calls and byte-selected writes. No normal return appears on the represented path.

No application-level purpose is established. Numeric addresses, scaling factors, and fixed offsets do not establish an allocated object, record format, device protocol, or ownership model.

This contract describes static evidence, not observed execution. Invalid-instruction and excluded-region notices limit conclusions about complete executable control flow.

## Interface

### Explicit inputs

| Contract name | Width and interpretation | Established use |
|---|---|---|
| \(P\) | Signed 32-bit integer | Stored twice; supplies the first two write-address calculations. |
| Second argument | Opaque 32-bit quantity | No explicit use in the represented body. |
| \(T\) | Unsigned 32-bit integer | Logically shifted right by seven twice; the resulting value is passed to F-d039. |

Absence of an explicit use of the second argument does not establish that dependencies cannot observe its incoming machine representation.

### Additional machine-state inputs

- \(R\): an incoming unsigned 32-bit value used in addresses, data, and arguments.
- An incoming link-related value used to form F-d001’s target.
- \(\delta\): a signed 32-bit secondary result observed immediately after F-d048. Its relationship to that call’s ordinary result is unknown.

### Outputs and preconditions

There is no explicit return value or represented normal exit after loop entry.

Progress requires each reached call to return and each reached access to succeed. Executable targets, accessible addresses, applicable alignment, and compatible machine conventions are unchecked preconditions. No validation, error status, rollback, or recovery mechanism is represented.

## Behavior

### Arithmetic and conversion rules

Word values and address expressions have represented width 32 bits. Word writes occupy four bytes; byte accesses occupy one eight-bit byte.

Call tables identify explicit arguments and consumed ordinary results. \(\varnothing\) means no explicit arguments are displayed, not that hidden inputs are absent. A dependency range denotes one call to each identifier in increasing order.

Signed ordinary results are \(a,d,g,h,n,p,q,s,t\) and \(\alpha,\beta,\gamma,\epsilon,\zeta,\eta,\theta,\iota,\kappa\). Results \(b,c,e,f,j,k,l,m,o\) are opaque 32-bit quantities. \(X,Y,Z\) are unsigned 32-bit results.

Define:

\[
M=(8R)\bmod 2^{32},\qquad A_{\mathrm u}=(M+20)\bmod 2^{32}.
\]

Both expressions use unsigned arithmetic. F-d025 receives \(M\); W6 uses \(A_{\mathrm u}\).

After W7, \(M\) is separately converted to signed 32-bit \(H\). F-d039 receives \(H\) twice; W8 stores \(H\) using the signed address expression \(H+20\). For \(M\le 2^{31}-1\), \(H=M\). Larger conversions are unresolved. W8’s signed addition is not established when it overflows.

Signed multiplication remains signed even when its result is subsequently converted to unsigned. Signed left shifts additionally require a nonnegative operand and representable result for the portable arithmetic interpretation used here. Assignment to an unsigned destination does not retrospectively make a signed shift unsigned.

### Plain-character byte conversion

B1, B2, and B3 convert a value to **plain character type** before storing one byte. Define \(\mathcal C(v)\) as that conversion and the resulting stored byte representation.

The source establishes the conversion and access width, but not an unconditional low-byte truncation rule:

- Plain character signedness is unknown.
- With unsigned plain characters, conversion uses reduction modulo 256.
- With signed plain characters, an out-of-range conversion requires implementation-defined behavior; a particular value, stored bit pattern, or successful continuation is not established.
- Signed character representation is not independently established.

Consequently, \(\mathcal C(v)\) is not generally specified as \(v\bmod256\). Low-byte descriptions below apply only under an established **truncating, bit-preserving byte-conversion interpretation**. Any preceding uncertain unsigned-to-signed conversion remains a separate condition.

The explicit zero-byte writes do not depend on these three plain-character conversions.

### Ordered word writes and calls

Each write precedes every call in its row. Those calls precede the next write.

| Event | Four-byte write: value at address | Subsequent calls |
|---|---|---|
| W1 | \(P\) at \(4P+20\) | F-d001(\(R\gg8\)); F-d002–F-d004; F-d005(\(\varnothing\)) → \(a\) |
| W2 | \(P\) at \(P+20\) | F-d006(\(a\ll2\)); F-d007–F-d009; F-d010(\(\varnothing\)) → \(\alpha\) |
| W3 | \(4\alpha\) at \(16\alpha+20\) | F-d011(\(16\alpha\)); F-d012(\(\varnothing\)) → \(b\); F-d013(\(b\)); F-d014(\(\varnothing\)) → \(c\); F-d015(\(c,R\gg8\)); F-d016–F-d018; F-d019(\(\varnothing\)) → \(d\) |
| W4 | Retained \(4\alpha\) at \(16\alpha+20\) | F-d020(\(4d\)); F-d021–F-d023; F-d024(\(\varnothing\)) → \(\beta\) |
| W5 | \(4\beta\) at \(16\beta+20\) | F-d025(\(4\beta,M\)); F-d026(\(\varnothing\)) → \(e\); F-d027(\(e\)); F-d028(\(\varnothing\)) → \(f\); F-d029(\(f,R\gg8\)); F-d030–F-d032; F-d033(\(\varnothing\)) → \(g\) |
| W6 | Retained \(4\beta\) at \(A_{\mathrm u}\) | F-d034(\(4g\)); F-d035–F-d037; F-d038(\(\varnothing\)) → \(\gamma\) |
| W7 | \(4\gamma\) at \(16\gamma+20\) | Establish \(H\); F-d039(\(H,H,R\gg8,T\gg14\)); F-d040–F-d042; F-d043(\(\varnothing\)) → \(h\) |
| W8 | \(H\) at signed expression \(H+20\) | F-d044(\(h\ll2\)); F-d045–F-d047; F-d048(\(\varnothing\)); observe \(\delta\) |
| W9 | \(2^{24}\delta\) at \(2^{26}\delta+20\) | F-d049(\(2^{24}\delta\)); F-d050–F-d052; F-d053(\(\varnothing\)) → \(\epsilon\) |
| W10 | Retained \(2^{24}\delta\) at \(4\epsilon+20\) | F-d054(\(4\epsilon\)); F-d055–F-d057; F-d058(\(\varnothing\)) → \(\zeta\) |
| W11 | \(4\zeta\) at \(16\zeta+20\) | F-d059(\(4\zeta\)); F-d060(\(\varnothing\)) → \(j\); F-d061(\(j\)); F-d062(\(\varnothing\)) → \(k\); F-d063(\(k\)); F-d064–F-d066; F-d067(\(\varnothing\)) → \(\eta\) |
| W12 | Retained \(4\zeta\) at \(4\eta+20\) | F-d068(\(4\eta\)); F-d069–F-d071; F-d072(\(\varnothing\)) → \(\theta\) |
| W13 | \(U\), the unsigned conversion of signed product \(4\theta\), at \(16\theta+20\) | F-d073(\(U\)); F-d074(\(\varnothing\)) → \(l\); F-d075(\(l\)); F-d076(\(\varnothing\)) → \(m\); F-d077(\(m\)); F-d078–F-d080; F-d081(\(\varnothing\)) → \(\iota\) |
| W14 | Unsigned \(R\gg16\) at \(4\iota+20\) | F-d082(\(4\iota\)); F-d083–F-d085; F-d086(\(\varnothing\)) → \(\kappa\) |
| W15 | \(4\kappa\) at \(16\kappa+20\) | Derive \(Q\); F-d087(\(16\kappa,16\kappa,R\gg15\)); F-d088–F-d090; F-d091(\(\varnothing\)) → \(n\) |
| W16 | Retained \(16\kappa\) at \(16\kappa+20\) | F-d092(\(n\ll2\)); F-d093–F-d095; F-d096(\(\varnothing\)) → \(X\) |

Exactly nine calls intervene between W3 and W4. W4 repeats the \(\alpha\)-derived write; F-d019’s result instead supplies F-d020.

W8 stores retained \(H\), not \(h\). F-d044 consumes \(h\). F-d048’s ordinary result is not explicitly consumed.

The first shift of \(T\) occurs before W5. Its second shift forms F-d039’s fourth argument. Both are unsigned logical shifts.

### Byte selection and retained values

After W15 and before F-d087:

\[
Q=(U\mathbin{\&}\text{0x7fffffff})\gg7,
\qquad 0\le Q\le\text{0x00ffffff}.
\]

Thus \(Q\) derives from \(\theta\), not \(\kappa\).

A **Q-selected write** first reads an unsigned byte at \(Q+1\), then writes one byte at address 17 plus the byte read. The destination is 17–272 inclusive, not an offset from \(Q\). Every selection performs a fresh read.

Let \(x_2\) and \(y_2\) be the signed conversions of the unsigned 32-bit expressions \(X\ll2\) and \(Y\ll2\). Each unsigned shift discards bits beyond width 32 before conversion. Conversion when the resulting high bit is set remains unresolved.

### Ordered byte events before the loop

| Event | Explicit memory effects | Subsequent calls |
|---|---|---|
| B1 | Q-selected write of \(\mathcal C(x_2)\) | F-d097(\(x_2,x_2,x_2\)); F-d098(\(\varnothing\)) → \(o\); F-d099(\(o\)); F-d100–F-d102; F-d103(\(\varnothing\)) → \(p\) |
| B2 | Fresh Q-selected write of \(\mathcal C((X\mathbin{\&}\text{0x3fffffff})\gg20)\) | F-d104(\(p\ll2\)); F-d105–F-d107; F-d108(\(\varnothing\)) → \(Y\) |
| B3 | Write \(\mathcal C(y_2)\) at \((R\gg23)+28\) | None |
| B4 | Fresh Q-selected write of zero | F-d109(\(R\gg23\)); F-d110(\(\varnothing\)); F-d111(\((Y\mathbin{\&}\text{0x3fffffff})\gg6\)); F-d112–F-d114; F-d115(\(\varnothing\)) → \(q\) |
| B5 | Fresh Q-selected write of zero | F-d116(\(q\ll2\)); F-d117–F-d119; F-d120(\(\varnothing\)) → \(Z\) |
| B6 | Read an unsigned byte at \(S=1024y_2+1\); write zero at address 17 plus that byte | F-d121(\(V\ll1\)); F-d122(\(\varnothing\)) → initial carried result |

Here,

\[
V=(Z\mathbin{\&}\text{0x3fffffff})\gg25,
\qquad 0\le V\le31.
\]

Before its plain-character conversion, B2’s value is in 0–1023. **Only under truncating, bit-preserving byte conversion** does its stored byte equal bits 20–27 of \(X\). Under that same condition, B1 and B3 store the low bytes of \(x_2\) and \(y_2\), respectively. Expressing those bytes directly from \(X\) and \(Y\) additionally requires compatible preceding signed conversions wherever those conversions are out of range.

B3’s unsigned destination calculation ranges from 28 through 539 without overflow. B3 completes before B4’s selector read, with no intervening call. Aliasing can therefore change B4’s destination.

The expression defining \(S\) uses retained \(y_2\), not \(q\), \(Z\), or a later loop result. Its multiplication and addition are signed.

F-d102 is a separate call with a code-symbol-derived target, between F-d101 and F-d103.

### Unconditional loop

If reached and its operations complete, the loop executes at least once.

| Position | Ordered behavior |
|---|---|
| Entry | First iteration: F-d123a(initial carried result, \(V\gg8\)). Later iterations: F-d123b(updated carried result, current unsigned loop word shifted right by eight). |
| Before byte access | F-d124–F-d126; F-d127(\(\varnothing\)) → \(s\). |
| Byte access | Fresh unsigned byte read at \(S\), then a zero-byte write at address 17 plus that byte. |
| After byte access | F-d128(\(s\ll2\)); F-d129–F-d131; F-d132(\(\varnothing\)) → \(t\). |
| State transition | Next carried result receives signed \(t\ll2\). Next unsigned loop word receives the result of signed \(t\ll26\). Repeat without a represented condition. |

F-d123a and F-d123b have different target expressions. Other loop call targets repeat.

The first entry call’s second argument is necessarily zero. Under an additional 32-bit wrapping, bit-preserving interpretation, a later entry call’s second argument is:

\[
(t\mathbin{\&}63)\ll18.
\]

That formula does not establish portable behavior for an otherwise invalid signed shift.

## Side effects

The direct effects are sixteen four-byte writes, six pre-loop byte writes with their specified selector reads, and one selected zero-byte write per loop iteration, conditional on reaching and completing each operation.

No non-aliasing guarantee exists. Word destinations may overlap each other; byte destinations may overlap word destinations or selector bytes. Dependencies may access or alter the same memory.

Each selector read is fresh and cannot be replaced by a cached earlier value. Repeated writes remain distinct events even when their address and value expressions match.

B1–B3 establish one-byte access widths, but their exact stored representations depend on the plain-character conversions. Unresolved conversion behavior may also prevent establishing continuation beyond the affected event.

No explicit allocation, release, reference counting, synchronization, or ownership transfer is established. Dependencies must not be assumed to preserve globals, preserve memory contents, acquire ownership, or release memory.

Endianness, atomicity, barriers, concurrency guarantees, and ordinary-memory versus device-memory interpretation are unknown. Four-byte accesses establish width, not object layout.

If an access faults or a dependency does not return, later effects need not occur. No restoration of earlier effects is represented.

## Dependencies

Identifiers designate represented call occurrences or repeated loop targets, not established semantic identities.

| IDs | Established property |
|---|---|
| F-d001 | Target depends on incoming link-related state. |
| F-d002–F-d101 and F-d103–F-d122 | Targets use fixed numeric-base expressions; explicit arguments and consumed results appear above. |
| F-d048 | Observed secondary result supplies \(\delta\); ordinary result is not explicitly consumed. |
| F-d102 | Target uses a code-symbol-derived expression. |
| F-d123a, F-d123b | Different first-iteration and later-iteration entry-target expressions. |
| F-d124–F-d132 | Repeated loop call sites. |

Distinct identifiers do not prove distinct implementations or non-aliasing targets. Actual signatures, hidden inputs, additional outputs, calling conventions, memory effects, ownership behavior, faults, and nonreturning behavior remain unresolved.

## Acceptance examples

These are conditional static expectations, not runtime results. Required calls must return, accesses must succeed, and affected arithmetic and conversions must have an established interpretation.

1. **Initial ordering.** For \(P=4\), W1 writes 4 at address 36. F-d001 through F-d005 occur before W2 writes 4 at address 24.

2. **Repeated write.** For \(\alpha=2\), W3 and W4 each write 8 at address 52. Nine calls intervene. F-d011 receives 32; F-d020 receives four times F-d019’s result. Intervening calls may alter address 52.

3. **Unsigned product wrap.** For \(R=\text{0x20000000}\), \(M=0\). F-d025’s second argument is zero; W6 targets address 20. Separately, \(H=0\), so F-d039’s first two arguments are zero and W8 writes zero at address 20.

4. **Unsigned versus signed addition.** For \(R=\text{0x0ffffffe}\), \(M=\text{0x7ffffff0}\). W6’s unsigned destination is \(\text{0x80000004}\). Conversion to \(H\) is exact, but W8’s signed addition exceeds the signed 32-bit range. W6 does not establish W8’s address.

5. **Secondary result.** If \(\delta=1\), W9 writes \(\text{0x01000000}\) at \(\text{0x04000014}\); F-d049 receives \(\text{0x01000000}\). Neither F-d043’s result nor F-d048’s ordinary result substitutes for \(\delta\).

6. **Unsigned input shift.** For \(T=\text{0xffffffff}\), F-d039’s fourth argument is \(\text{0x0003ffff}\).

7. **Fresh selection and conditional byte values.** Selector values 5 and subsequently 9 select addresses 22 and 26. For \(X=\text{0x0ab00035}\), \(x_2=\text{0x2ac000d4}\), and B2’s pre-conversion value is 171. Under truncating, bit-preserving byte conversion, B1 stores \(\text{0xd4}\) and B2 stores \(\text{0xab}\), with seven intervening calls. Without that conversion interpretation, neither stored byte is established: both conversions exceed the range of an eight-bit signed plain character.

8. **Representable character values.** For \(X=1\), \(x_2=4\), and B2’s pre-conversion value is zero. These character values are representable regardless of plain character signedness; the out-of-range uncertainty does not apply.

9. **Adjacent effects.** For \(R=\text{0xffffffff}\), B3 targets address 539. If its conversion and write complete, B4’s selector read follows without an intervening call.

10. **Negative signed shift.** If \(a=-1\), F-d006’s argument is not established as \(-4\) merely because \(-4\) is representable. The negative signed shift requires executable-level interpretation.

11. **Loop transition.** F-d123a first receives zero as its second argument. Later entries use F-d123b. For \(t=3\), the next carried argument is 12 and the next second argument is \(\text{0x000c0000}\); both signed shifts are nonnegative and representable.

## Uncertainties

### documented_only: arithmetic, conversions, and addresses

Unsigned multiplication, addition, masks, and shifts follow the specified 32-bit modular or logical operations.

Signed multiplication and addition support their mathematical results only when all relevant intermediates are representable. Signed left shifts additionally require nonnegative operands and representable results. These restrictions affect the table entries and both loop updates.

Out-of-range unsigned-to-signed conversions remain unresolved, notably for \(H,x_2,y_2\). Out-of-range conversion to signed plain character is a separate uncertainty affecting B1–B3. Neither a low-byte value nor successful continuation follows merely from the one-byte destination width. No authoritative target semantics supplied here resolves these conversions.

Under an additional wrapping, bit-preserving word-arithmetic model:

\[
S=((4096Y)\bmod2^{32})+1.
\]

This is conditional, not an independently established address guarantee. Numeric-to-pointer interpretation remains environment-dependent.

### documented_only: reconstruction and environment

Invalid-instruction and excluded-region notices prevent establishing complete executable control flow. Missing exits or misidentified operations cannot be excluded. The no-normal-return statement concerns the represented path.

Instruction mode, endianness, character signedness and representation, alignment enforcement, fault delivery, actual target validity, and secondary-result conventions remain unresolved.

### Inferred intent

Repeated scaling and fixed offsets suggest numeric structure, but establish neither application intent nor ownership. No intent-based assumption is required by this contract.

## Evidence

Established static evidence comprises represented widths and operation categories, explicit access order, indirect-call occurrences, explicit arguments, consumed results, F-d048’s secondary-result use, fresh selector reads, plain-character conversions at B1–B3, and the unconditional loop with different first and subsequent entry targets.

Low-byte formulas are conditional interpretations, not established target behavior.

No runtime tests, execution traces, observed memory effects, dependency implementation inspection, or independent executable validation have been performed.
