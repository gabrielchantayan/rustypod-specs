<!-- Trusted provenance: F-39be3fbec775; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-39be3fbec775

## Purpose

F-39be3fbec775 exposes an ordered sequence of indirect calls, derived-address word stores, selector-directed byte stores, and repeated clearing of a word at address `0x24`.

The supplied static representation does not establish a domain-level purpose. Initialization, registration, serialization, linking, and teardown are possible interpretations, not established behavior. Numerical relationships between stored values and addresses do not establish an object layout.

The visible sequence ends at a reported invalid-instruction boundary rather than an explicit return. This contract therefore describes a conditional execution prefix: each operation is required only if execution reaches it and the preceding calls and accesses complete.

## Interface

### Explicit inputs

| Input | Width | Established direct use |
|---|---:|---|
| First input | 32 bits | No explicit use in the visible body. |
| Second input | 32 bits | One word store at an address derived from an early dependency result. |
| Third input | 32 bits | Two word stores at independently derived addresses. |

The inputs are opaque bit patterns. No semantic signedness, ownership, pointer validity, accepted range, or caller-side validation is established.

The lack of a direct use of the first input does not prove that it is irrelevant to the complete machine interface. Incompletely described indirect calls may observe incoming argument state not represented as explicit arguments.

### Additional entry-state dependencies

The visible behavior also requires:

- Incoming link state, used to derive the first indirect-call target.
- An incoming unsigned 32-bit value, logically shifted right by three for the first call.
- An incoming 32-bit address operand, multiplied by `0x80000` for the first store of the third input.

These dependencies are additional to the three explicit inputs. An interface exposing only those inputs is not established as sufficient for equivalent behavior.

### Results and completion

No explicit returned value or successful return is visible. Possible outcomes include a completed prefix of effects, dependency-controlled nonreturn, a memory or control-transfer fault, and arrival at the reported invalid-instruction boundary.

There is no visible status result, error translation, rollback, or cleanup path.

### Widths and notation

A **word** access means a 32-bit access. A **byte** access means an 8-bit access. Selector reads yield unsigned values from zero through 255.

The following mathematical labels describe retained values and relationships; they are not additional storage requirements:

| Label | Meaning |
|---|---|
| \(A\) | Early dependency result used for the second-input store. |
| \(U\) | Early unsigned 32-bit dependency result. |
| \(P\) | \(16U\). |
| \(Q\) | \((U \mathbin{\&} \texttt{0x3fffffff}) \gg 1\). |
| \(K\) | \(Q \times \texttt{0x400}\). |
| \(M\) | \((Q \mathbin{\&} \texttt{0x3fffff}) \ll 7\). |
| \(N\) | \(Q \times \texttt{0x4000000}\). |

Other capital letters below identify results retained at particular dependency boundaries.

For any unsigned result \(v\), its **high-field byte** is the low eight bits of \((v \mathbin{\&} \texttt{0x3fffffff}) \gg 20\). The intermediate can contain ten bits; the stored byte contains only original bits 20 through 27.

A **selector update at base \(b\)** means an 8-bit read at \(b+1\), followed by an 8-bit write at the zero-extended selector plus `0x11`. It does not mean loading a pointer from the selected location. The selected destination lies between `0x11` and `0x110`, inclusive.

The representation uses 32-bit arithmetic quantities, including some signed intermediate forms. Where arithmetic wraps, the examples use a 32-bit machine interpretation: results retained in 32 bits discard higher bits. Original language-level signed overflow and signed-shift rules are not established. Endianness, alignment requirements, atomicity, and low-address mapping are also unknown.

## Behavior

### Ordering and dependency notation

F-D001 through F-D128 identify successive indirect-call occurrences, not proven distinct target identities. A range such as F-D002–F-D005 contains every call in that range, in order. When its last result is named, earlier results in that range have no explicit subsequent use.

Every call remains an observable boundary. No-argument notation means no arguments are displayed, not that the call has no machine-level inputs. Unused results do not imply effect-free calls.

The following phases specify the required event order without assuming that dependencies leave memory or globals unchanged.

### Entry and input placement

F-D001 targets incoming link state plus `0xffe` and receives the incoming unsigned entry value shifted right by three.

