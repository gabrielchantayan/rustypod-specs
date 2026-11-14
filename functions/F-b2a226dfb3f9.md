<!-- Trusted provenance: F-b2a226dfb3f9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b2a226dfb3f9

## Purpose

The supplied static representation describes a three-input routine that executes an ordered sequence of indirect calls, performs sixteen coprocessor-related transfers, writes one 16-bit value to a computed address, and reaches an invalid-instruction termination marker.

No application-level purpose is established. Initialization, decoding, hardware configuration, and resource management are not adopted as inferred intent.

The contract below preserves every represented call, including calls whose results are unused. Execution of any later operation is conditional on all preceding operations completing normally. The representation reports invalid instruction data and truncated control flow; it does not establish that the entire displayed path is executable.

## Interface

### Explicit inputs

| Input | Represented width | Established use |
|---|---:|---|
| First input | 32 bits | First argument of call 1. |
| Second input | Signed 32 bits | Second argument of call 1; also shifted left by 10 for its fourth argument. |
| Third input | 32 bits | Third argument of call 1. |

No later direct use of these original inputs is visible. Effects propagated through callees or machine state remain unknown.

### Implicit inputs and prerequisites

Execution additionally depends on:

- The incoming link-register value, which determines the first indirect target.
- Eight represented entry-state selector values, identified here as \(E_1,E_3,E_5,E_6,E_7,E_9,E_{10},E_{15}\). Each is represented as a 32-bit value.
- Four implicit register-derived words consumed at specific calls.
- Executable targets, suitable machine state, and any memory or architectural state required by callees.
- A writable final store address if execution is to complete that store.

There is no visible validation of arguments, call targets, privileges, alignment, or writable memory.

### Result and arithmetic conventions

No ordinary return value or successful-return path is established.

For the operation inventory:

- \(w_n\) means the retained 32-bit result of call \(n\).
- \(q_n\) means the retained 64-bit result of call \(n\).
- \(L_n\) and \(H_n\) mean the low and high 32-bit halves of \(q_n\).
- \(\operatorname{sh}_s(x)\) means a left shift by \(s\), retaining the low 32 bits.
- \(\operatorname{asr}_s(x)\) means a signed, arithmetic right shift of a 32-bit value.
- “None shown” means no explicit arguments are represented; it does not exclude implicit inputs.
- “Unused” means no explicit result is consumed; it does not mean the callee returns no value.

Fixed-width left-shift conclusions are conditional on machine bit-vector semantics. The represented signed expressions are not asserted to have portable language-level semantics.

## Behavior

### Ordering and target identities

The displayed path has **130 call invocations: 128 indirect calls and two directly identified dependency calls**. It contains no visible conditional branch, loop, retry, or error test.

Opaque IDs beginning **F-U** identify unresolved target expressions. They are not assertions of recovered function boundaries or signatures. Repeated IDs below preserve established equalities between fixed numeric targets:

- Calls 34 and 47.
- Calls 42 and 89.
- Calls 60 and 98.
- Calls 97 and 125.
- Calls 106 and 116.
- Calls 122 and 127.

Other target aliasing is not excluded. In particular, call 1’s incoming-state-derived target could coincide with another target.

Call 1 targets the incoming link-register value plus 0xE90. All subsequent indirect targets are represented using fixed numeric address expressions or fixed symbol anchors plus offsets; none visibly depends on the three explicit inputs.

Transfers T1–T16 are defined under Side effects. Their placement in the inventory is part of the required order.

### Calls 1–43

