<!-- Trusted provenance: F-c04fd6ade413; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c04fd6ade413

## Purpose

F-c04fd6ade413 performs ordered indirect dependency invocations, computed-address word stores, and byte-selected byte stores. If the represented path completes its initialization, it enters an unconditional loop containing further dependency invocations and byte-selected writes. No normal return is represented.

No application-level purpose is established. The arithmetic and memory operations do not, by themselves, establish encoding, device control, buffer management, or another domain-specific function.

This contract describes the supplied static representation, conditional on its recovered control flow being accurate and preceding operations completing. Warnings about invalid instruction data and removed unreachable regions limit confidence in the completeness of that representation.

## Interface

### Explicit inputs

| Input | Represented width | Established use |
|---|---:|---|
| First argument | 32 bits, uninterpreted | Passed to F-u001. |
| Second argument | 32 bits, uninterpreted | No explicit use in the represented body. |
| Third argument | Unsigned 32 bits | Supplies logical right shifts used in word data, dependency arguments, and a byte-store address. |

The absence of an explicit use of the second argument does not establish that dependencies ignore its incoming machine-level location.

### Implicit inputs and preconditions

Execution additionally requires:

- An incoming address-like base for the first word store.
- An incoming 32-bit value written by that store.
- An incoming control value from which the first dependency target is derived.
- A secondary 32-bit register result available after F-u005.
- Accessible memory at each computed read and write address.
- Valid executable targets and a compatible calling convention for every indirect invocation.

The representation does not establish how callers initialize this implicit state. It supplies no bounds, allocation sizes, or memory-region descriptors.

### Output and termination

There is no represented return value and no normal exit. Once the loop is reached, its back edge is unconditional.

This does not guarantee infinite execution in a real environment. A dependency can fail to return, an access can fault, or execution can depart from the recovered representation. No recovery behavior is established.

### Widths and layout

Word stores write four bytes; selector loads and byte stores access one byte each. Selector bytes are unsigned values from 0 through 255.

The contract uses these arithmetic conventions:

- **Word reduction** means retaining the low 32 bits.
- Logical right shifts of unsigned values introduce zero bits.
- A byte store retains only the low eight bits of its supplied value.
- Scaled address formulas describe the represented arithmetic; they do not independently establish address width or wrapping.

Where multiplication or left shift is described using low-32-bit results, that is the nominal machine-word interpretation. It is not a guarantee that signed overflow or signed left shift is defined in a source language.

Byte order, alignment requirements, address truncation, and the interpretation of negative signed intermediates as addresses remain **documented_only** unknowns. A four-byte store establishes its width and starting address, not its byte order.

## Behavior

### Invocation notation

F-u001 through F-u089 identify dependency invocation sites, not necessarily distinct functions.

A range described as a **five-call group** invokes its five sites in numerical order. Only the first site has the explicit arguments stated here; the other four have no explicit arguments displayed. Unless otherwise stated, the group’s result is the ordinary result of its fifth invocation.

“No explicit arguments displayed” does not establish a parameterless machine-level interface. Dependencies may consume additional register or stack state.

All stated memory operations occur in the described order relative to these invocations. A dependency between two accesses may observe or change memory.

### Initial word production

The first observable memory effect is a four-byte store of the implicit incoming word at the implicit incoming base plus 20.

The subsequent word-producing stages are:

| Dependency group | Explicit argument to first site | Result and following memory effect |
|---|---|---|
| F-u001–F-u005 | First argument | Use the secondary register result after F-u005, rather than an ordinary return value. |
| F-u006–F-u010 | Seed word, defined below | Treat the final result as the first index; store the seed word at four times that index plus 20. |
| F-u011–F-u015 | Four times the first index | Treat the final result as the second index; store four times that index at sixteen times that index plus 20. |

For the secondary register result after F-u005:

- The **seed word** is its product with \(2^{24}\), reduced to 32 bits.
- The seed-store address is its product with \(2^{26}\), plus 20.

The seed store occurs before F-u006. The third argument logically shifted right by 8 is also retained for later use.

The word produced from the second index is the **first scaled word**. Its nominal value is four times that index, reduced to 32 bits.

### Forwarding chains and further word stores

After the first scaled word is stored, the following ordered stages occur:

| Invocations | Explicit dataflow | Following memory effect |
|---|---|---|
| F-u016, F-u017, F-u018, F-u019 | F-u016 receives the first scaled word; F-u018 receives F-u017’s result. Retain F-u019’s result. | None directly. |
| F-u020–F-u024 | F-u020 receives F-u019’s result. The final result is the third index. | Store the first scaled word at four times the third index plus 20. |
| F-u025–F-u029 | F-u025 receives four times the third index. The final result is the fourth index. | Store four times the fourth index at sixteen times that index plus 20. |
| F-u030, F-u031, F-u032, F-u033 | F-u030 receives the word just produced; F-u032 receives F-u031’s result. Retain F-u033’s result. | None directly. |
| F-u034–F-u038 | F-u034 receives F-u033’s result. The final result is the fifth index. | Store the third argument logically shifted right by 8 at four times the fifth index plus 20. |
| F-u039–F-u043 | F-u039 receives four times the fifth index. The final result is the sixth index. | Store four times the sixth index at sixteen times that index plus 20. |

F-u017, F-u019, F-u031, and F-u033 have no explicit arguments displayed.

The **second scaled word** is four times the fourth index, reduced to 32 bits. It remains relevant after the later word stores.

The **retained base** is sixteen times the sixth index, under the applicable word/address arithmetic. Thus the last store in the table writes four times the sixth index at retained base plus 20.

The **selector base** is:

\[
S=(\text{second scaled word}\mathbin{\&}0x7fffffff)\mathbin{\gg}7.
\]

Its range is 0 through \(0x00ffffff\). The selector byte used in several subsequent operations is read at \(S+1\).

### Retained-base overwrite and first byte source

F-u044–F-u048 form a five-call group. F-u044 receives three explicit arguments, in order:

1. Retained base.
2. Retained base again.
3. Third argument logically shifted right by 7.

After the group returns, a four-byte store writes the retained base itself at retained base plus 20. This address is independent of F-u048’s result and overwrites the location previously used for four times the sixth index, unless another effect has changed the underlying memory interpretation.

F-u049–F-u053 form the next five-call group. F-u049 receives F-u048’s result shifted left by 2. The unsigned 32-bit result of F-u053 is the **first byte source**.

### Byte-selected stores

A **byte-selected store from an address** has two ordered memory effects:

- Read one unsigned byte from that address.
- Write one byte at the loaded byte’s numeric value plus 17.

The selected destination therefore lies numerically between 17 and 272 inclusive. The selector is an eight-bit integer, not a pointer-sized value loaded from memory.

Every occurrence performs a fresh read. Neither the selector value nor its selected destination is cached across occurrences.

The first such operation reads at \(S+1\) and writes the low byte of the first byte source shifted left by 2.

F-u054 then receives three identical arguments: the first byte source shifted left by 2, reduced to 32 bits. F-u055 is invoked with no explicit argument displayed. Its result is supplied to F-u056, beginning the five-call group F-u056–F-u060.

After F-u060 returns, another byte-selected store reads at \(S+1\). Its stored byte is:

\[
\operatorname{low8}
\left((\text{first byte source}\mathbin{\&}0x3fffffff)\mathbin{\gg}20\right).
\]

This selects original source bits 20 through 27. The intermediate expression can contain ten significant bits, but the store discards all except its low eight.

F-u061–F-u065 form a five-call group. F-u061 receives F-u060’s result shifted left by 2. The unsigned 32-bit result of F-u065 is the **second byte source**.

### Input-derived byte store and preparation for the loop

Let the **retained byte word** be the second byte source shifted left by 2, reduced to 32 bits.

A byte store writes its low byte at:

\[
(\text{third argument}\mathbin{\gg}15)+28.
\]

The representation forms the shift through the previously retained third-argument-right-shift-by-8 value, followed by another logical right shift of 7. The resulting destination ranges from 28 through 131099 inclusive.

Immediately afterward, a byte-selected store reads at \(S+1\) and writes zero to the selected destination.

The remaining pre-loop sequence is:

| Invocations | Explicit arguments and result use | Following memory effect |
|---|---|---|
| F-u066, F-u067 | F-u066 receives the third argument logically shifted right by 15. F-u067 has no explicit argument displayed; its result is not explicitly consumed. | None directly. |
| F-u068–F-u072 | F-u068 receives \((\text{second byte source}\mathbin{\&}0x3fffffff)\mathbin{\gg}6\). | Read a fresh selector at \(S+1\), then write zero to the selected destination. |
| F-u073–F-u077 | F-u073 receives F-u072’s result shifted left by 2. | Derive the initial loop field and perform the fixed-address selector operation described below. |
| F-u078, F-u079 | F-u078 receives twice the initial loop field. F-u079 has no explicit argument displayed. | F-u079’s result becomes the initial loop-carried ordinary value. |

The **initial loop field** is:

\[
(\text{F-u077 result}\mathbin{\&}0x3fffffff)\mathbin{\gg}25.
\]

It ranges from 0 through 31.

The **loop selector address** is the retained byte word multiplied by 1024, plus 1, subject to the target’s arithmetic. After F-u077 and before F-u078, the function reads one selector byte at this address and writes zero at that byte’s numeric value plus 17.

The retained byte word is not replaced by later loop results. Consequently, this loop selector address remains the same represented address throughout the loop.

### Unconditional loop

Each cycle has two five-call groups separated by a byte-selected zero store:

- F-u080–F-u084: F-u080 receives the current loop-carried ordinary value and the current loop field logically shifted right by 8.
- After F-u084 returns, read one selector byte from the fixed loop selector address and write zero at the loaded byte’s numeric value plus 17.
- F-u085–F-u089: F-u085 receives F-u084’s result shifted left by 2.

After F-u089 returns, its result supplies two next-cycle values:

- The next ordinary value is that result shifted left by 2, with nominal 32-bit reduction.
- The next loop field is that result shifted left by 26, with nominal 32-bit reduction.

Neither update changes the retained byte word or the loop selector address. The loop repeats unconditionally.

For the first cycle, the second explicit argument to F-u080 is zero because the initial loop field is at most 31.

For later cycles, under the nominal 32-bit interpretation, only the low six bits of the preceding F-u089 result survive in the loop field. They occupy bits 26 through 31; the right shift by 8 presents them in bits 18 through 23.

### Loop-entry versus back-edge target selection

F-u080 denotes one represented invocation site, but its indirect target changes after the first cycle:

| Occurrence | Represented target basis |
|---|---|
| First invocation | The control value established immediately before F-u079, plus the displayed target offset. |
| Every later invocation | The different control value established immediately before F-u089, plus that same offset. |

The two displayed control constants differ by 62, so the corresponding numeric target expressions differ by 62 under ordinary address arithmetic. This is a target-selection transition, not merely a change in F-u080’s arguments.

The back-edge control value is re-established before each F-u089 invocation. The representation does not subsequently restore the entry control value. F-u081–F-u089 continue to use their respective displayed target expressions.

Distinct target expressions do not prove distinct underlying implementations. Target mapping, aliasing, and executable validity remain unknown.

## Side effects

### Explicit memory effects

Strictly before loop entry, the represented path contains:

| Effect | Count |
|---|---:|
| Four-byte stores | 9 |
| Byte stores whose supplied value is explicitly zero | 3 |
| Other byte stores | 3 |
| One-byte selector reads | 5 |

The three “other” byte stores may themselves write zero for particular inputs or results; the classification concerns their expressions, not guaranteed nonzero values.

Four pre-loop selector reads use \(S+1\). The fifth uses the fixed loop selector address. Thus there are six total byte stores before the loop: five selector-mediated stores and one input-derived-address store.

Each completed loop cycle adds one selector-byte read and one explicit zero-byte store. Dependency-internal memory effects are excluded from all counts.

### Aliasing and observation order

No calculated addresses are established as disjoint.

A word store can overlap a selector byte or a selected destination. A selected byte destination can equal the address from which its selector was read. Dependencies can change selector contents between reads.

In particular, a fixed loop selector address does not imply a fixed selected destination. Each cycle rereads the byte at that address; dependencies or earlier writes may have changed it.

The retained-base overwrite is a distinct observable write separated from the earlier write by dependency invocations. Intermediate writes must not be eliminated merely because a later write targets the same address.

### Ownership, state, and errors

