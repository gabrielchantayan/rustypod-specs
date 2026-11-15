<!-- Trusted provenance: F-0671cee1ee3c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0671cee1ee3c

## Purpose

F-0671cee1ee3c represents an indefinitely repeating sequence of indirect calls, computed-address memory accesses, and retained-state updates. The supplied static representation establishes no application-level purpose, object model, ownership protocol, or successful completion condition.

The contract is an ordered trace: calls separate explicit reads and writes, returned values determine later addresses and arguments, and selected values carry into the next iteration. The represented loop has no exit test or ordinary return. This does not establish that execution continues successfully forever: a dependency may not return, and a memory access may fail.

**Established static evidence:** the argument uses, memory-access widths, call ordering, arithmetic relationships, and loop transition described below.

**Inferred intent:** repeated scaling and field extraction may construct encoded addresses or values. No specific encoding or data structure is established.

**Unknown:** whether the represented function boundary, calling conventions, and control flow accurately capture executable behavior. No runtime tests have been performed.

## Interface

### Explicit inputs

The represented interface has four arguments and no declared result.

| Argument | Represented width | Observable use |
|---|---:|---|
| First | 32 bits, uninterpreted | No explicit body use. |
| Second | 32 bits, represented as signed | Passed twice to the first call of an iteration and stored as a word. Replaced at the iteration boundary. |
| Third | 32 bits, represented as signed | Passed as the third argument to the first call. Replaced at the iteration boundary. |
| Fourth | 32 bits, uninterpreted | Low eight bits retained at iteration entry for four direct byte writes. Set to zero late in the iteration. |

The absence of an explicit use of the first argument does not prove that its incoming machine register is irrelevant to the actual ABI.

### Additional entry state

The body uses two values without explicit argument bindings:

- A represented 32-bit machine-state value multiplied by 2²⁸ to form the initial memory base.
- A link-like machine-state value used to derive the first indirect-call target.

The body does not initialize these before their first use. Valid incoming values are therefore prerequisites for any concrete interpretation.

One global word is explicitly read during each iteration. Its address, meaning, initialization, and independence from overlapping storage are unresolved.

### Results and preconditions

There is no ordinary result, status code, explicit failure return, or termination protocol.

Completion of an iteration requires all reached calls to return and all reached accesses to be permitted. Actual calling conventions must provide the values consumed after calls. The body contains no explicit validation of these conditions.

No allocation, deallocation, reference counting, ownership transfer, or release obligation is established. A computed value passed to a dependency is not, by itself, evidence of a pointer or transferred ownership.

## Behavior

### Widths and conventions

A **word** is 32 bits; a **byte** is eight bits. Byte stores retain only the low eight bits. Selector-byte reads are unsigned and yield values from 0 through 255.

The following descriptions use semantic labels for retained values, not implementation identifiers:

- **Quadruple** means multiplication by four.
- **Field extraction** means retaining bits 0–29 of a word, then shifting right by 20. The result is ten bits, ranging from 0 through 1023.
- **Byte-selected store through a base** means reading a byte at that base plus 1 and writing a byte at the resulting unsigned value plus 17.

A byte-selected destination is consequently between numerical addresses 17 and 272. Selection does not load a pointer-sized value.

Unsigned word operations establish bit-level truncation. Other expressions use represented signed 32-bit values; signed overflow and signed left shifts do not establish portable arithmetic behavior. Conclusions requiring discarded high bits are conditional on 32-bit bit-vector arithmetic. Actual address interpretation, including wraparound, remains unresolved.

The seven phases below describe one completed represented iteration. Calls described as intervening remain part of the observable ordering even when their ordinary results are unused.

### 1. Entry and initial publication

The low byte of the fourth argument is retained. The initial base is formed by multiplying the incoming machine-state value by 2²⁸.

The first dependency receives the second argument twice, then the third argument. Three intervening calls follow. The next call returns the **initial index**.

The current second argument is stored as a word at four times the initial index plus 20. A dependency then receives four times that index.

After three intervening calls, another call returns the **initial packed word**. Two byte writes occur in order:

1. Retained argument byte at initial base plus 17.
2. Retained argument byte at initial base plus 1.

A dependency receives the initial packed word shifted left by two. The next call returns an opaque word, which is passed to the following dependency.

After three intervening calls, another call returns a value. The same two direct byte writes occur again, in the same order, before a dependency receives that returned value shifted left by two.

These are four distinct write events. Intervening dependencies may change memory, so the second pair cannot be treated as redundant.

### 2. Initial extraction and zero publication

After three intervening calls, a call returns the **first record index**.

A word is written at sixteen times this index plus 20. Its value is the initial index shifted left by 31. Under 32-bit bit-vector arithmetic, the result depends only on the initial index’s low bit.

Next, a word is read at the field extraction of the initial packed word plus 20. This numerical address ranges from 20 through 1043. There is no additional multiplication by four, and alignment is not assured.

A dependency receives four times the first record index. The next call returns the retained **bridge result**. Another dependency receives the loaded word shifted right by 22, then left by 28. With 32-bit truncation, only original bits 22–25 remain, placed in bits 28–31.

After three intervening calls, another call returns the **zero-target index**. A zero word is stored at four times that index plus 20. A dependency then receives four times that index.

### 3. Initial-base selections and global snapshot

After three intervening calls, a call returns the **second packed word**.

A fresh selector byte is read at initial base plus 1. The low byte of the second packed word shifted left by two is written at selector plus 17. A dependency receives the full shifted word.

The next call returns an opaque word. A two-argument dependency receives that word and the second packed word shifted left by 30.

After three intervening calls, another call returns a value. A second selector byte is read at initial base plus 1 and retained. The low byte of the second packed word’s field extraction is written at retained selector plus 17. A dependency receives the newly returned value shifted left by two.

Four calls follow; the final ordinary result is unused. The bridge result shifted left by two is then stored as a word at retained selector plus 20. This store reuses the retained byte; it performs no additional selector read.

The global word is read and retained as the **global snapshot**. A dependency receives four times the first record index. After three intervening calls, a call returns a represented 64-bit value.

Its low 32 bits shifted left by two form the **first wide-derived scale**. A third selector is freshly read at initial base plus 1. The low byte of the global snapshot shifted right by 22 is written at selector plus 17. A dependency then receives the high 32 bits of the wide result.

The next call returns an opaque word. Another dependency receives that word and the first wide-derived scale shifted left by 28.

After three intervening calls, a call returns another value. A fourth selector is freshly read at initial base plus 1, and zero is written at selector plus 17. A dependency receives the returned value shifted left by two.

### 4. First paired word publication

Four calls follow, the last returning a value supplied twice, each time shifted left by two, to a dependency.

After three intervening calls, a call returns the **first publication value**. Four times this value is retained as the **first publication word** and stored at the first wide-derived scale multiplied by 4096, plus 20.

A dependency receives the first wide-derived scale shifted left by 14. After three intervening calls, another call returns the **first publication index**. The retained publication word is stored at four times that index plus 20, followed by a dependency receiving four times the index.

After three intervening calls, a call returns a second represented 64-bit value. Its low half shifted left by two forms the **second wide-derived scale**.

A selector byte is read at the first wide-derived scale multiplied by 16384, plus 1. The low byte of the first publication word’s field extraction is written at selector plus 17. A dependency receives the high half of the second wide result.

The next call returns an opaque word, which is passed to another dependency. A further call returns a third represented 64-bit value. Its low and high halves are supplied separately to the following dependency, in that order.

### 5. Selector clears and second paired publication

After one intervening call, a call returns a value. A byte-selected zero store occurs through the second wide-derived scale multiplied by 2²⁸. A dependency receives the returned value shifted left by two.

After three intervening calls, another call returns the **second publication source**. Four times this source becomes the **second publication base**.

A second byte-selected zero store occurs through four times the second wide-derived scale. This latter base is retained as the **retained selector base** for subsequent selections.