| Call | Dependency | Explicit arguments, in order | Retained result / following transfer |
|---:|---|---|---|
| 1 | F-U001 | First input; second input; third input; \(\operatorname{sh}_{10}(\text{second input})\) | Unused |
| 2 | F-U002 | None shown | Unused |
| 3 | F-U003 | None shown | Unused |
| 4 | F-U004 | None shown | \(w_4\); then T1 |
| 5 | F-U005 | None shown | Unused |
| 6 | F-U006 | None shown | Unused |
| 7 | F-U007 | None shown | Unused |
| 8 | F-U008 | None shown | Unused |
| 9 | F-U009 | None shown | Unused |
| 10 | F-U010 | None shown | Unused |
| 11 | F-U011 | None shown | Unused |
| 12 | F-U012 | None shown | \(q_{12}\) |
| 13 | F-U013 | \(L_{12}\) | \(w_{13}\) |
| 14 | F-U014 | \(w_{13}\) | \(w_{14}\) |
| 15 | F-U015 | \(w_{14}\) | \(w_{15}\); then T2 |
| 16 | F-U016 | None shown | \(q_{16}\) |
| 17 | F-U017 | \(L_{16}\); \(\operatorname{sh}_{17}(H_{16})\) | Unused |
| 18 | F-U018 | None shown | Unused |
| 19 | F-U019 | None shown | Unused |
| 20 | F-U020 | \(\operatorname{sh}_{28}(H_{12})\) | \(w_{20}\) |
| 21 | F-U021 | \(w_{20}\) | \(w_{21}\) |
| 22 | F-U022 | \(w_{21}\) | Unused |
| 23 | F-U023 | None shown | \(w_{23}\) |
| 24 | F-U024 | \(w_{23}\); 0; 0 | Unused; then T3 |
| 25 | F-U025 | 0; \(\operatorname{sh}_{7}(I_A)\); 0; 0 | Unused |
| 26 | F-U026 | None shown | Unused |
| 27 | F-U027 | None shown | Unused |
| 28 | F-U028 | None shown | Unused |
| 29 | F-U029 | 0 | \(w_{29}\) |
| 30 | F-U030 | \(w_{29}\); 0 | Unused |
| 31 | F-U031 | None shown | Unused |
| 32 | F-U032 | None shown | Unused |
| 33 | F-U033 | None shown | Unused |
| 34 | F-U034 | None shown | \(w_{34}\) |
| 35 | F-U035 | \(w_{34}\) | \(w_{35}\) |
| 36 | F-U036 | \(w_{35}\); \(\operatorname{sh}_{20}(w_{35})\) | Unused |
| 37 | F-U037 | None shown | \(q_{37}\) |
| 38 | F-U038 | \(L_{37}\) | Unused |
| 39 | F-U039 | \(\operatorname{sh}_{14}(L_{37})\); \(I_B\); \(\operatorname{sh}_{23}(H_{37})\) | Unused |
| 40 | F-U040 | \(\operatorname{sh}_{23}(H_{37})\) | Unused |
| 41 | F-U041 | None shown | Unused |
| 42 | F-U042 | 0; 0; \(\operatorname{sh}_{4}(L_{37}\mathbin{\&}\text{0x3FFFF})\); 0 | Unused |
| 43 | F-U043 | None shown | \(w_{43}\); then T4 |

Here \(I_A\) is the implicit register-derived word exposed after call 24 and consumed at call 25. \(I_B\) is a separate implicit register-derived word exposed after call 38 and consumed, unchanged, as call 39’s second argument. Neither is established as an ordinary return value.

The argument at call 20 is represented as successive shifts by 19 and 9. Call 39’s and call 40’s high-half argument is represented as successive shifts by 9 and 14.

### Calls 44–88

| Call | Dependency | Explicit arguments, in order | Retained result / following transfer |
|---:|---|---|---|
| 44 | F-U044 | None shown | Unused |
| 45 | F-U045 | None shown | Unused |
| 46 | F-U046 | None shown | Unused |
| 47 | F-U034 | None shown | Unused |
| 48 | F-U048 | None shown | \(q_{48}\); then T5 |
| 49 | F-U049 | \(L_{48}\); \(\operatorname{sh}_{20}(H_{48})\) | \(q_{49}\) |
| 50 | F-U050 | \(L_{49}\); \(H_{49}\); \(\operatorname{sh}_{12}(H_{49})\) | Unused |
| 51 | F-U051 | None shown | Unused |
| 52 | F-U052 | None shown | Unused |
| 53 | F-U053 | None shown | Unused |
| 54 | F-U054 | None shown | \(w_{54}\) |
| 55 | F-0e02e4c13579 | \(w_{54}\) | Unused |
| 56 | F-U056 | 0 | Unused |
| 57 | F-U057 | None shown | Unused |
| 58 | F-U058 | None shown | Unused |
| 59 | F-U059 | None shown | Unused |
| 60 | F-U060 | None shown | Unused |
| 61 | F-U061 | None shown | \(q_{61}\) |
| 62 | F-U062 | \(\operatorname{sh}_{4}(L_{61})\); \(H_{61}\); \(\operatorname{sh}_{28}(w_{54})\) | \(q_{62}\) |
| 63 | F-U063 | \(L_{62}\); \(H_{62}\); 0 | \(w_{63}\) |
| 64 | F-U064 | \(w_{63}\) | Unused |
| 65 | F-U065 | None shown | Unused |
| 66 | F-U066 | 0; 0 | \(w_{66}\) |
| 67 | F-U067 | \(w_{66}\) | Unused |
| 68 | F-U068 | 0; \(I_C\); 0; 0 | \(w_{68}\); then T6 |
| 69 | F-U069 | \(w_{68}\) | Unused |
| 70 | F-U070 | None shown | \(w_{70}\) |
| 71 | F-U071 | \(w_{70}\) | \(w_{71}\) |
| 72 | F-U072 | \(w_{71}\) | Unused |
| 73 | F-U073 | None shown | Unused |
| 74 | F-U074 | None shown | Unused |
| 75 | F-U075 | None shown | Unused |
| 76 | F-U076 | None shown | \(w_{76}\) |
| 77 | F-U077 | \(w_{76}\) | Unused |
| 78 | F-U078 | 0 | Unused |
| 79 | F-U079 | None shown | Unused |
| 80 | F-U080 | None shown | Unused |
| 81 | F-U081 | 0; \(I_D\); 0 | Unused |
| 82 | F-U082 | None shown | Unused |
| 83 | F-U083 | None shown | Unused |
| 84 | F-U084 | None shown | \(w_{84}\) |
| 85 | F-U085 | \(w_{84}\); 0; \(\operatorname{sh}_{5}(w_{84})\) | Unused |
| 86 | F-U086 | None shown | \(w_{86}\) |
| 87 | F-U087 | \(w_{86}\) | Unused |
| 88 | F-3de949054bdd | None shown | Unused |

