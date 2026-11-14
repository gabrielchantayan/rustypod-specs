<!-- Trusted provenance: F-c8223feba85e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c8223feba85e

## Purpose

The established behavior is an unconditional loop containing indirect dependency invocations, computed-address word writes, byte-selected writes, and reads of computed storage and a global word. Each completed iteration explicitly replaces two carried argument values and assigns a continuation-related link context before its final dependency invocation.

No normal return, finite completion condition, success result, or explicit error-handling branch is visible. A higher-level interpretation such as initialization, dispatch, serialization, or device control is not established.

This contract describes the visible operations and their ordering. Dependency effects remain opaque; in particular, dependencies are not assumed to preserve globals, calling-context state, or memory ownership.

## Interface

### Arguments and execution context

Four argument positions are displayed:

| Position | Width | Established explicit use |
|---|---:|---|
| First | 32 bits | No explicit use in the visible body. |
| Second | 32 bits | No explicit use in the visible body. |
| Third | 32 bits | Provides a shifted dependency argument; replaced later in each iteration. |
| Fourth | 32 bits | Provides computed addresses and a shifted dependency argument; replaced near the iteration’s end. |

The third and fourth positions act as carried state. Their entry values apply initially; later iterations use the replacements shown by the representation, subject to unresolved calling-convention behavior.

An incoming link-context value also participates in execution. The first dependency target is formed by adding 4094 to that context. Other targets are computed from fixed continuation-related values.

Near the end of every completed iteration, the representation assigns a fixed continuation-related value to the link context **after F-dep-145 and before F-dep-146**. The final invocation therefore follows that assignment. Preservation or alteration of this context across F-dep-146 is unknown; the assignment does not establish the context’s value after that invocation.

The four arguments alone are insufficient execution prerequisites. Valid call targets, an appropriate calling convention, accessible computed storage, and any requirements imposed by dependencies are also necessary.

### Results and widths

There is no visible normal return value. Observable results comprise dependency invocations, memory accesses, carried-state replacements, and continued looping.

For this contract:

- A **word** is 32 bits.
- A **selector byte** is an unsigned 8-bit value.
- A **pair result** is a displayed 64-bit result interpreted as low and high 32-bit halves.
- A **selected byte destination** is address 17 plus a freshly read selector byte, unless a saved selector is explicitly specified.
- A word’s **field** is obtained by masking with `0x3fffffff` and shifting right by 20.

The field is ten bits wide, ranging from 0 through 1023. A field written as a byte retains only its low eight bits: original word bits 20–27.

Address expressions below retain the displayed scaling. Exact overflow, signed-shift, pointer-arithmetic, and address-width semantics are not fully established. Conclusions requiring 32-bit modular arithmetic are explicitly conditional.

## Behavior

### Execution and dependency notation

F-dep-01 through F-dep-146 identify invocation occurrences in execution order, not necessarily distinct implementations.

A range such as “F-dep-03 through F-dep-06” means four separate invocations in ascending order. Unless arguments are specified, the representation displays none. This does not prove absence of machine-level arguments or hidden calling-context inputs.

All later behavior is conditional on earlier invocations returning and earlier memory accesses completing.

### 1. First word publication

F-dep-01 is invoked through the incoming link-context-relative target. Its word result is shifted left by two and supplied twice, as identical arguments, to F-dep-02.

F-dep-03 through F-dep-06 follow. The result of F-dep-06, shifted left by two, becomes the **first scaled word**. That word is written at address `4096 × fourth carried input + 20`.

F-dep-07 receives the fourth carried input shifted left by 14. After F-dep-08 through F-dep-11, the first scaled word is written again, this time at `4 × result of F-dep-11 + 20`.

The two stores retain the same word value. Their addresses need not coincide. F-dep-12 then receives four times the result of F-dep-11.

### 2. First pair result and two selected byte writes

F-dep-13 through F-dep-16 execute. F-dep-16 supplies a pair result. Four times its low half becomes the **first scaled selector**.