| Dependency boundary | Required subsequent direct effect |
|---|---|
| F-D002–F-D005; F-D005 supplies \(A\) | Store the second input as a word at \(4A+\texttt{0x14}\). |
| F-D006 receives \(4A\); F-D007–F-D010 follow, with F-D010 supplying \(U\) | Store the third input as a word at the additional incoming address operand times `0x80000`, plus `0x24`. |
| F-D011 receives \(P\) and \(U \ll 2\); F-D012–F-D015 follow | Store the third input as a word at \(P+\texttt{0x24}\). |

The two third-input stores are distinct events with five indirect calls between them. No direct read of the first destination is shown. The addresses may coincide for some entry states; such coincidence does not eliminate either event.

The values \(P\) and \(Q\) derive from the \(U\) acquired at F-D010, not from later results.

### Early selector updates

A word read at \(P+\texttt{0x74}\) supplies the displayed argument to F-D016. The read occurs before that call; F-D016 is not established to receive the address itself.

| Following dependency boundaries | Required subsequent byte effects |
|---|---|
| F-D017–F-D020; F-D020 supplies \(B\) | Read the selector at \(P+1\), then write the high-field byte of \(U\) to its selected destination. |
| F-D021 receives \(B \ll 2\); F-D022 supplies a result passed to F-D023; F-D024–F-D027 follow, with F-D027 supplying \(C\) | Reread the selector at \(P+1\), then write zero to its selected destination. |
| F-D028 receives \(C \ll 2\); F-D029–F-D032 follow | First write zero directly at \(Q+\texttt{0x0f}\). Then read the selector at \(K+1\) and write zero to its selected destination. |

The two selector reads at \(P+1\) are independent observations. Their destinations need not match. Dependencies, overlapping direct writes, or external agents may change the selector between reads.

### Saved selector and first word-clearing pair

F-D033 receives four explicit arguments: three copies of \(B \ll 23\), followed by \(B \ll 25\). These arguments use the earlier result \(B\), not \(C\) or \(U\).

F-D034 supplies a result passed to F-D035. F-D036–F-D039 follow, with F-D039 supplying \(D\).

The next selector read is at \(K+1\). Its unsigned byte value is retained as \(S\), and zero is written at \(S+\texttt{0x11}\).

F-D040 then receives \(D \ll 2\), followed by F-D041–F-D044. After those calls:

- A zero word is stored at \(S+\texttt{0x44}\).
- A zero word is then stored at address `0x24`.

The word address uses the saved selector \(S\); it does not reread \(K+1\). Its possible range is `0x44` through `0x143`, including potentially unaligned addresses.

### Further clearing and split-result handling

F-D045 receives `0`, `0x98`, \(K\), and `0x98`. F-D046 supplies a result passed to F-D047; F-D048 supplies another result passed to F-D049. F-D050–F-D053 then run, with F-D053 supplying \(E\).

The next effects, in order, are a zero word store at \(4E+\texttt{0x44}\), followed by a zero word store at `0x24`.

F-D054 receives \(4E\). F-D055–F-D058 follow, with F-D058 supplying a displayed 64-bit result. Its low and high 32-bit halves have separate uses:

- F-D059 receives the low half shifted left by two.
- After F-D059, a zero word is stored at twice the retained high half plus `0x44`.
- A zero word is then stored at `0x24`.

The high-half-derived address is based on the F-D058 result, despite the intervening F-D059 call.

F-D060 receives three displayed arguments: zero, an additional register-result value, and that value shifted left by two. The representation exposes this as extra register-result state rather than an ordinary explicit returned value. Its full provenance and ABI interpretation remain unresolved; it must not be substituted with either half of the 64-bit result.

F-D061 follows without displayed arguments. F-D062 receives \(M\). F-D063–F-D066 follow, with F-D066 supplying \(G\). The next effects are a zero word store at \(4G+\texttt{0x44}\), then a zero word store at `0x24`.

At this point, the visible prefix has performed four separate zero word stores to `0x24`. They are not one logical initialization that may be moved or deduplicated.

### Zeroing and numerical-value stores

The next phase establishes exact address/value relationships without establishing a data-structure interpretation.