\(I_C\) is the implicit register-derived word exposed after call 67 and consumed at call 68. \(I_D\) is another such word exposed after call 80 and consumed at call 81. In particular, call 81 precedes the two argumentless calls and call 84 that obtains the word later shifted left by 5.

The retained contribution from call 54 is first shifted by 11, then by 17 for call 62. Its separate later shift by 31 produces call 68’s zero first argument under 32-bit semantics. The other displayed zeros in this portion follow from explicitly retained zero-valued dataflow, not from assumed callee preservation of globals.

### Calls 89–130

For compactness, let:

\[
P=\operatorname{sh}_{27}(w_{98}),\qquad
R=\operatorname{sh}_{4}(w_{113}).
\]

These are descriptions of retained values, not additional operations with external effects.

| Call | Dependency | Explicit arguments, in order | Retained result / following transfer |
|---:|---|---|---|
| 89 | F-U042 | None shown | \(w_{89}\) |
| 90 | F-U090 | \(w_{89}\) | Unused |
| 91 | F-U091 | None shown | \(w_{91}\) |
| 92 | F-U092 | \(w_{91}\) | Unused |
| 93 | F-U093 | 0; \(\operatorname{asr}_{5}(\operatorname{sh}_{5}(w_{84}))\) | \(w_{93}\) |
| 94 | F-U094 | \(w_{93}\) | Unused |
| 95 | F-U095 | None shown | \(q_{95}\); then T7 |
| 96 | F-U096 | \(\operatorname{sh}_{1}(H_{95})\) | Unused |
| 97 | F-U097 | None shown | \(w_{97}\); then T8, T9 |
| 98 | F-U060 | None shown | \(w_{98}\) |
| 99 | F-U099 | 0; 0; \(P\); 0 | Unused |
| 100 | F-U100 | None shown | Unused |
| 101 | F-U101 | None shown | Unused |
| 102 | F-U102 | None shown | Unused |
| 103 | F-U103 | None shown | Unused |
| 104 | F-U104 | None shown | Unused |
| 105 | F-U105 | None shown | \(w_{105}\); then T10 |
| 106 | F-U106 | None shown | Unused |
| 107 | F-U107 | None shown | Unused |
| 108 | F-U108 | None shown | Unused; then T11 |
| 109 | F-U109 | 0; 0 | Unused |
| 110 | F-U110 | None shown | Unused |
| 111 | F-U111 | None shown | \(w_{111}\); then T12 |
| 112 | F-U112 | None shown | Unused |
| 113 | F-U113 | 0; 0; 0 | \(w_{113}\); then T13 |
| 114 | F-U114 | \(\operatorname{sh}_{1}(w_{113})\); 0; \(R\); 0 | Unused |
| 115 | F-U115 | None shown | Unused |
| 116 | F-U106 | None shown | Unused; then T14, T15 |
| 117 | F-U117 | 0 | \(w_{117}\); then T16 |
| 118 | F-U118 | \(\operatorname{sh}_{8}(R)\); \(\operatorname{sh}_{12}(R)\); \(\operatorname{sh}_{9}(w_{117})\) | Unused |
| 119 | F-U119 | None shown | \(q_{119}\) |
| 120 | F-U120 | \(L_{119}\); \(H_{119}\); \(\operatorname{sh}_{1}(L_{119})\) | Unused |
| 121 | F-U121 | 0 | \(w_{121}\) |
| 122 | F-U122 | \(w_{121}\) | \(w_{122}\) |
| 123 | F-U123 | \(w_{122}\) | Unused |
| 124 | F-U124 | None shown | Unused |
| 125 | F-U097 | None shown | \(q_{125}\) |
| 126 | F-U126 | \(L_{125}\); \(H_{125}\); \(\operatorname{sh}_{11}(H_{125})\) | \(q_{126}\) |
| 127 | F-U122 | \(L_{126}\) | Unused |
| 128 | F-U128 | None shown | \(w_{128}\) |
| 129 | F-U129 | \(w_{128}\); 0 | Unused |
| 130 | F-U130 | None shown | Unused |

