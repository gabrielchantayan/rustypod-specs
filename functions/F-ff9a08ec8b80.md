<!-- Trusted provenance: F-ff9a08ec8b80; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ff9a08ec8b80

## Purpose

F-ff9a08ec8b80 performs a fixed startup sequence of indirect dependency calls and selector-directed byte writes, then enters an unconditional processing loop. The loop carries forward dependency-derived arguments and dispatch state.

The first invocation of the loop’s leading call uses a different represented target from every subsequent invocation. This distinction is independent of the argument changes between the first and later iterations.

The supplied static evidence establishes call ordering, byte-access relationships, arithmetic transformations, and represented control flow. It does not establish an application-level purpose, such as device control, serialization, decoding, or allocation.

This contract distinguishes:

- **Established static evidence:** relationships directly present in the supplied representation.
- **Conditional interpretation:** consequences requiring a stated machine-width or execution assumption.
- **documented_only unknowns:** matters not resolved by the available evidence.

The representation reports invalid instruction data and removed unreachable regions. Its completeness and correspondence to executable instructions are therefore unresolved. No runtime tests have been performed.

## Interface

### Explicit inputs

| Input | Represented width | Established use |
|---|---:|---|
| First input | 32-bit unsigned word | Supplies an argument after a logical right shift by eight; also determines a repeatedly read selector-source address. |
| Second input | 32-bit uninterpreted value | Passed unchanged to F-D01. |

No pointer, handle, object, ownership, or signedness interpretation is established for the second input.

### Implicit entry state

The represented behavior additionally depends on:

- An incoming 32-bit unsigned word supplying two startup write values.
- An incoming address-like value locating a selector byte one byte beyond that value.
- An incoming control-address-like value used to determine the target of F-D01.

These are prerequisites, not additional conventional parameters established by the evidence. Their origins, initialization, and calling-convention roles are unknown.

### Output and termination

No explicit return value or normal return path is shown.

After startup, execution repeats the loop without a represented exit condition. Indefinite continuation is conditional on dependencies returning and memory accesses completing. Faults, non-returning dependencies, external interruption, or inaccuracies in the supplied representation can prevent that continuation.

### Preconditions for progress

The represented sequence requires accessible selector sources, writable selected destinations, callable indirect targets, and compatible argument/result conventions. No validation of these conditions is shown.

No independence between selector sources and write destinations is required by the representation. Aliasing remains possible.

## Behavior

### Selector reads and byte-write layout

Each explicit write stores exactly one byte and is preceded by a fresh, one-byte selector read.

For the first two writes, the selector source is one byte beyond the implicit incoming address. For the startup clear and every loop clear, the source is derived from the first explicit input.

A selector is interpreted as an unsigned eight-bit value. Its destination is the selector value plus `0x11`, giving the inclusive destination range `0x11` through `0x110`.

The selector-source address is **not** added to this destination. For example, selector `0x20` selects destination `0x31`, regardless of where the selector was read.

No selector value is cached across writes. Changes caused by dependencies, earlier writes, or external actors can affect later destination selection.

### Dependency naming and dispatch

F-D01 through F-D24 identify displayed call sites, not authenticated implementations. F-D25 and F-D26 identify the two represented targets used by the shared call site F-D15:

- **F-D25:** the target selected for the first iteration.
- **F-D26:** the target selected for every subsequent iteration.

These two represented target values differ. That does not independently establish distinct implementation bodies, address-space mappings, or instruction-entry semantics.

“No displayed arguments” means only that the supplied representation lists none. Hidden register inputs, memory inputs, and other machine-state dependencies remain unknown.

### Startup: initial writes and calls

Startup begins by reading the implicit-address selector and writing the low eight bits of the implicit incoming word to the selected destination.

F-D01 then receives the second explicit input unchanged. Its target is derived from the incoming control-address-like state.

The next calls occur in this order:

- F-D02, with no displayed arguments.
- F-D03, receiving the first input logically shifted right by eight.
- F-D04, F-D05, and F-D06, with no displayed arguments.
- F-D07, supplying a represented 32-bit signed result.

After F-D07 returns, the function rereads the implicit-address selector and writes bits 22–29 of the implicit incoming word to the selected destination. This is not the word’s most significant byte.

F-D08 receives the F-D07 result shifted left by two. F-D09, F-D10, and F-D11 follow, with no displayed arguments. F-D12 then supplies a represented 32-bit unsigned result.

The ordering places the second startup selector read and write **between** F-D07 and F-D08. An implementation that delays that write until after F-D08 would not preserve the represented order.

### Startup: state extraction and loop preparation

The F-D12 result is masked to retain its low 30 bits, then logically shifted right by 25. The resulting unsigned state is exactly the original result’s bits 25–29, with range 0–31.

The function then reads the input-derived selector source and writes zero to its selected destination. F-D13 receives twice the extracted state, an even value in the range 0–62.

Before invoking F-D14, the representation establishes the dispatch state that will select F-D25 at the first F-D15 invocation. F-D14 supplies the initial loop-carried argument.

Thus, startup establishes three separate pieces of loop-entry state:

- The F-D14 result, used as the first argument.
- The five-bit state extracted from F-D12, used to derive the second argument.
- The dispatch selection for F-D25.

The dispatch selection is not derived from the F-D14 result.

Startup contains fourteen displayed calls and three selector-read/write pairs.

### Repeated processing and observable order

Each complete iteration has ten displayed dependency calls and one selector-read/zero-write pair.

The iteration begins at F-D15, using the currently selected target and two displayed arguments:

1. The current loop-carried argument.
2. The current unsigned state logically shifted right by eight.

F-D16, F-D17, and F-D18 follow, in that order, with no displayed arguments. F-D19 then supplies a represented 32-bit signed result.

After F-D19 and before F-D20, the function freshly reads the input-derived selector source and writes zero to the selected destination. F-D20 receives the F-D19 result shifted left by two.

F-D21, F-D22, and F-D23 follow, in that order, with no displayed arguments.

During setup for the final call, F-D24, the representation establishes the dispatch state selecting F-D26 for the next iteration’s F-D15 invocation. This selection is established **before F-D24 executes**, rather than being calculated from its returned value.

F-D24 supplies a represented 32-bit signed result. That result determines both next-iteration argument states:

- The next loop-carried argument is the result shifted left by two.
- The next unsigned state receives the result shifted left by 26.

Execution then returns unconditionally to F-D15. No represented operation restores the first-iteration dispatch selection. Consequently, all later iterations select F-D26, subject to the representation’s fidelity and calling-convention limitations.

### First iteration versus subsequent iterations

The first F-D15 invocation has these requirements:

| Property | First iteration |
|---|---|
| Selected target | F-D25, established immediately before the startup call F-D14 |
| First argument | F-D14 result |
| Second argument | Zero |

The second argument is zero because the startup state is at most 31 and is logically shifted right by eight.

For every subsequent F-D15 invocation:

| Property | Subsequent iterations |
|---|---|
| Selected target | F-D26, established during the preceding iteration’s F-D24 setup |
| First argument | Preceding F-D24 result shifted left by two |
| Second argument | Preceding F-D24 result shifted left by 26, assigned to unsigned state, then logically shifted right by eight |

Under 32-bit machine-word shift semantics, the later first argument retains the low 32 bits of the two-bit left shift. The later second argument consists of the preceding F-D24 result’s low six bits placed in positions 18–23. It is a multiple of `0x40000`, ranging from zero through `0xFC0000`.

Correct arguments alone are insufficient: using F-D25 again after the first iteration would fail to preserve the represented dispatch behavior.

### Widths and address arithmetic

The input-derived selector source is calculated by multiplying the first input by `0x400` and adding one.

