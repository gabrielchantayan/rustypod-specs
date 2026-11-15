<!-- Trusted provenance: F-2cbf5c3439af; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2cbf5c3439af

## Purpose

The supplied static representation describes indirect dependency calls interleaved with byte reads, word reads, byte stores, and word stores. Addresses and values depend on the fourth explicit argument, dependency results, and incoming machine state outside the explicit argument list.

No higher-level purpose is established. Recurring offsets and zero stores do not establish object construction, initialization, serialization, or successful clearing of a subsystem.

This contract describes conditional observable behavior: an operation is reached only if preceding calls return and preceding memory accesses complete. The representation ends at an invalid-instruction boundary rather than an established normal return. Decoding warnings limit confidence that this representation corresponds to executable behavior.

## Interface

### Explicit inputs

| Input | Width | Established use |
|---|---:|---|
| First argument | 32 bits | Supplied to the first indirect call. |
| Second argument | 32 bits | No explicit use shown. |
| Third argument | 32 bits | No explicit use shown. |
| Fourth argument, \(D\) | Unsigned 32 bits | Its logical right shift by three supplies the initial quotient \(Q\). |

Unused explicit arguments may remain observable through machine state. Their absence from displayed dependency argument lists is not evidence that dependencies cannot observe them.

### Additional incoming state

The initial call target is derived from incoming return-link state plus `0xFEC`. A separate incoming register supplies the 32-bit value for the first direct store.

These are additional inputs to the displayed behavior, not ordinary arguments established by this interface.

### Outputs and completion

No normal return value or successful return path is established. Observable outputs consist of memory effects and dependency interactions.

Two calls supply results represented as 64-bit quantities, with independently consumed low and high 32-bit halves. Another call supplies an additional register result used outside an ordinary return-value representation.

Whether the terminal invalid-instruction marker means an actual trap, an incorrectly decoded continuation, or something else is `documented_only`.

### Widths and arithmetic

Unless stated otherwise:

- Values and computed addresses are 32-bit bit patterns.
- Addition, multiplication, and left shifts retain their low 32 bits.
- Right shifts explicitly described as logical insert zero bits.
- Byte reads yield unsigned values from 0 through 255.
- Byte stores retain the low eight bits; word accesses are 32 bits.

Define:

\[
Q=D\gg3,\qquad
\operatorname{highbyte}(x)=((x\mathbin{\&}\texttt{0x3FFFFFFF})\gg20)\bmod256.
\]

Thus \(\operatorname{highbyte}(x)\) is bits 20–27 of \(x\), not a ten-bit stored value.

This arithmetic is a conditional bit-vector interpretation, not a claim about portable semantics of overflowing signed expressions. Endianness, alignment rules, valid mappings, and device-memory behavior are unknown.

## Behavior

### Ordering and dependency notation

All calls below are to **F-unknown**, an opaque unresolved-dependency designation. Different occurrences need not identify the same callee.

A **four-call interval** means four consecutive calls with no displayed explicit arguments. The first three results are unused; the fourth result is either captured as stated or unused. A **forwarding pair** means a call with no displayed arguments immediately followed by a call receiving that result as its sole displayed argument.

These intervals and pairs are observable dependency interactions, not pure computations. They may modify memory or other state. “No displayed arguments” does not establish an ABI-level absence of inputs.

The sequence has no displayed validation, conditional branch, loop, retry, rollback, or error return. All following ordering claims are conditional on reaching the operations concerned.

### Entry and quotient-derived stores

The entry call receives the first argument. A subsequent call with no displayed arguments supplies a paired result. The next call receives its low 32-bit half, high 32-bit half, and low half again, in that order.

A four-call interval supplies a selector \(P\). The first word store writes the hidden incoming register value at \(4P+\texttt{0x14}\). The next call receives \(4P\).

After a four-call interval with unused results, a word store writes \(Q\times\texttt{0x80000}\) at \(Q\times\texttt{0x200000}+\texttt{0x14}\). The next call receives three copies of the stored value.

A four-call interval supplies another selector. Its word at four times the selector plus `0x14` receives \(Q\); the next call receives four times that selector.