| Dependency boundaries | Required subsequent word store |
|---|---|
| F-D067 receives \(4G\); F-D068–F-D071 follow; F-D072 receives F-D071’s result shifted left by two and supplies \(H\) | Zero at \(4H+\texttt{0x14}\). |
| F-D073 receives \(H,H,H,M\); F-D074–F-D077 follow, with F-D077 supplying \(I\) | Zero at \(4I+\texttt{0x14}\). |
| F-D078 receives \(4I\); F-D079–F-D082 follow, with F-D082 supplying \(J\) | The value \(L=4J\) at \(16J+\texttt{0x14}\). |
| F-D083 receives \(L\); F-D084–F-D087 follow, with F-D087 supplying \(R\) | The previously retained \(L\) at \(4R+\texttt{0x14}\). |
| F-D088 receives \(4R\); F-D089–F-D092 follow, with F-D092 supplying \(T\) | The value \(4T\) at \(16T+\texttt{0x14}\). |

In particular, the store based on \(R\) reuses \(L\). It does not replace that value with \(4R\).

### Mask-derived word and late selector updates

F-D093 receives \(4T\) and \(16T\). F-D094 supplies a result passed as the first argument to F-D095; the second displayed argument is \((Q \mathbin{\&} \texttt{0x3fffff}) \ll 4\).

F-D096–F-D099 follow, with F-D099 supplying \(V\). A word containing \(M\) is stored at \(4V+\texttt{0x14}\).

F-D100 receives \(4V\). F-D101–F-D104 follow, with F-D104 supplying a fresh unsigned 32-bit result \(W\).

| Boundary | Required byte effect |
|---|---|
| After F-D104 | Read the selector at \(N+1\), then write the low eight bits of \(W \ll 2\) to its selected destination. |
| F-D105 receives \(W \ll 2\); F-D106 supplies a result passed to F-D107; F-D108–F-D111 follow, with F-D111 supplying \(X\) | Reread the selector at \(N+1\), then write the high-field byte of \(W\) to its selected destination. |
| F-D112 receives \(X \ll 2\); F-D113–F-D116 follow, with F-D116 supplying a fresh unsigned 32-bit result \(Y\) | Reread the selector at \(N+1\), retain it as \(Z\), and write the low eight bits of \(4Y\) at \(Z+\texttt{0x11}\). |

F-D117 receives the zero-extended selector \(Z\). Its displayed argument is not \(Y \ll 2\), \(N\), or the byte just written.

All three reads at \(N+1\) are fresh reads. The function does not establish a stable selector across the intervening dependencies.

### Final update and visible endpoint

F-D118 supplies a result passed to F-D119. F-D120–F-D123 follow, with F-D123 supplying \(AA\).

The final visible selector read is at \(8Y+1\), not at \(N+1\). The high-field byte of \(Y\) is then written to the selected destination.

F-D124 receives \(AA \ll 2\). F-D125–F-D128 follow. The representation then reports an invalid-instruction boundary.

No explicit return, final status update, or cleanup is visible after that boundary.

## Side effects

Conditional on traversing the complete visible prefix, the direct memory effects are:

| Access category | Count and distribution |
|---|---|
| Selector-byte reads | **Eight**: two at \(P+1\), two at \(K+1\), three at \(N+1\), and one at \(8Y+1\). |
| Selector-directed byte writes | **Eight**, one following each selector read. |
| Other byte writes | **One**: zero at \(Q+\texttt{0x0f}\). |
| Total byte writes | **Nine**. |
| Word reads | One, at \(P+\texttt{0x74}\). |
| Word stores | Seventeen: three input stores, four derived-address clears with displacement `0x44`, four clears at `0x24`, two derived-address clears with displacement `0x14`, and four numerical-value stores with displacement `0x14`. |

These are access-event counts, not counts of distinct addresses. Address aliasing can make multiple events touch the same bytes.

No disjointness is established. A selector-directed byte write can overlap a later selector, a word field, or dependency-observed memory. The four-byte region beginning at `0x24` can overlap selector-directed destinations. Such overlap does not permit suppressing or reordering the accesses.

No explicit allocation, release, ownership transfer, reference-count update, lock, or synchronization operation is identifiable in the direct effects. Dependencies may perform any of these. No ownership or lifetime guarantee can be inferred merely because a numeric result is later used to form an address.

There is no visible transaction boundary. Completed effects remain potentially observable if a later call fails, faults, or does not return.

## Dependencies

The F-D identifiers are document-local opaque IDs for call occurrences. Different IDs may resolve to the same target; apparent similarity does not establish interchangeability.

Most targets are represented as fixed code-reference values plus an offset of `0xffe`. Exceptions include:

- F-D021 and F-D033: offset `0xfc4`.
- F-D045: offset `0xf1c`.
- F-D047 and F-D060: offset `0xfec`.
- F-D059 and F-D072: offset `0xe02`.
- F-D093, F-D105, and F-D117: offset `0xff0`.

F-D001 instead depends on incoming link state. No resolved dependency contract is supplied.

Consequently, this contract does not assume that any dependency:

- Returns normally.
- Preserves globals, selectors, derived-address memory, or address `0x24`.
- Preserves unspecified registers or incoming argument state.
- Allocates, retains, or releases memory in any particular way.
- Accepts only the displayed arguments.
- Is effect-free when its displayed result is unused.

Removing intermediate calls while retaining only direct stores would not preserve the established behavior.

## Acceptance examples

These are conditional static predictions, not executed tests.

1. **Initial placement.** If \(A=3\), the second input is stored at `0x20`. That store precedes F-D006 with explicit argument `12`.

2. **Distinct input-store events.** If \(U=2\) and the additional incoming address operand is zero, the third-input destinations are `0x24` and `0x44`. Both stores remain required, with F-D011–F-D015 between them.

3. **Unsigned selector interpretation.** A selector value of `0xfe` selects destination `0x10f`. It is neither a negative displacement nor a pointer-sized dereference.

4. **High-field narrowing.** For a relevant unsigned result of `0x3ff00000`, the masked shift yields `0x3ff`, but the byte stored is `0xff`.

5. **Fresh reads.** If the first selector read at \(P+1\) yields `2` and an intervening dependency changes it to `7`, the first selected destination is `0x13` and the subsequent zeroing destination is `0x18`.

6. **Saved selector.** If \(S=5\), the saved-selector word clear targets `0x49`, even if F-D040–F-D044 change the byte at \(K+1\). Whether that unaligned word access is permitted remains platform-dependent.

7. **Retained numerical value.** If \(J=3\) and \(R=5\), the corresponding stores place `12` at `0x44` and then `12` at `0x28`. The latter value is not `20`.

8. **Repeated clearing.** If a dependency writes a nonzero word at `0x24` between explicit clearing events, the later explicit clear still writes zero. One consolidated clear is insufficient.

9. **Late selector argument.** If \(Z=9\), F-D117 receives `9`, independently of \(Y \ll 2\).

10. **Complete byte trace.** If every call returns and every access succeeds, the trace contains eight selector reads, eight corresponding selected byte writes, and the additional direct zero byte write. Equal destinations do not reduce these event counts.

11. **32-bit wraparound.** Under the stated machine-arithmetic interpretation, \(U=\texttt{0x10000000}\) gives \(P=0\); the later third-input store therefore targets `0x24`. This is conditional on permitted wraparound and addressability.

12. **Interrupted execution.** If F-D059 does not return, its preceding effects may remain, but the following high-half-derived word clear and subsequent stores are not required to occur. No rollback is established.

## Uncertainties

The following remain **documented_only**:

- Complete calling convention, register inputs, and true return interface.
- Resolved identities and contracts of indirect dependencies.
- Original signed arithmetic, overflow, and shift semantics.
- ABI interpretation of the displayed 64-bit result and additional register-result state.
- Actual executable boundaries and the nature of the reported terminal invalid instruction.
- Memory mapping, endianness, alignment tolerance, volatility, atomicity, and fault delivery.
- Valid input and entry-state ranges.
- Ownership, lifetime, and higher-level data-structure meaning.
- Normal completion beyond the visible endpoint.

Traversal of the entire prefix requires each reached target to be executable and returning, and each reached access to be permitted. These are operational prerequisites, not evidence of validation or an established accepted input domain.

## Evidence

Established static evidence is limited to the supplied representation: displayed input widths, arithmetic relationships, call order, memory-access widths, saved-result reuse, and the reported endpoint.

The representation also reports overlapping global symbols. That report does not independently establish an object layout; the repeated clears are therefore specified by address and width rather than by a presumed global-object identity.

The byte totals follow the explicit access sites: \(2+2+3+1=8\) selector reads, eight associated selector-directed writes, and one additional direct byte write.

Inferred intent is limited to recognizing clearing operations and numerical relationship construction. No domain-specific purpose or ownership model is asserted.

No implementation access, dependency inspection, runtime execution, or runtime tests have been performed. The acceptance examples describe conditional required behavior, not observed outcomes.