A selector byte is read at `16384 × fourth carried input + 1`. The low byte of the first scaled word’s field is written at the selected byte destination.

Only after that write does F-dep-17 receive the pair’s high half. F-dep-18 supplies the argument to F-dep-19. F-dep-20 produces another pair, whose low and high halves are passed to F-dep-21 in that order. F-dep-22 and F-dep-23 follow.

A selector byte is then read at `268435456 × first scaled selector + 1`; zero is written at its selected byte destination. F-dep-24 receives the result of F-dep-23 shifted left by two.

The field publication precedes all of F-dep-17 through F-dep-23. The selected zero write follows those invocations.

### 3. Reused selector storage and second word source

F-dep-25 through F-dep-28 execute. The result of F-dep-28 becomes the **second word source**; its value shifted left by two is also retained.

The **reused selector address** is `4 × first scaled selector + 1`. A byte is read there, and zero is written at its selected destination.

F-dep-29 receives the first scaled selector shifted left by ten. F-dep-30 supplies the first argument to F-dep-31; the second argument is the third carried input shifted left by 28. F-dep-32 through F-dep-35 follow.

The reused selector address is read again, and zero is written at the destination selected by this new read. F-dep-36 receives the result of F-dep-35 shifted left by two. F-dep-37 through F-dep-40 then execute.

The second selector read is not reuse of the first value. Intervening dependencies or aliasing writes may change the byte.

### 4. Second word publication

The **second scaled word** is the second word source shifted left by 14. It is written at address `4096 × (second word source shifted left by two) + 20`.

F-dep-41 receives the retained, two-bit-shifted second word source shifted left by another 14. F-dep-42 follows. F-dep-43 receives two displayed arguments:

- The second scaled word shifted left by 28.
- An auxiliary register-derived value whose provenance is unresolved.

After F-dep-44 through F-dep-47, the second scaled word is written at `4 × result of F-dep-47 + 20`. F-dep-48 receives four times that result.

F-dep-49 through F-dep-52 execute. The result of F-dep-52 becomes the **third selector**.

The reused selector address is read again. The low byte of the second scaled word’s field is written at its selected destination.

Under 32-bit truncating shift semantics, F-dep-43’s first argument is zero: the original word has already been shifted left by 14 before a further shift by 28. This does not resolve its auxiliary argument.

### 5. Selected clear and third word publication

F-dep-53 receives the second word source shifted left by 26. F-dep-54 supplies the first argument to F-dep-55; the second is `(third selector shifted left by two)` shifted left by 28.

F-dep-56 produces a pair result. Its low and high halves, in that order, are passed to F-dep-57. F-dep-58 and F-dep-59 follow.

The reused selector address is read again, and zero is written at its selected destination. F-dep-60 then receives the result of F-dep-59 shifted left by two.

After F-dep-61 through F-dep-64, the result of F-dep-64 shifted left by two becomes the **third scaled word**. It is written at `4096 × (third selector shifted left by two) + 20`.

F-dep-65 receives four arguments, in order:

1. `1024 × (third selector shifted left by two)`;
2. The same value;
3. The third selector shifted left by 14;
4. The third scaled word.

### 6. Further word publications

After F-dep-66 through F-dep-69, the third scaled word is written at `4 × result of F-dep-69 + 20`. F-dep-70 receives four times that result.

F-dep-71 through F-dep-74 produce the next retained result. Four times the result of F-dep-74 becomes the **fourth scaled word**, written at `16 × result of F-dep-74 + 20`.

F-dep-75 receives three identical arguments, each the fourth scaled word. After F-dep-76 through F-dep-79, that same word is written at `4 × result of F-dep-79 + 20`. F-dep-80 receives four times that result.

F-dep-81 through F-dep-84 follow. The result of F-dep-84 becomes the **late selector**. Four times it becomes the **fifth scaled word**, written at `16 × late selector + 20`.

The **late base** is `1073741824 × late selector`. F-dep-85 receives three identical arguments, each the fifth scaled word.