All these are direct computed addresses. No allocator, object-base lookup, bounds check, or null check is established.

### Initial byte-selected stores

A four-call interval supplies \(U\). A byte read at \(Q+1\) supplies a selector \(a\). The selected byte at \(a+\texttt{0x11}\) receives the low byte of \(4U\).

The immediately following call receives, in order:

\[
a\gg3,\quad 4U,\quad 4Q.
\]

A four-call interval supplies a separate result \(E\). The byte at \(Q+1\) is then reread; call this observation \(B\). The byte at \(B+\texttt{0x11}\) receives \(\operatorname{highbyte}(U)\). The immediately following call receives \(4E\).

After a four-call interval with unused results, the word at \(4B+\texttt{0x14}\) receives \(4U\). The next call receives \(B,B,B,B\gg1\).

A four-call interval supplies another selector. Its word at four times the selector plus `0x14` also receives \(4U\). The next call receives four times that selector.

The two reads at \(Q+1\) are independent observations. Calls and potentially overlapping stores separate them; equality is not guaranteed.

### Intermediate scaled-word effects

A four-call interval supplies \(R\). The word at \(16R+\texttt{0x14}\) receives \(4R\), and the immediately following call receives \(4R\).

A four-call interval supplies a destination selector. The word at four times that selector plus `0x14` receives the captured \(4R\). The next call receives four times that destination selector.

A four-call interval supplies \(S\). The word at \(16S+\texttt{0x14}\) receives \(4S\). The next call receives \(4S\). A forwarding pair follows.

Another four-call interval supplies a destination selector. Its word at four times the selector plus `0x14` receives

\[
((4S)\mathbin{\&}\texttt{0x3FFFFFFF})\gg1.
\]

The next call receives four times that selector.

A four-call interval supplies \(T\). The word at \(16T+\texttt{0x14}\) receives \(4T\). **Without an intervening dependency call**, the word at \(4T+\texttt{0x14}\) is read. Its observed value, logically shifted right by three, becomes the next call’s argument.

Those two addresses generally differ, so the read cannot generally be replaced with the stored value. They coincide when \(12T=0\) modulo \(2^{32}\). Under this model, the preceding store then directly determines the read, absent external interference or nonordinary memory behavior.

A four-call interval supplies another destination selector. Its word at four times the selector plus `0x14` receives the captured \(4T\). The next call receives four times that selector.

The quantity \(B\gg3\), derived from the retained second selector observation, is used in the next group.

### Central word and byte effects

A four-call interval supplies \(V\). Define

\[
H=(V\mathbin{\&}\texttt{0x3FFFFFFF})\gg1.
\]

Before the next call, the word at \((B\gg3)\times\texttt{0x80000}+\texttt{0x24}\) receives \(B\). The next call receives \(16V,4V\).

After a four-call interval with unused results, the word at \(16V+\texttt{0x24}\) receives \(B\). The word at \(16V+\texttt{0x74}\) is then read and supplied as the next call’s sole displayed argument.

A four-call interval supplies a result \(C\). A byte read at \(16V+1\) selects a byte destination at the observed selector plus `0x11`; it receives \(\operatorname{highbyte}(V)\). The next call receives \(4C\), followed by a forwarding pair.

A four-call interval supplies another result \(G\). The byte at \(16V+1\) is reread, and its newly selected destination plus `0x11` receives zero. The next call receives \(4G\).

After a four-call interval with unused results, the following direct effects occur in order, without an intervening dependency call:

- A zero byte is stored at \(H+\texttt{0x0F}\).
- A byte is read at \(1024H+1\).
- A zero byte is stored at that selector plus `0x11`.

The next call receives three copies of \(C\ll23\), followed by \(C\ll25\). These arguments use the earlier captured \(C\), not \(G\).

A forwarding pair follows. A four-call interval then supplies another result. The byte at \(1024H+1\) is reread; call this observation \(b\). Its selected destination \(b+\texttt{0x11}\) receives zero, and the next call receives four times the separately obtained result.

After a four-call interval with unused results, the word at \(b+\texttt{0x44}\) receives zero. The retained \(b\) is used here without another selector read. Immediately afterward, the word at absolute address `0x24` receives zero.

