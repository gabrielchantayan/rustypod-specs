<!-- Trusted provenance: F-0a72fdf9ed2c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0a72fdf9ed2c

## Purpose

F-0a72fdf9ed2c performs an ordered sequence of indirect dependency calls, word publications, captured memory reads, and byte updates, then enters an unconditional repeating call-and-clear cycle.

The supplied static representation establishes no application-level purpose. Allocation, initialization, decoding, serialization, and cleanup are not established interpretations.

There is no visible normal return, completion result, error result, or conditional exit. This contract describes the represented path when each dependency returns normally and each memory access completes. Faults, dependency failures, and nonlocal transfers have unspecified consequences.

## Interface

### Explicit and implicit inputs

The represented interface has three 32-bit arguments. The first two have no visible direct use. The third is interpreted as unsigned and logically shifted right by seven before the first dependency call.

Additional execution inputs are:

- An incoming address-like register determining the first indirect-call destination.
- An incoming context-like base used for the initial word publication.
- A global storage location at absolute address `0x3C`.
- Memory selected by returned words, scaled indices, and selector bytes.
- An auxiliary register value used after a later call sequence.
- The targets, signatures, and behavior of all indirect dependencies.

The representation does not establish a complete binary calling convention. Unused explicit arguments do not prove the absence of implicit register or stack inputs.

### Widths, addresses, and notation

Words are 32 bits; bytes are eight bits. The initial dependency has a represented 64-bit result, separated into low and high 32-bit halves.

The following contract notation denotes captured values, not recovered program identifiers:

| Name | Meaning |
|---|---|
| \(S,H\) | Low and high halves of the initial result. |
| \(P\) | Primary sample. |
| \(Q\) | \((P \mathbin{\&} 0x3FFFFFFF)\gg5\). |
| \(E\) | Early byte-publication word. |
| \(G\) | Captured 32-bit conversion of the global value. |
| \(K\) | First indexed word snapshot, retained for later phases. |
| \(T\) | Separate indexed word snapshot. |
| \(D\) | \(8(K\gg8)\). |
| \(B,V\) | Later selector-publication word and direct-byte-publication word. |
| \(I_n\) | Distinct captured dependency results identified below. |

For compactness:

- \(A_4(x)=4x+0x14\).
- \(A_{16}(x)=16x+0x14\).
- A **selector publication through \(s\)** means an eight-bit read at \(s+1\), followed by an eight-bit write at the unsigned byte read plus `0x11`.

These computed addresses are not established object-relative offsets.

Unsigned shifts are logical. Unsigned 32-bit calculations retain the low 32 bits. Byte stores retain only the low eight bits. Signed-looking products and left shifts have unresolved overflow semantics; these are **documented_only**, not permission to assume unbounded arithmetic or universally defined wraparound.

Byte order, alignment requirements, atomicity, and memory-mapped attributes are unknown. A word access is not an established sequence of byte accesses.

### Outputs and ownership

Observable outputs are dependency invocations and memory accesses. There is no visible returned value.

No allocation, release, ownership transfer, resource closure, or preservation guarantee is established. A dependency argument is a value, not evidence of a borrowed buffer, owned allocation, or released resource.

## Behavior

### Inventory conventions

Every dependency reference below is document-local and opaque. Labels identify represented call sites, not resolved function identities. Ranges enumerate every site in order.

“Unspecified arguments” means no explicit arguments appear in the representation; it does not establish an argument-free binary interface. “Producer” identifies a represented returned word. A forwarded producer result is captured before the consuming call.

Each inventory row finishes with the next direct memory effect. Thus calls between apparently repeated writes remain observable ordering boundaries.

### 1. Initial result and context publication

F-A01 receives the third argument logically shifted right by seven. Its represented 64-bit result supplies \(S,H\).

The function writes \(S\) as a word at incoming context base plus `0x74` before proceeding.