After F-dep-86 through F-dep-89, the result of F-dep-89 becomes the **late destination selector**. The fifth scaled word is written at `4 × late destination selector + 20`. F-dep-90 receives four times the late destination selector.

F-dep-91 through F-dep-94 execute. The word result of F-dep-94 is retained for later argument formation and address selection.

### 7. Direct byte clears and a computed word read

Zero is written directly at `late base + 17`, then directly at `late base + 1`. These writes do not read a selector byte.

F-dep-95 receives the retained result of F-dep-94 shifted left by two. F-dep-96 supplies the argument to F-dep-97. F-dep-98 through F-dep-101 follow.

The same two direct zero writes occur again, in the same order: `late base + 17`, then `late base + 1`.

F-dep-102 receives the result of F-dep-101 shifted left by two. F-dep-103 through F-dep-106 execute. Four times the result of F-dep-106 becomes the **final scaled selector**.

The late destination selector shifted left by 31 is written as a word at `16 × result of F-dep-106 + 20`.

Next, a word is read at `field of retained result of F-dep-94 + 20`. This read follows the preceding write; aliasing can make that ordering significant.

### 8. Third carried-input replacement and late selected writes

F-dep-107 receives the final scaled selector. F-dep-108 then supplies the replacement third carried input.

F-dep-109 receives the word just read, shifted logically right by 22 and then left by 28. F-dep-110 through F-dep-113 follow.

A zero word is written at `4 × result of F-dep-113 + 20`. F-dep-114 receives four times that result. After F-dep-115 through F-dep-118, the result of F-dep-118 becomes the **final field word**.

A byte is read at `late base + 1`. The low byte of the final field word shifted left by two is written at its selected destination.

F-dep-119 receives the final field word shifted left by two. F-dep-120 supplies the first argument to F-dep-121; the second is the final field word shifted left by 30. F-dep-122 through F-dep-125 follow.

The byte at `late base + 1` is read again and retained as the **saved selector**. The low byte of the final field word’s field is written at `saved selector + 17`.

F-dep-126 receives the result of F-dep-125 shifted left by two. F-dep-127 through F-dep-130 execute.

The replacement third carried input shifted left by two is written as a word at `saved selector + 20`. This address uses the saved byte, not another read of `late base + 1`.

### 9. Global snapshot, fourth carried-input replacement, and continuation

A global 32-bit word is read after the saved-selector word write. Its value is retained as the **global snapshot**.

F-dep-131 receives the final scaled selector. F-dep-132 through F-dep-135 follow, with F-dep-135 supplying a pair result. The low half shifted left by two replaces the fourth carried input.

A fresh selector byte is read at `late base + 1`. The low byte of the global snapshot shifted logically right by 22 is written at its selected destination. This uses the earlier snapshot, not a new global read.

F-dep-136 receives the pair’s high half. F-dep-137 supplies the first argument to F-dep-138; the second is the replacement fourth carried input shifted left by 28.

F-dep-139 through F-dep-142 execute. Another byte is read at `late base + 1`, and zero is written at its selected destination.

F-dep-143 receives the result of F-dep-142 shifted left by two. F-dep-144 and F-dep-145 then execute.

**After F-dep-145 returns, the link context is assigned the fixed continuation-related value. F-dep-146 is invoked afterward.** Its target is separately computed from fixed continuation-related state; the visible target calculation does not use the newly assigned link context.

If F-dep-146 returns, execution reaches the unconditional loop repetition. No additional link-context assignment is shown between that invocation and the next iteration. Its effects on calling-context state, including preservation of the assigned context, remain unresolved.

## Side effects

The explicit memory interface consists of unsigned byte reads, byte writes, 32-bit writes, one computed-address word read, and one global word read per completed iteration. These operations are ordered, not transactional.

Selected byte destinations range from address 17 through address 272. The saved-selector word destination ranges from address 20 through address 275 and may be unaligned. The field-derived word-read address ranges from 20 through 1043 and may also be unaligned.

There is no established record layout beyond these address calculations. In particular, offsets 1, 17, and 20 do not prove a shared structure or allocation.