The same target is therefore represented with different argument lists at calls 42 and 89, and with different retained result widths at calls 97 and 125. Neither observation establishes a conventional callee signature.

The source relationships reduced to zero above include:

- \(\operatorname{sh}_{24}(P)=0\), used by T11 and call 109.
- \(P\mathbin{\&}\text{0x7FFFFF}=0\), contributing call 114’s second argument.
- \(R\mathbin{\&}3=0\), whose subsequent shifts supply call 121’s argument and call 129’s second argument.

### Final store and termination

The address-producing result is \(q_{126}\). Its unsigned bits 41–63 are first extracted into a 32-bit value. That value is nonnegative even when interpreted as signed, so the subsequent right shift by 11 selects the original result’s top twelve bits.

Let \(k\) be the unsigned value of bits 52–63 of \(q_{126}\). The final address is:

\[
A=k+\left\lfloor k/4\right\rfloor,\qquad
0\leq k\leq4095,\qquad 0\leq A\leq5118.
\]

No object base or buffer pointer is added. The maximum address is 0x13FE.

Calls 127–130 occur before the store. After they complete, the routine requests a **16-bit write of 0x00CD at address \(A\)**. The displayed path then reaches the invalid-instruction termination marker, not an ordinary return.

## Side effects

### Exact coprocessor-related operands

Each transfer has the ordered operand structure:

**first numeric field; second numeric field; third numeric field; data word; first entry-state field; second entry-state field.**

The table preserves all visible numeric fields and entry-state identities without assigning architectural meanings.

| Transfer | Position | Numeric fields | Data word | Entry-state fields |
|---|---|---|---|---|
| T1 | After call 4 | 0; 4; 7 | \(w_4\) | \(E_1;E_{15}\) |
| T2 | After call 15 | 0; 1; 7 | \(w_{15}\) | \(E_{10};E_{15}\) |
| T3 | After call 24 | 0; 6; 7 | 0 | \(E_6;E_{15}\) |
| T4 | After call 43 | 0; 5; 7 | \(w_{43}\) | \(E_{15};E_{15}\) |
| T5 | After call 48 | 0; 4; 7 | \(L_{48}\) | \(E_7;E_{15}\) |
| T6 | After call 68 | 0; 6; 7 | \(w_{68}\) | \(E_3;E_{15}\) |
| T7 | After call 95 | 0; 6; 7 | \(L_{95}\) | \(E_3;E_{15}\) |
| T8 | After call 97 | 0; 3; 7 | \(w_{97}\) | \(E_6;E_{15}\) |
| T9 | Immediately after T8 | 0; 3; 7 | \(w_{97}\) | \(E_{15};E_{15}\) |
| T10 | After call 105 | 0; 1; 7 | \(w_{105}\) | \(E_1;E_{15}\) |
| T11 | After call 108 | 0; 5; 7 | \(\operatorname{sh}_{24}(P)=0\) | \(E_5;E_{15}\) |
| T12 | After call 111 | 0; 6; 7 | \(w_{111}\) | \(E_9;E_{15}\) |
| T13 | After call 113 | 0; 5; 7 | \(\operatorname{sh}_{1}(w_{113})\) | \(E_{10};E_{15}\) |
| T14 | After call 116 | 0; 7; 7 | 0 | \(E_9;E_{15}\) |
| T15 | Immediately after T14 | 0; 0; 7 | 0 | \(E_{15};E_{15}\) |
| T16 | After call 117 | 0; 2; 7 | \(\operatorname{sh}_{8}(R)\) | \(E_9;E_{15}\) |

The entry-state notation preserves the representation’s dataflow; it does not assert that callees preserve the corresponding architectural registers or globals.

These transfers cannot be treated as ordinary memory writes, logging, or no-ops. Even equal data words do not justify merging transfers with distinct operands.

### Memory, ownership, and failures

The final store is the only explicit ordinary data-memory access shown. No explicit ordinary data-pointer read is visible. Calls and architectural operations may nevertheless access memory or devices.