| Ordered dependency interactions | Following direct effect |
|---|---|
| F-A02 receives \((H\ll3,H\ll3,S,S)\). F-A03 produces a word; F-A04 receives it. F-A05 produces another word; F-A06 receives it. F-A07–F-A09 have unspecified arguments. F-A10 produces \(I_1\). | Write \(S\) at \(4I_1+0x74\). |
| F-A11 receives \(4I_1\). F-A12–F-A14 have unspecified arguments. F-A15 produces \(P\). | Write \(Q\) at \(A_{16}(P)\). |

The two seed publications are separate accesses; their destinations are not established to be equal.

The first four-argument interaction uses a signed-looking left shift of the represented high half. Its overflowing cases remain unresolved.

### 2. Primary sample and early byte publications

The early selector component is \((P\mathbin{\&}0x3FFFFFFF)\gg6\).

| Ordered dependency interactions | Following direct effect |
|---|---|
| F-B01 receives \(4P\), immediately after the primary-sample word publication. F-B02–F-B04 have unspecified arguments. F-B05 produces \(I_2\). | Write \(Q\) at \(A_4(I_2)\). |
| F-B06 receives \(4I_2\). F-B07–F-B09 have unspecified arguments. F-B10 produces \(E\). | Through the early selector, publish the low byte of \(E\ll2\). |
| F-B11 receives the full 32-bit \(E\ll2\), not the byte just stored. F-B12–F-B14 have unspecified arguments. F-B15 produces \(I_3\). | Reread the early selector and publish the low byte of \((E\mathbin{\&}0x3FFFFFFF)\gg20\). |
| F-B16 receives \(I_3\ll2\). F-B17–F-B20 have unspecified arguments. | Capture \(G\) from the global at `0x3C`; write \(G\) at \(A_4(S\gg7)\). |

Both byte publications use the captured \(E\), but each obtains its destination from a fresh byte read. Their destinations can differ. Every selector-selected destination lies between `0x11` and `0x110`, inclusive.

The global conversion supplies 32 bits. The underlying global access width is not independently established.

### 3. Global repetition and indexed word phases

| Ordered dependency interactions | Following direct effect |
|---|---|
| F-C01 receives \(G\). F-C02–F-C04 have unspecified arguments. F-C05 produces \(I_4\). | Rewrite captured \(G\) at \(A_4(S\gg7)\), without an explicit global reread. |
| F-C06 receives \(I_4\ll2\). F-C07–F-C09 have unspecified arguments. F-C10 produces \(I_5\). | Write \(4I_5\) at \(A_{16}(I_5)\), then read a word from \(A_4(I_5)\) as \(K\). |
| F-D01 receives \((16I_5,4I_5,I_5\ll26,4I_5)\). F-D02 produces a word; F-D03 receives it. F-D04 produces another word; F-D05 receives it. F-D06–F-D08 have unspecified arguments. F-D09 produces \(I_6\). | Rewrite \(4I_5\) at \(A_{16}(I_5)\). |
| F-D10 receives \(4I_6\). F-D11–F-D13 have unspecified arguments. F-D14 produces \(I_7\). | Write \(4I_7\) at \(A_{16}(I_7)\), then read a word from \(A_4(I_7)\) as \(T\). |
| F-E01 receives \((4I_7,4I_7)\). F-E02 produces a word; F-E03 receives it. F-E04 produces another word; F-E05 receives it. F-E06–F-E08 have unspecified arguments. F-E09 produces \(I_8\). | Write \(T\gg8\) at \(A_4(I_7)\). |
| F-E10 receives \(4I_8\). F-E11–F-E13 have unspecified arguments. F-E14 produces \(I_9\). | Write \(4I_9\) at \(A_{16}(I_9)\). |
| F-F01 receives \(K\gg16\). F-F02–F-F04 have unspecified arguments. F-F05 produces \(I_{10}\). | Write \(4I_9\) at \(A_4(I_9)\). |
| F-F06 receives \(I_{10}\ll2\). F-F07–F-F09 have unspecified arguments. F-F10 produces \(I_{11}\). | Write \(4I_{11}\) at \(A_{16}(I_{11})\). |
| F-G01 receives \(16I_{11}\). F-G02 produces a word; F-G03 receives it. F-G04 produces another word; F-G05 receives that result and \(K\gg16\). F-G06–F-G08 have unspecified arguments. F-G09 produces \(I_{12}\). | Rewrite \(4I_{11}\) at \(A_{16}(I_{11})\). |
| F-G10 receives \(4I_{12}\). F-G11–F-G13 have unspecified arguments. F-G14 produces \(I_{13}\). | Write \(4I_{13}\) at \(A_{16}(I_{13})\). |
| F-H01 receives \((4I_{13},D)\). F-H02 produces a word; F-H03 receives it. F-H04 produces another word; F-H05 receives that result and \(K\gg16\). F-H06–F-H08 have unspecified arguments. F-H09 produces \(I_{14}\). | Write \(4I_{13}\) at \(D+0x14\). |
| F-H10 receives \(4I_{14}\). F-H11–F-H13 have unspecified arguments. F-H14 produces \(I_{15}\). | Write \(4I_{15}\) at \(A_{16}(I_{15})\). |