No explicit allocation, deallocation, ownership transfer, reference-count change, or resource release is established in the represented body.

Dependencies may perform such operations. Their effects on globals, memory lifetime, and retained machine state are unknown. The represented retention of values is not an independently verified register-preservation guarantee.

There is no displayed validation, bounds check, error return, rollback, or exception handler. Failures of memory accesses or indirect invocations have no established recovery contract.

## Dependencies

F-u001–F-u089 are opaque invocation-site IDs. The loop repeatedly uses F-u080–F-u089, with the F-u080 target transition specified above.

The first dependency target derives from incoming control state. Later targets derive from displayed control values or address-like anchors and offsets. Neither executable validity nor actual callee identity is established.

The dependency contract establishes only invocation order, displayed arguments, consumed ordinary results, the secondary register result after F-u005, and the represented target-selection relationships.

It does not establish that any dependency preserves globals, leaves memory untouched, returns normally, or owns or releases memory. Ignoring a result does not make a call effect-free. All unshown dependency semantics are **documented_only**.

## Acceptance examples

These are conditional static examples, not executed tests. They assume the relevant operations complete and the nominal word interpretation applies.

1. **Unsigned input shifts.** For third argument \(0x80000000\), shifts by 8, 7, and 15 produce \(0x00800000\), \(0x01000000\), and \(0x00010000\). The input-derived byte destination is \(0x0001001c\).

2. **Byte extraction.** For first byte source \(0x12345678\), the first selector-mediated byte value is \(0xe0\); the later extraction writes \(0x23\). The selectors determining their destinations are read separately.

3. **Retained-base overwrite.** If the sixth index is 3, the retained base is 48. One word store writes 12 at address 68. After F-u044–F-u048, another writes 48 at address 68. An intervening dependency must not be assumed to observe 48.

4. **Fixed loop selector address.** If the second byte source is 1, the retained byte word is 4 and the loop selector address is 4097. Even if successive F-u089 results are 7 and 9, later selector reads remain at 4097. Those results change loop arguments, not this address.

5. **Fresh reads at that fixed address.** If the byte at the loop selector address is 5 in one cycle and 9 in the next, zero stores target addresses 22 and 26. Reusing the first selector would violate the represented behavior.

6. **Loop-field transition.** The first F-u080 invocation receives zero as its second explicit argument for every permitted initial loop field. If F-u089 later returns 1, the next ordinary value is 4 and the next second argument is \(0x00040000\), under 32-bit reduction.

7. **Target transition.** The first F-u080 invocation uses the entry-derived target. Its second invocation uses the back-edge-derived target, even if both cycles’ explicit arguments happen to match.

8. **No normal completion.** If every access succeeds and every dependency returns, reaching the loop does not lead to a represented return. Executing one cycle and returning would change the contract.

## Uncertainties

- **Control-flow fidelity — documented_only.** Invalid instruction data and removed regions may conceal or distort executable behavior.
- **Actual ABI — documented_only.** Explicit arguments do not account for all consumed incoming state or dependency inputs.
- **Arithmetic and addressing — documented_only.** Nominal widths are visible; signed-overflow execution, address wrapping, and address conversion are not independently verified.
- **Memory model — documented_only.** Alignment, byte order, mapping, volatility, concurrency, and device semantics are unknown.
- **Dependency semantics — documented_only.** Target identities, global preservation, ownership effects, and failure behavior are unresolved.
- **Application intent — inferred only.** Bit manipulation and indirect mutation are evident operations, but no higher-level purpose is established.

## Evidence

Established static evidence consists of the supplied representation’s access widths, explicit argument dataflow, call ordering, arithmetic, nine pre-loop word stores, six pre-loop byte stores, five pre-loop selector reads, and unconditional loop.

The loop’s selector-address operand remains tied to the retained second-byte-source-derived value. The back edge updates the ordinary value and loop field, not that retained address operand. Separately, the control value selecting F-u080’s target differs between entry and the back edge.

The supplied warnings report invalid instruction data and removal of unreachable regions. They limit confidence in completeness; they do not establish successful execution.

No runtime tests, implementation inspection beyond the supplied material, dependency inspection, or memory observations have been performed. The acceptance examples express static arithmetic and ordering consequences, not observed results.