No allocation, release, ownership transfer, reference-count change, or buffer-lifetime contract is established. A word passed to a dependency is not thereby established to be a pointer or resource handle.

Store endianness, alignment handling, and address protection are unknown. Address zero and odd addresses are possible. The write spans two bytes if the target implements an ordinary byte-addressed 16-bit store, but that architectural interpretation remains conditional.

No rollback or recovery path is visible. A call that does not return, a transfer fault, or a store fault can prevent later operations. Previously completed effects need not be undone.

## Dependencies

**F-0e02e4c13579** is called once, at call 55, with \(w_{54}\). Its result is not visibly consumed.

**F-3de949054bdd** is called once, at call 88, with no explicit arguments. Its result is not visibly consumed.

Every **F-U** dependency in the inventory is **documented_only** as to identity, complete signature, effects, and termination behavior. The inventory establishes invocation order, visible arguments, retained results, and the listed target equalities—not independent callee contracts.

No dependency is assumed to preserve globals, preserve undocumented registers, own or release memory, or return normally. Unknown effects are not grounds for omitting any invocation.

## Acceptance examples

These are conditional static expectations, not executed tests.

1. **Entry arguments.** If the second input is 1, call 1’s fourth argument is 0x00000400. If it is 0x00400000, that argument is zero under 32-bit shift semantics.

2. **Retained high-half bits.** If \(H_{12}=1\), call 20 receives 0x10000000. If \(H_{12}=\text{0x10}\), it receives zero.

3. **Implicit argument preservation.** If \(L_{37}=1\), \(H_{37}=1\), and \(I_B=\text{0x12345678}\), call 39 receives 0x00004000, 0x12345678, and 0x00800000, in that order.

4. **Separate implicit input.** If \(I_D=\text{0x89ABCDEF}\), call 81 receives zero, 0x89ABCDEF, and zero. Calls 82 and 83 still precede call 84.

5. **Masked packing.** If \(L_{37}=\text{0xFFFFFFFF}\), call 42’s third argument is 0x003FFFF0. Its other arguments remain zero.

6. **Signed reconstruction.** If \(w_{84}=\text{0x04000000}\), the retained five-bit left shift is 0x80000000. With arithmetic right shift, call 93’s second argument is 0xFC000000, not 0x04000000.

7. **Distinct transfers.** After call 97 returns normally, T8 precedes T9. Both use \(w_{97}\), but their first entry-state operands are \(E_6\) and \(E_{15}\), respectively; neither transfer is omitted.

8. **Forced late zeros.** For every \(w_{98}\), T11’s data and call 109’s first argument are zero. For every \(w_{113}\), the low-two-bit-derived arguments at calls 121 and 129 are zero.

9. **Address boundaries.** Top twelve bits of \(q_{126}\) equal to 0, 3, 4, or 4095 produce addresses 0, 3, 5, or 5118. Address 4 is not produced. Lower 52 bits do not affect the address.

10. **Store order and interruption.** Even after \(q_{126}\) is available, calls 127–130 precede the store. If any does not return, no final store is required by the displayed path. If the store completes, its represented width is 16 bits and its value is 0x00CD.

## Uncertainties

The following remain **documented_only**:

- Accuracy and reachability of the supplied instruction interpretation, particularly around invalid instruction data.
- Whether the terminal marker represents a deliberate trap, corrupted code, embedded data, or interpretation failure.
- Actual ABI, privilege requirements, target validity, and complete implicit machine-state interface.
- Unresolved target identities, symbol-anchor addresses, and additional target aliasing.
- Callee effects, ownership policies, and whether represented 64-bit results are scalar returns or grouped register results.
- The origins and meanings of \(I_A,I_B,I_C,I_D\).
- Architectural meanings and effects of the exact transfer operands.
- Actual signed-shift semantics where the representation alone is insufficient.
- Address conversion, mapping, endianness, alignment, and writability.
- Application-level intent and any behavior beyond the truncated path.

## Evidence

**Established static evidence:** three explicit inputs; the complete ordered inventory of 130 calls; six repeated fixed-target relationships; four specifically consumed implicit register-derived words; two directly identified dependencies; sixteen transfers with the operand combinations above; one final 16-bit store of 0x00CD; and an invalid-instruction termination marker.

**Conditional arithmetic deductions:** combined shifts, forced-zero arguments, discarded-bit relationships, signed reconstruction, and the final address range follow from the represented expressions under the stated width and shift assumptions.

**Inferred intent:** none adopted.

**Runtime evidence:** none. No runtime tests have been performed.