A dependency receives the second wide-derived scale shifted left by ten. The next call returns an opaque word. Another dependency receives that word and the bridge result shifted left by 28.

After three intervening calls, another call returns a value. A fresh byte-selected zero store occurs through the retained selector base. A dependency receives the returned value shifted left by two.

Four calls follow; their final ordinary result is unused. The second publication source shifted left by 14 becomes the **second publication word**. It is stored at the second publication base multiplied by 4096, plus 20.

A dependency receives the second publication base shifted left by 14. The following call’s ordinary result is unused. The next dependency receives:

- The second publication word shifted left by 28.
- An unresolved secondary outgoing machine-state value.

Under 32-bit bit-vector arithmetic, the first argument is zero: the retained word already has fourteen trailing zero bits.

After three intervening calls, another call returns the **second publication index**. The retained publication word is stored at four times that index plus 20. A dependency receives four times the index.

### 6. Third paired publication

After three intervening calls, a call returns the **third publication source**. Four times this source becomes the **third publication base**.

A fresh selector is read through the retained selector base. The low byte of the second publication word’s field extraction is written at selector plus 17.

A dependency receives the second publication source shifted left by 26. The following call returns an opaque word. Another dependency receives that word and the third publication base shifted left by 28.

The next call returns a fourth represented 64-bit value. The following dependency receives its low and high halves separately, in that order.

After one intervening call, a call returns another value. A fresh byte-selected zero store occurs through the retained selector base. A dependency receives the returned value shifted left by two.

After three intervening calls, another call returns the **third publication value**. Four times it becomes the **third publication word**, stored at the third publication base multiplied by 4096, plus 20.

A four-argument dependency then receives, in order:

1. Third publication base multiplied by 1024.
2. The same value again.
3. Third publication source shifted left by 14.
4. Third publication word.

After three intervening calls, a call returns an index. The third publication word is stored at four times that index plus 20. A dependency receives four times the index.

### 7. Final publications and loop transition

After three intervening calls, a call returns the **penultimate index**. Four times this index becomes the **penultimate word**, stored at sixteen times the index plus 20.

The fourth argument is set to zero. A dependency receives the penultimate word three times.

After three intervening calls, another call returns an index. The penultimate word is stored at four times that index plus 20. A dependency receives four times the index.

After three intervening calls, a final call returns the **next-iteration index**. Four times this index replaces the second argument and is stored at sixteen times the index plus 20. The same quadrupled value then replaces the third argument and the machine-state value used to form the initial base.

The loop repeats without an exit test. Its next first-call target uses the link-like value assigned during the final call sequence, not necessarily the original entry value.

## Side effects

Each fully completed represented iteration has **27 explicit memory writes: 14 byte writes and 13 word writes**.

| Phase | Byte writes | Word writes | Selector-byte reads |
|---|---:|---:|---:|
| 1 | 4 | 1 | 0 |
| 2 | 0 | 2 | 0 |
| 3 | 4 | 1 | 4 |
| 4 | 1 | 2 | 1 |
| 5 | 3 | 2 | 3 |
| 6 | 2 | 2 | 2 |
| 7 | 0 | 3 | 0 |
| **Total** | **14** | **13** | **10** |

The ten explicit unsigned selector-byte reads comprise:

- Four at initial base plus 1.
- One at first wide-derived scale multiplied by 16384, plus 1.
- One at second wide-derived scale multiplied by 2²⁸, plus 1.
- Four at retained selector base plus 1.

The retained-selector word store in phase 3 adds no selector read. Other explicit reads are one computed-address word read and one global-word read.

These counts exclude accesses performed by dependencies, instruction fetching, and any ABI-level stack or register-save traffic not explicitly represented.

Addresses may alias. Selected byte destinations may overlap selector storage, word destinations, or global storage. Every fresh read must therefore retain its position in the trace. Conversely, a retained selector must not be replaced with a fresh read.

No explicit synchronization, memory barrier, transaction, rollback, or recovery operation appears. Word accesses are not universally aligned. Endianness and access atomicity are unspecified.

## Dependencies