The multiplication is represented as 32-bit unsigned arithmetic, so overflow beyond 32 bits is discarded. Only the first input’s low 22 bits affect the selector-source address. Under that interpretation, the resulting address ranges from `0x00000001` through `0xFFFFFC01` in increments of `0x400`.

This wrapping does not apply to F-D03’s argument calculation: that argument uses a logical right shift of the full first input. Two inputs selecting the same selector-source address can therefore supply different F-D03 arguments.

Five-bit extraction and doubling do not overflow their represented widths.

Several other left shifts operate on represented signed results. Exact behavior for negative values and overflowing shifts is not independently established. The low-32-bit descriptions in this contract are conditional machine-word interpretations, not assertions of defined portable-language signed arithmetic.

Pointer width, address conversion, and validity of the calculated addresses remain unknown.

## Side effects

### Direct memory effects

The represented direct writes are:

- One startup byte containing the implicit word’s low eight bits.
- One startup byte containing that word’s bits 22–29.
- One startup zero byte through the input-derived selector.
- One additional zero byte per completed iteration.

All destinations are selected through fresh byte reads. Writes can overlap one another or alias selector sources.

The evidence does not establish volatility, atomicity, synchronization, memory barriers, exclusive access, or cross-thread observation order. The stated order is the explicit order in the supplied representation.

### Dependency effects and preservation

Dependencies may read or modify memory, change global state, block, fail, or not return. No dependency body or preservation contract is available.

The described loop-carried values and dispatch transitions reflect the supplied representation. They do not establish that arbitrary called implementations preserve physical registers, globals, or other implicit machine state.

### Ownership and lifetime

No explicit allocation, release, reference-count update, or ownership transfer is shown.

Passing the second input to F-D01 does not establish that it identifies memory or transfers ownership. Dependency results are consumed as words without an established resource-lifetime interpretation.

### Errors and partial execution

No validation, retry, error result, rollback, or cleanup path is displayed.

An operation that faults, blocks, or fails to return prevents the later represented operations from being required along that execution prefix. Earlier completed writes are not undone by any displayed local mechanism.

## Dependencies

| Opaque ID | Displayed contract |
|---|---|
| F-D01 | Receives the second input unchanged; target depends on incoming control-address-like state. |
| F-D02 | No displayed arguments; result unused. |
| F-D03 | Receives the first input logically shifted right by eight. |
| F-D04–F-D06 | Ordered startup calls; no displayed arguments; results unused. |
| F-D07 | Supplies the signed word shifted left by two for F-D08. |
| F-D08 | Receives the shifted F-D07 result. |
| F-D09–F-D11 | Ordered startup calls; no displayed arguments; results unused. |
| F-D12 | Supplies the unsigned word whose bits 25–29 become startup state. |
| F-D13 | Receives twice that five-bit state. |
| F-D14 | Supplies the first loop-carried argument; its call setup establishes the initial F-D15 target selection. |
| F-D15 | Shared loop-leading call site; first dispatches to F-D25, subsequently to F-D26; receives the current argument and unsigned state shifted right by eight. |
| F-D16–F-D18 | Ordered per-iteration calls; no displayed arguments; results unused. |
| F-D19 | Supplies the signed word shifted left by two for F-D20. |
| F-D20 | Receives the shifted F-D19 result. |
| F-D21–F-D23 | Ordered per-iteration calls; no displayed arguments; results unused. |
| F-D24 | Supplies both next-iteration argument states; its setup establishes F-D26 selection for the next F-D15 invocation. |
| F-D25 | Represented target of F-D15 only on the first iteration. |
| F-D26 | Different represented target of F-D15 on every subsequent iteration. |

All displayed calls are indirect. Apart from F-D01’s incoming-state-derived target, target selection is expressed using fixed arithmetic values and carried dispatch state. Separate identifiers do not prove separate implementation bodies.

## Acceptance examples