Both indexed snapshots are read after their preceding sixteenfold-location writes. Aliasing can therefore affect the captured values. Later uses of \(K\) and \(T\) are captured-value uses, not implicit memory rereads.

### 4. Packed-derived, auxiliary, and later word publications

The next four-argument call uses a retained 32-bit quantity initially formed by \(I_5\ll26\), then logically shifted right by seven three times. Intermediate narrowing must not be discarded when interpreting this argument. The initial signed-looking left shift remains unresolved for problematic operands.

| Ordered dependency interactions | Following direct effect |
|---|---|
| F-I01 receives \(D,D,K\gg16\), and the retained shift-derived quantity. F-I02–F-I04 have unspecified arguments. F-I05 produces \(I_{16}\). | Write \(D\) at \(D+0x14\). |
| F-I06 receives \(I_{16}\ll2\). F-I07–F-I10 have unspecified arguments. | Using the auxiliary register value, write its product with `0x01000000` at its product with `0x04000000`, plus `0x14`. |
| F-J01 receives the first auxiliary product. F-J02–F-J04 have unspecified arguments. F-J05 produces \(I_{17}\). | Write that same product at \(A_4(I_{17})\). |
| F-J06 receives \(4I_{17}\). F-J07–F-J09 have unspecified arguments. F-J10 produces \(I_{18}\). | Write \(4I_{18}\) at \(A_{16}(I_{18})\). |
| F-K01 receives \(4I_{18}\). F-K02 produces a word; F-K03 receives it. F-K04 produces another word; F-K05 receives it. F-K06–F-K08 have unspecified arguments. F-K09 produces \(I_{19}\). | Write \(4I_{18}\) at \(A_4(I_{19})\). |
| F-K10 receives \(4I_{19}\). F-K11–F-K13 have unspecified arguments. F-K14 produces \(I_{20}\). | Capture \(U=4I_{20}\); write \(U\) at \(A_{16}(I_{20})\). |
| F-L01 receives \(U\). F-L02 produces a word; F-L03 receives it. F-L04 produces another word; F-L05 receives it. F-L06–F-L08 have unspecified arguments. F-L09 produces \(I_{21}\). | Write \(K\gg24\) at \(A_4(I_{21})\). |
| F-L10 receives \(4I_{21}\). F-L11–F-L13 have unspecified arguments. F-L14 produces \(I_{22}\). | Write \(4I_{22}\) at \(A_{16}(I_{22})\). |
| F-M01 receives \((16I_{22},16I_{22},K\gg23)\). F-M02–F-M04 have unspecified arguments. F-M05 produces \(I_{23}\). | Write \(16I_{22}\) at \(A_{16}(I_{22})\). |
| F-M06 receives \(I_{23}\ll2\). F-M07–F-M09 have unspecified arguments. F-M10 produces \(B\). | Through the later selector, publish the low byte of \(B\ll2\). |