All external calls are indirect. Dependency sites may be identified as **F-0671cee1ee3c-X001**, **F-0671cee1ee3c-X002**, and onward in represented execution order. These opaque IDs identify call sites, not proven distinct target functions.

Targets are derived from incoming state or code-associated constants with several offsets. Target identities and complete calling conventions are unresolved.

The representation shows calls with zero through four explicit arguments and four calls returning 64-bit values. A displayed argument-free call does not prove a complete zero-argument ABI. One call explicitly consumes an unresolved secondary machine-state value.

No dependency is assumed to preserve global or selector memory, avoid reentrancy, return normally, or own or release storage. Ordinary result retention described above does not establish broader machine-state preservation.

## Acceptance examples

These are conditional static criteria, not executed tests. They assume required calls return and accesses are permitted.

1. **Write accounting.** A completed iteration exposes 27 explicit writes, distributed by phase exactly as listed above. Dependency-internal writes are additional and unknown.

2. **Selector accounting.** A completed iteration performs ten explicit selector-byte reads. The bridge-result word store reuses its earlier selector and does not create an eleventh read.

3. **Argument truncation.** With fourth argument 0x123456AB, all four direct byte writes in phase 1 use 0xAB. Later completed iterations use zero for these writes because phase 7 resets that argument.

4. **Initial publication.** An initial index of 3 selects numerical address 32 for the first word store, whose value is the current second argument.

5. **Unsigned selection.** A selector byte of 250 selects numerical byte destination 267, not a pointer loaded from selector storage.

6. **Fresh versus retained selection.** Fresh selectors of 4 and 9 select byte destinations 21 and 26. If phase 3 retains 9, its later bridge-result word store targets 29 even if a dependency subsequently changes selector storage.

7. **Field extraction.** A ten-bit extracted value of 0x3AB narrows to 0xAB for a byte store. When used for the phase-2 word-read address, it selects 959 without further word scaling.

8. **Shift truncation.** Under 32-bit bit-vector arithmetic, initial indices 2 and 3 produce respectively zero and 0x80000000 for the phase-2 shifted-index store. The phase-5 second publication word shifted left by 28 is always zero under the same arithmetic model.

9. **Snapshot timing.** A dependency changing global storage after the explicit snapshot read does not change the retained snapshot used by the later byte store.

10. **Iteration propagation.** A final index of 7 makes the next second argument, third argument, and initial-base machine-state input all equal to 28.

11. **Interrupted execution.** If a dependency never returns, subsequent explicit operations are not required to occur. Earlier writes are not rolled back by any represented mechanism.

## Uncertainties

The following remain **documented_only**:

- Actual indirect targets, ABI details, and implicit register inputs.
- Accuracy of the represented function boundary and endless loop.
- Validity and initial values of the implicit machine-state inputs.
- Global storage address, interpretation, and overlap relationships.
- Provenance of the unresolved secondary outgoing value.
- Actual signed-overflow, signed-shift, and address-wrap behavior.
- Endianness, alignment requirements, access atomicity, translation, and permissions.
- Whether numerical low addresses denote ordinary memory, mapped hardware, or representation artifacts.
- Dependency side effects, ownership behavior, nonlocal exits, and failure modes.

The supplied warning about removed unreachable material prevents conclusions about excluded paths. The warning about overlapping global symbols prevents treating the global word as necessarily independent storage.

## Evidence

This contract uses only the supplied static representation: declared widths, casts, masks, shifts, explicit access widths, call order, retained-value relationships, and unconditional repetition.

Write counts follow the explicit stores in phases 1–7: byte counts 4, 0, 4, 1, 3, 2, 0 and word counts 1, 2, 1, 2, 2, 2, 3. Selector-read counts follow the four initial-base reads, the two individually computed-base reads, and four retained-base reads. The later retained-selector word store contains no new read.

No implementation access, dependency bodies, runtime traces, or tests were available. No runtime tests have been performed. Application purpose and unresolved ABI behavior are not inferred from the prior draft, and arithmetic conclusions requiring 32-bit truncation remain conditional.