### Word clearing and paired results

The call following that first absolute-address clear receives:

\[
0,\quad\texttt{0x98},\quad1024H,\quad\texttt{0x98}.
\]

Two forwarding pairs follow, in order. A four-call interval supplies a selector. The word at four times that selector plus `0x44` is cleared, followed immediately by the second word clear at absolute address `0x24`. The next call receives four times that selector.

A four-call interval supplies a paired 64-bit result. A call receives four times its captured low half. After that call returns, the word at twice the captured high half plus `0x44` is cleared, followed immediately by the third clear at absolute address `0x24`.

The next call receives zero, an additional register result from the call that received the scaled low half, and four times that additional result. The high-half selector is not reread or obtained from this later call.

One call with no displayed arguments follows, with its ordinary result unused. The next call receives

\[
M=(H\mathbin{\&}\texttt{0x3FFFFF})\times128.
\]

A four-call interval supplies a selector. The word at four times it plus `0x44` is cleared, followed immediately by the fourth clear at absolute address `0x24`. The next call receives four times that selector.

A four-call interval supplies a result. The next call receives four times that result and supplies a new selector \(Z\). The word at \(4Z+\texttt{0x14}\) is cleared. The following call receives \(Z,Z,Z,M\).

A four-call interval supplies another selector. The word at four times it plus `0x14` is cleared; the next call receives four times that selector.

### Final scaled-word effects

A four-call interval supplies a result. Four times that result is stored at sixteen times it plus `0x14`. The next call receives the constructed scaled value.

A four-call interval supplies an independent destination selector. Its word at four times the selector plus `0x14` receives the captured scaled value. The next call receives four times that destination selector.

A four-call interval supplies another result. Four times it is stored at sixteen times it plus `0x14`. The next call receives both forms, in order: four times the result, then sixteen times the result.

A call with no displayed arguments supplies a value. The next call receives that value and \((H\mathbin{\&}\texttt{0x3FFFFF})\times16\).

A four-call interval supplies a selector. Its word at four times the selector plus `0x14` receives \(M\). The next call receives four times that selector.

### Final byte effects and termination boundary

Define \(A=H\times\texttt{0x4000000}\), with 32-bit wrapping.

A four-call interval supplies \(J\). A byte read at \(A+1\) selects a destination at the observed byte plus `0x11`. That destination receives the low byte of \(4J\). **The immediately following call receives the full 32-bit \(4J\).** A forwarding pair follows.

A four-call interval supplies an independent result \(L\). The byte at \(A+1\) is reread; its selected destination plus `0x11` receives \(\operatorname{highbyte}(J)\). **The immediately following call receives \(4L\)**, not \(4J\) and not the observed selector.

A four-call interval supplies \(K\). The byte at \(A+1\) is read a third time; call this observation \(c\). The byte at \(c+\texttt{0x11}\) receives the low byte of \(4K\). The immediately following call receives the unsigned selector \(c\). A forwarding pair follows.

A four-call interval supplies an independent result \(N\). A byte is read at \(8K+1\), and its selected destination plus `0x11` receives \(\operatorname{highbyte}(K)\). **The immediately following call receives \(4N\)**.

Four calls with no displayed arguments and unused results follow. The supplied representation then ends at the invalid-instruction marker. No ordinary return is shown.

## Side effects

Direct effects include unsigned byte-selector reads, 32-bit reads used as dependency arguments, computed-address byte and word stores, and four distinct 32-bit zero stores at absolute address `0x24`.

Selector-based byte destinations range from `0x11` through `0x110`. The retained-selector word clear at \(b+\texttt{0x44}\) ranges from `0x44` through `0x143` and may be unaligned.

Different groups can overlap in memory. Direct stores, absolute-address clears, or dependency effects can change later observations. Captured selector values remain distinct from subsequent memory contents.

No allocation, ownership transfer, release, reference-count effect, or lifetime guarantee is established. Dependencies must not be assumed to preserve globals, preserve memory, or acquire or release ownership.