The later selector component is \((U\mathbin{\&}0x7FFFFFFF)\gg7\). The calculation of \(U\) includes a signed-looking multiplication before its unsigned use.

The auxiliary value is not an ordinary declared return in the representation. Its exact supplying dependency and register contract are **documented_only**.

### 5. Later byte publications and clearing

| Ordered dependency interactions | Following direct effect |
|---|---|
| F-N01 receives the full 32-bit \(B\ll2\) three times. F-N02 produces a word; F-N03 receives it. F-N04–F-N06 have unspecified arguments. F-N07 produces \(I_{24}\). | Reread the later selector; publish the low byte of \((B\mathbin{\&}0x3FFFFFFF)\gg20\). |
| F-N08 receives \(I_{24}\ll2\). F-N09–F-N11 have unspecified arguments. F-N12 produces \(V\). | Write the low byte of \(V\ll2\) directly at \((K\gg31)+0x1C\). Immediately afterward, reread the later selector and publish zero through it. |
| F-O01 receives \(K\gg31\). F-O02 has unspecified arguments. F-O03 receives \((V\mathbin{\&}0x3FFFFFFF)\gg6\). F-O04–F-O06 have unspecified arguments. F-O07 produces \(I_{25}\). | Reread the later selector and publish zero through it again. |

The direct byte destination is `0x1C` or `0x1D`. Its write precedes the immediately following selector read; aliasing can therefore affect that clear even without an intervening dependency.

Neither clear is guaranteed to revisit a previous publication destination.

### 6. Loop entry

F-O08 receives \(I_{25}\ll2\). F-O09–F-O11 have unspecified arguments. F-O12 returns a word whose low 30 bits, logically shifted right by twenty-five, form the initial loop-control value. This value is between zero and thirty-one.

The function then performs a selector-based zero publication using a different selector component:

\[
(V\ll2)\times0x400.
\]

The captured shifted word is reused; \(V\) is not reread. The multiplication is signed-looking, so overflowing address calculations remain **documented_only**.

F-O13 receives twice the initial loop-control value. F-O14 then returns the **unshifted first argument** for the first loop-opening call.

### 7. Repeating behavior and opening-target transition

The opening indirect-call destination differs between the first iteration and subsequent iterations:

- **F-P01a** identifies the first-iteration opening destination.
- **F-P01b** identifies the recurring opening destination.

A saved destination base is changed inside the loop and is not reset at its head. These destinations must not be collapsed into one assumed dependency identity.

Each iteration has this observable order:

1. F-P01a, on the first iteration, or F-P01b thereafter, receives the carried first argument and the current unsigned loop-control word shifted right by eight.
2. F-P02–F-P04 have unspecified arguments.
3. F-P05 returns an index.
4. The function freshly reads the selector at the loop-entry selector component plus one, then writes zero at the byte-selected destination.
5. F-P06 receives the returned index shifted left by two.
6. F-P07–F-P09 have unspecified arguments.
7. F-P10 returns a word.

For the next iteration, the F-P10 result supplies two separate derivations:

- The next opening call’s first argument is that result shifted left by **two**.
- The next loop-control word is the same unshifted result shifted left by **twenty-six**.

These signed-looking shifts retain their documented uncertainty. The first iteration is different: its first argument is the unshifted F-O14 result.

The first opening call’s second argument is necessarily zero. Later second arguments are not necessarily zero.

The selector component remains captured across iterations, but the selector byte is reread each time. No condition, break, cancellation check, or normal return is visible.

## Side effects

Direct effects include context-relative word storage, computed-address word reads and writes, a global read, selector-byte reads, byte publications, and repeated zero writes. Every listed dependency call is an observable ordering boundary.