Aliasing remains possible among computed addresses, selector storage, global storage, and dependency-accessed memory. Consequently:

- Repeated selector reads are distinct observable operations.
- Direct zeroing of selector storage does not establish later selector values across intervening calls.
- Saved values remain distinct from later reads of their original storage.
- Repeated writes cannot be discarded merely because their displayed values match.

No explicit allocation, release, ownership transfer, bounds validation, null check, synchronization, rollback, or fault recovery is visible. Dependency ownership and resource effects are unknown.

## Dependencies

F-dep-01 through F-dep-146 are opaque occurrence IDs. They establish ordering without asserting implementation identity or distinctness. All displayed invocations are indirect.

The following dependency properties are **documented_only**:

- Full calling conventions, hidden arguments, and return-register conventions.
- Global, register, and link-context preservation.
- Memory reads and writes, allocation, release, callbacks, and external effects.
- Failure reporting, exceptions, traps, or nonreturning behavior.
- Whether pair results represent unified 64-bit returns or paired register outputs.

Displayed argument lists are not complete dependency contracts. In particular, assigning link context before F-dep-146 neither proves that invocation ignores it nor proves that it preserves it.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Field narrowing.** If the first scaled word is `0x3ab00000`, its field is `0x3ab`; the selected byte write stores `0xab`.

2. **Unsigned selection.** A selector byte of `0xff` selects address 272, not address 16.

3. **Fresh reads.** If the reused selector byte is 3 for the first clear and 9 for the next clear, those writes target addresses 20 and 26, respectively.

4. **Saved selection.** A saved selector of 7 makes the later word destination address 27 even if intervening dependencies change the original selector storage to 12.

5. **Snapshot timing.** A global snapshot of `0x00400000` supplies byte value 1 to the later snapshot-derived write even if a dependency subsequently changes the global to zero.

6. **Conditional modular addressing.** Under 32-bit modular arithmetic, a late selector congruent to 2 modulo 4 produces late base `0x80000000`. Each direct-clear pair targets `0x80000011` before `0x80000001`, assuming valid accesses.

7. **Carried values.** If F-dep-108 returns 5 and F-dep-135’s low half is 3, the explicit replacements are third carried input 5 and fourth carried input 12.

8. **Link-context ordering.** If execution reaches the final calls, the order is F-dep-144, F-dep-145, link-context assignment, F-dep-146, then loop repetition if that invocation returns. A dependency trace alone cannot justify moving the assignment after F-dep-146. The context remaining afterward is unknown without a preservation contract.

9. **No finite success.** If every invocation returns and every explicit access succeeds, the visible control flow repeats rather than returning normally.

## Uncertainties

The following remain **documented_only**:

- Exact overflow and signed-shift semantics; a modular machine-word interpretation is not proven by the representation alone.
- Pointer width, endianness, alignment requirements, valid address ranges, and invalid-access behavior.
- Whether addressed storage is ordinary memory, device memory, or another address space.
- Auxiliary register provenance and omitted calling-context inputs.
- Dependency identities, effects, and preservation guarantees.
- Actual context observed by dependencies through calling machinery, including F-dep-146.
- Whether F-dep-146 preserves the assigned link context for the next iteration.
- Global-storage overlap and aliasing with computed accesses.
- Completeness of the supplied representation and externally imposed termination.

Inferred intent is limited to repeated propagation of scaled words and narrowed fields. No application-level meaning is established.

## Evidence

Established static evidence consists of the supplied function representation: four displayed 32-bit arguments, an unconditional loop, indirect invocation ordering, explicit access widths and address expressions, field masks, carried-value replacements, and no visible normal return or error branch.

The final sequence explicitly places the link-context assignment between the penultimate and final invocations. This establishes assignment timing, not preservation across the final invocation.

Accompanying warnings identify removed unreachable material and overlapping global symbols. They limit confidence in completeness and storage interpretation without establishing additional reachable behavior.

No runtime tests have been performed. All acceptance examples are conditional consequences of static evidence, not observed execution results.