No fault recovery is shown. Invalid targets, inaccessible memory, alignment restrictions, nonreturning dependencies, or the final instruction boundary can prevent later effects. Earlier completed writes have no displayed rollback.

## Dependencies

Only F-2cbf5c3439af is resolved by the supplied dependency map. All displayed calls remain F-unknown.

Known dependency interactions include:

- The entry call using incoming return-link-derived targeting.
- Calls producing 32-bit selectors and two paired 64-bit results.
- Calls receiving scaled values, scaled destination selectors, captured byte observations, literal values, and observed words.
- Forwarding pairs and calls with unused ordinary results.
- A call whose additional register result is consumed afterward.

Most later targets are represented through fixed code-relative arithmetic with varying displacements. Actual identities, signatures, calling conventions, target validity, preservation rules, and additional effects are `documented_only`.

Unresolved identity does not make displayed arguments unknown. Every argument flow above remains part of the conditional contract.

## Acceptance examples

These are static, conditional expectations—not executed tests.

1. **Initial quotient boundary.** For \(D=8\), \(Q=1\). The quotient-derived store writes `0x00080000` at `0x00200014`; the initial selector read uses address `0x00000002`.

2. **Low-bit equivalence is limited.** For \(D=0\) through \(7\), \(Q=0\), so the quotient-derived store writes zero at `0x14`. Whole-function equivalence is not established because dependencies can observe other incoming state.

3. **Byte truncation and full-width arguments.** For \(J=\texttt{0x12345678}\), the final low-fragment store writes `0xE0`, and its immediately following call receives `0x48D159E0`. The later high-fragment store writes `0x23`; its following argument instead depends on the separately obtained \(L\).

4. **Independent repeated reads.** If the two reads at \(Q+1\) yield 5 and 9, their byte destinations are `0x16` and `0x1A`. The subsequent \(B\)-selected word store targets `0x38`.

5. **Immediate store/read alias.** For \(T=\texttt{0x40000000}\), both intermediate addresses reduce to `0x14`, and \(4T=0\). Under ordinary memory semantics without external interference, the store is followed directly by a read of that zero, so the next call receives zero. No dependency call separates those accesses.

6. **Distinct immediate addresses.** For \(T=1\), the store writes 4 at `0x24`, while the following read accesses `0x18`. Its value is not established by that store.

7. **Wrapping input arithmetic.** For \(D=\texttt{0xFFFFFFFF}\), \(Q=\texttt{0x1FFFFFFF}\). The quotient-derived store targets `0xFFE00014` and writes `0xFFF80000`.

8. **Clearing is not persistence.** Reaching all four explicit clears of `0x24` produces four ordered word writes there. Later direct effects or dependencies may change that location.

9. **Partial completion.** If a dependency fails to return after a store, that completed store remains an effect. Later accesses and the terminal marker are not reached; no rollback is established.

## Uncertainties

The following remain `documented_only`:

- Actual decoding and validity of the reconstructed straight-line control flow.
- Dependency identities, real signatures, ABI, register preservation, and extra-register result semantics.
- Valid addresses, alignment, endianness, device-memory behavior, and concurrent modification.
- Ownership and lifetime behavior, including dependency-induced changes.
- Whether the terminal marker corresponds to an actual runtime trap.
- Any semantic interpretation of recurring offsets or apparent record layouts.

**Inferred intent:** recurring scaled selectors and offsets suggest structured state manipulation. They do not establish object boundaries, field meanings, initialization success, or a safe public API.

## Evidence

Established static evidence consists of the supplied representation’s explicit widths, argument flows, access expressions, ordering, repeated reads, four absolute-address clears, and terminal invalid-instruction marker.

In particular, it places no dependency call between the \(T\)-derived store and its following word read. It also explicitly supplies scaled values and destination selectors to calls throughout the intermediate and final groups, including the three emphasized calls in the final byte sequence.

The representation reports invalid instruction data and overlapping global symbols. It is not independently authenticated executable behavior.

No implementation access, dependency inspection, runtime execution, memory trace, or runtime tests were available or performed. Acceptance examples are arithmetic consequences of the conditional static model, not observed outcomes.