There is no visible rollback. If a dependency or memory access fails, earlier effects are not visibly compensated.

Safe progress requires valid indirect targets and accessible memory for every executed access. No visible null, bounds, alignment, or permission checks precede these operations.

Dependencies may observe or alter globals and memory. Their ownership effects, register clobbers, synchronization, and nonlocal behavior are not supplied. No purity or global-preservation assumption is part of this contract.

## Dependencies

Only F-0a72fdf9ed2c has a supplied resolved opaque identity. F-A01 through F-P10, including F-P01a and F-P01b, are document-local references.

The inventory preserves all represented calls, their explicit argument order, producer/consumer forwarding, and boundaries around direct accesses. It does not establish complete signatures, distinctness across all destinations, error conventions, or memory ownership.

The initial result is represented as 64 bits. Other captured ordinary results are 32-bit values. The auxiliary-register observation requires an additional, unresolved calling-convention interpretation.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Initial argument boundary.** Third arguments zero through 127 supply zero to F-A01; 128 through 255 supply one. This does not establish whole-function equivalence.

2. **Initial ordering.** If F-A01 supplies \(S=0x12345678,H=1\), the context-relative word is written before F-A02 receives eight, eight, `0x12345678`, and `0x12345678`.

3. **Primary publication boundary.** For \(P=0x40\), \(Q=2\). The word at `0x414` is written before F-B01 receives `0x100`.

4. **Byte versus full-word argument.** For \(E=0x12345678\), the first early byte publication stores `0xE0`; F-B11 receives `0x48D159E0`. The later publication stores `0x23`. Equal destinations require equal freshly read selector values.

5. **Captured snapshot.** If a dependency changes the memory from which \(K\) was captured, subsequent explicit packed-derived calculations still use captured \(K\), not an implicit reread.

6. **Distinct shifts and ordered calls.** For \(K=0xFFFFFFFF\), the published \(K\gg24\) is 255, F-M01’s third argument is 511, and F-O01 receives one. F-O02 occurs before F-O03 receives the masked right-by-six quantity derived from \(V\).

7. **First versus recurring iteration.** If F-O14 returns three, F-P01a receives three as its first argument. If the first F-P10 returns one, F-P01b next receives four, with second argument `0x00040000`. These small operands avoid the problematic signed left-shift cases.

8. **Fresh clearing destination.** A changed selector byte can cause successive clears to target different addresses. Acceptance must not require one persistent cleared location.

9. **No finite completion.** If dependencies return normally and memory accesses succeed indefinitely, the represented operation continues rather than reporting success.

## Uncertainties

- **documented_only — Control flow:** The supplied material reports invalid instruction data and removed unreachable regions. Executable control-flow fidelity is unverified.
- **documented_only — Binary interface:** Implicit inputs, complete signatures, preserved registers, and auxiliary-result provenance remain unresolved.
- **documented_only — Arithmetic:** Signed-looking overflowing products and left shifts require machine-level clarification, including loop-carried derivations.
- **documented_only — Memory interpretation:** Computed low addresses may be literal mappings or recovery artifacts.
- **documented_only — Global representation:** Full width, volatility, and overlapping-symbol interpretation at `0x3C` are unknown.
- **documented_only — Memory behavior:** Byte order, concurrency, alignment, atomicity, and device-memory semantics are unavailable.
- **documented_only — Dependency outcomes:** Failure, nonlocal transfer, ownership, and external side effects are unspecified.
- **Inferred intent:** Repeated publications and clears suggest staged state manipulation; no domain-specific purpose is established.

## Evidence

This contract derives solely from the supplied static representation. Established evidence includes represented access widths, explicit dependency arguments, call ordering, captured-value reuse, fresh selector reads, separate loop-carried shifts, the first-versus-recurring opening destination transition, and the unconditional final cycle.

No runtime tests, dependency execution, executable inspection, or independent control-flow validation have been performed.