These are conditional static deductions, not executed tests.

### Fresh selector selection

If the first selector read yields `0x20`, the first destination is `0x31`.

If the selector subsequently changes to `0x21`, the second startup write must target `0x32`. Reusing the first selector without rereading would not preserve the behavior.

Selector `0xFF` selects `0x110`, not `0x10`; destination addition is not an eight-bit truncation.

### Startup byte extraction

For implicit word `0xFFC000A5`, the two startup payload bytes are `0xA5` and `0xFF`.

For implicit word `0x003FFFFF`, they are `0xFF` and zero. These values distinguish extraction of bits 22–29 from extraction of the most significant byte.

### Five-bit startup state

If F-D12 returns `0xC0000000`, the extracted state is zero and F-D13 receives zero.

If it returns `0x3E000000`, the extracted state is 31 and F-D13 receives 62.

In both cases, the first F-D15 invocation selects F-D25 and receives zero as its second argument.

### Dispatch transition with later arguments

Suppose F-D14 returns `0x12345678`, and the first F-D24 returns `0x00000003`.

The first F-D15 invocation must select F-D25 and receive `0x12345678` and zero.

Under 32-bit machine-word shift semantics, the second invocation must select F-D26 and receive `0x0000000C` and `0x000C0000`.

If later F-D24 calls also return three, later invocations continue selecting F-D26 with those same arguments. Dispatch does not alternate back to F-D25.

Even if F-D14 returns 12 and every F-D24 returns three, making the first arguments identical across iterations, the first-versus-later target distinction remains required.

### Later-state truncation

Under the same machine-word assumption, an F-D24 result of `0x00000040` produces next arguments `0x00000100` and zero, dispatched to F-D26. The second argument retains only information from the result’s low six bits.

### Input-derived selector source

Under represented 32-bit unsigned arithmetic:

- First input zero selects source `0x00000001`.
- First input one selects source `0x00000401`.
- First input `0x00400000` again selects source `0x00000001`.

These calculations do not establish address accessibility.

### Partial progress and continuation

If the first byte write completes but F-D01 never returns, no later call or write is required to occur. No local rollback is displayed.

If startup and successive iterations complete, each completed state update is followed by another F-D15 invocation. A normal return after a fixed iteration count is not represented.

## Uncertainties

- **documented_only — control-flow fidelity:** Invalid-instruction and removed-region warnings prevent treating the representation as a complete verified execution model.
- **documented_only — calling convention:** Physical argument passing, hidden inputs, preservation rules, and result interpretation are not independently established.
- **documented_only — implicit entry state:** Origins and validity of the incoming word, address-like value, and control-address-like value are unknown.
- **documented_only — dependency behavior:** Implementations, failures, global effects, ownership effects, and termination behavior are unavailable.
- **documented_only — dispatch interpretation:** F-D25 and F-D26 have different represented target values, but distinct implementation bodies and executable-entry semantics are not established.
- **documented_only — signed shifts:** Negative operands and overflowing signed shifts require lower-level evidence for unconditional machine behavior.
- **documented_only — address model:** Pointer width, mapping, permissions, and hardware effects remain unknown.
- **documented_only — concurrency:** Selector stability, atomicity, and synchronization are not established.
- **Inferred intent:** The sequence suggests stateful repeated processing, but no application domain is established.

## Evidence

The sole evidence is the supplied static representation and its warnings.

It directly supports the displayed call order, fresh selector reads, byte writes, five-bit unsigned extraction, input-derived selector calculation, and unconditional loop.

It also supports the dispatch distinction: the target-selection state established immediately before F-D14 differs from the state established during F-D24 setup. The former reaches the first F-D15 invocation; the latter reaches all subsequent invocations. This transition is separate from the result-derived argument updates.

No dependency implementations, verified calling convention, memory map, caller behavior, or runtime observations were supplied. Acceptance examples are conditional deductions only; none has been executed.
