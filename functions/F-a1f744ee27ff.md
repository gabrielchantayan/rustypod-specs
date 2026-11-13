<!-- Trusted provenance: F-a1f744ee27ff; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a1f744ee27ff

## Purpose

F-a1f744ee27ff performs an ordered sequence of opaque calls and computed-address memory writes, then enters an unconditional repeating region. No normal return is visible.

The repeating region has an established dispatch transition: its head call uses one computed target on the first iteration and a different computed target on every subsequent iteration. The preceding iteration’s tail installs the target used by the next iteration.

No application-level purpose is established. The evidence does not justify describing this operation as serialization, allocation, device initialization, or ordinary object manipulation.

The supplied static representation includes warnings about invalid instruction data and removed unreachable regions. This contract describes the visible representation, not independently verified executable behavior.

## Interface

### Explicit inputs

| Input | Represented width | Visible use |
|---|---:|---|
| First argument | Signed 32-bit integer | Stored at computed word addresses and supplied to the first opaque call. |
| Second argument | Uninterpreted 32-bit value | No visible use. |
| Third argument | Unsigned 32-bit integer | Logically shifted right by seven and supplied to an opaque call. |

No argument is established to be a pointer, object reference, index, length, or ownership token.

### Implicit inputs

Execution also depends on:

- An unsigned 32-bit entry value, designated **E**.
- A link-related entry value that participates in computing the first call target.
- A secondary result-register value used after an opaque call sequence.
- Bytes read from computed selector addresses.
- Results and possible external effects of opaque dependencies.

The explicit arguments alone therefore do not determine execution.

### Outputs and completion

There is no represented return value. The visible path ends in unconditional repetition rather than normal completion.

Observable effects include calls, 32-bit memory writes, byte reads, and byte writes. A dependency might fail to return or transfer control elsewhere; a memory access might fault. Those possibilities are unresolved, not specified completion mechanisms.

### Widths and notation

The following names identify contract values rather than implementation identifiers:

- **B**: E multiplied by eight, retaining the unsigned 32-bit result.
- **H**: E logically shifted right by eight.
- **J**: E logically shifted right by sixteen.
- **Rₙ**: the visible primary result of call occurrence F-d*n*, where needed below.

Word writes are 32 bits. Byte writes retain the low eight bits of their value. Selector reads produce unsigned eight-bit values, so adding 17 yields a destination between 17 and 272 inclusive.

Unsigned shifts are logical. Unsigned 32-bit arithmetic retains the low 32 bits. Some products and left shifts instead appear in signed 32-bit expressions, including expressions subsequently assigned to unsigned values. Their overflow behavior is **documented_only**; assignment to an unsigned destination does not independently establish how an earlier signed overflow behaves.

Address width, byte order, alignment rules, and mapped address ranges are unknown. Mathematical address formulas below do not establish valid pointers. Examples involving potentially overflowing signed arithmetic are conditional on the stated width interpretation.

## Behavior

### Call identification

F-d001 through F-d108 identify call positions in execution order, not proven distinct implementations.

A **five-call chain** consists of five consecutive call positions. Only the first has the explicit arguments stated for that chain; the remaining four have no explicit arguments in the representation. The fifth supplies the result used afterward, when one is identified.

An absent explicit argument does not establish a zero-argument ABI. Additional register, memory, and global inputs remain unknown.

F-d099 denotes the loop-head call position. Its two target roles are identified separately:

- **F-d099-first**: the computed target used on the first iteration.
- **F-d099-repeat**: the different computed target used on every later iteration.

These are distinct represented target addresses. Their implementations and any relationship between them are unknown.

### Initial writes and entry-dependent calls

The first visible effect is a word write: the first argument is stored at four times that argument plus 20.

F-d001 then receives the first argument and B. Its target depends on the link-related entry value.

The subsequent calls occur in this order:

- F-d002 supplies its result to F-d003.
- F-d004 supplies its result to F-d005, whose second explicit argument is H.
- F-d006 through F-d009 follow without explicit arguments.

After F-d009 returns, the first argument is written as a word at B + 20.

The five-call chain F-d010–F-d014 receives four times R₉. Its final result determines another word write: at sixteen times R₁₄ plus 20, the operation stores four times R₁₄.

No visible check precedes these accesses.

### Entry-derived overwrite and secondary-result use

The five-call chain F-d015–F-d019 receives, in order:

1. B;
2. B again;
3. H;
4. The third argument logically shifted right by seven.

Afterward, B is written as a word at B + 20. This overwrites the address that earlier received the first argument, subject to any intervening dependency effects.

The five-call chain F-d020–F-d024 receives R₁₉ shifted left by two.

The next calculation uses a secondary result-register value, designated **S**. Its producer and ABI meaning are not reliably established. It must not be substituted with R₂₄ merely because it appears after that call.

Define **T** as S multiplied by 2²⁴ in the represented signed expression. The operation writes T at S multiplied by 2²⁶, plus 20.

The next chains and writes are:

| Chain | First explicit argument | Write after the chain |
|---|---|---|
| F-d025–F-d029 | T | Store T at four times R₂₉ plus 20. |
| F-d030–F-d034 | Four times R₂₉ | Store four times R₃₄ at sixteen times R₃₄ plus 20. |

Designate the value stored by the second row as **V**.

### Further indexed word writes

F-d035 receives V. F-d036 supplies its result to F-d037; F-d038 supplies its result to F-d039. F-d040 through F-d043 then execute without explicit arguments.

The operation stores V at four times R₄₃ plus 20.

The five-call chain F-d044–F-d048 receives four times R₄₃. It is followed by a word write at sixteen times R₄₈ plus 20, containing four times R₄₈. Designate the resulting unsigned word value as **W**. The product producing W is represented as signed before conversion, so its overflow interpretation remains unresolved.

F-d049 receives W. F-d050 supplies its result to F-d051; F-d052 supplies its result to F-d053. F-d054 through F-d057 follow without explicit arguments.

The operation stores J at four times R₅₇ plus 20.

The five-call chain F-d058–F-d062 receives four times R₅₇. Define **L** as sixteen times R₆₂. The operation writes four times R₆₂ at L + 20.

Two retained quantities now govern later accesses:

- **P** is W with its top bit cleared, logically shifted right by seven.
- J remains the entry-derived value E logically shifted right by sixteen.

The five-call chain F-d063–F-d067 receives L, L again, and H logically shifted right by seven. After that chain, the operation overwrites the word at L + 20 with L.

The five-call chain F-d068–F-d072 receives R₆₇ shifted left by two. Designate its final unsigned result as **X**.

### Repeated selector reads and byte writes

The operation reads the unsigned byte at P + 1. It adds 17 to that byte and writes the low byte of X shifted left by two at the resulting address.

F-d073 receives the 32-bit result of X shifted left by two in all three explicit argument positions. F-d074 supplies its result to F-d075. F-d076 through F-d079 follow without explicit arguments.

The operation then reads the byte at P + 1 again. At that newly read selector value plus 17, it writes the low byte of X masked to 30 bits and logically shifted right by 20.

This is a fresh read, not reuse of the first selector. Dependencies may have changed the selector. The earlier selected write may also have changed it through aliasing.

The five-call chain F-d080–F-d084 receives R₇₉ shifted left by two. Designate its final unsigned result as **Y**.

Two accesses then occur without an intervening opaque call:

1. Write the low byte of Y shifted left by two at J logically shifted right by seven, plus 28.
2. Read the byte at P + 1 and write zero at that selector value plus 17.

The first write can affect the selector used by the second access if the addresses alias.

### Transition into repetition

The following calls execute:

- F-d085 receives J logically shifted right by seven.
- F-d086 has no explicit arguments.
- F-d087 receives Y masked to 30 bits and logically shifted right by six.
- F-d088 through F-d091 have no explicit arguments.

The operation again reads the byte at P + 1 and writes zero at the selected address plus 17.

The five-call chain F-d092–F-d096 receives R₉₁ shifted left by two. Its final unsigned result, masked to 30 bits and logically shifted right by 25, supplies the initial **loop fragment**. This initial fragment lies between 0 and 31 inclusive.

A different selector address, designated **Q**, is then used. It is represented as the signed 32-bit interpretation of Y shifted left by two, multiplied by 1024, plus one. The multiplication’s signed-overflow behavior is unresolved. Under a wrapping 32-bit interpretation, Q corresponds to the low 32 bits of 4096 times Y, plus one.

The operation reads the unsigned byte at Q and writes zero at that byte’s value plus 17.

F-d097 receives twice the initial loop fragment. F-d098 then supplies the initial **carried result**.

The dispatch state entering the loop selects F-d099-first.

### Repeating region and target transition

Each iteration has the following observable order:

1. Execute the five-call chain F-d099–F-d103. F-d099 receives the carried result and the loop fragment logically shifted right by eight.
2. Read the unsigned byte at Q and write zero at that byte’s value plus 17.
3. Execute the five-call chain F-d104–F-d108. F-d104 receives R₁₀₃ shifted left by two.
4. Derive the next carried result from R₁₀₈ shifted left by two.
5. Derive the next loop fragment from R₁₀₈ shifted left by 26.
6. Begin the next iteration unconditionally.

The target used at step 1 depends on iteration position:

| Iteration | F-d099 target | Established selection |
|---|---|---|
| First | F-d099-first | Dispatch state established before entering the repeating region. |
| Second and every later iteration | F-d099-repeat | Dispatch state installed by the preceding iteration’s tail. |

During the tail’s final-call setup, the retained dispatch state is replaced with the value selecting F-d099-repeat. Consequently, the next loop head does not reuse the first-iteration target.

The later target selection is fixed by this visible tail update, not selected from R₁₀₈ or the next loop fragment. Both target addresses are computed, and they are different. No difference—or equivalence—in their implementations is established.

Q remains the same computed address throughout repetition, but its byte is reread on every iteration.

The initial loop fragment is at most 31, making the first F-d099 call’s second explicit argument zero. Later fragments originate in a represented signed left shift followed by conversion to unsigned. Under a 32-bit bit-pattern interpretation, their logical right shift by eight can be nonzero.

No visible condition exits the loop.

## Side effects

### Memory layout and access ordering

The visible accesses comprise computed-address 32-bit writes, unsigned byte selector reads, and byte writes.

The displacements 20, one, 17, and 28 establish address arithmetic only. They do not establish object headers, field layouts, array bounds, or ownership.

Repeated accesses must retain their order:

- B + 20 receives the first argument and later B.
- L + 20 receives four times R₆₂ and later L.
- P + 1 is read separately for each associated selector-based write.
- Q is read before the pre-loop zero write and once per loop iteration.

Calls intervene between many accesses. Addresses may also alias without intervening calls. Neither repeated stores nor selector reads may be collapsed on the assumption that memory remains unchanged.

No ordinary readback of the written words is visible outside dependencies.

### Ownership and external state

No explicit allocation, ownership transfer, retention, or release is identifiable.

No dependency is known to preserve globals, preserve arbitrary memory, retain arguments, or avoid releasing memory. Retained values in the visible dataflow do not establish a general callee-preservation guarantee.

Thread safety, reentrancy, synchronization, atomicity, and volatile-access semantics are unknown.

### Preconditions and errors

No explicit bounds check, null check, alignment check, error result, recovery path, or rollback is visible.

Continued execution requires each encountered target to be callable, each required memory access to be permitted, and each dependency to return when the next step is reached. These are execution prerequisites, not validations performed by this operation.

A failure after earlier writes may leave those writes and dependency effects in place.

## Dependencies

F-d001–F-d108 are opaque call-position IDs. The established dependency information is limited to call order, explicit argument relationships, visible result uses, target-selection relationships, and placement relative to memory accesses.

F-d099-first and F-d099-repeat identify the two known target roles at the loop-head call position. Distinct target addresses do not establish distinct underlying implementations. Conversely, unresolved implementation identity does not erase the established first-to-later target transition.

Unresolved dependency properties include:

- Complete ABI and argument counts.
- Additional register inputs and outputs.
- Register, global, and memory preservation.
- Exceptions and nonlocal transfers.
- Allocation, retention, and release.
- Whether a dependency returns.

The secondary-register value S particularly requires further calling-convention evidence.

## Acceptance examples

These are conditional static examples, not executed tests.

### Entry-derived overwrite

Assume the first argument is 3, E is 2, the relevant addresses are writable, and intervening dependencies return.

- The first word write stores 3 at address 32.
- B is 16, so the next entry-derived word write stores 3 at address 36.
- After F-d015–F-d019, the operation stores 16 at address 36.

Dependencies may produce additional effects between those writes.

### Distinct entry shifts

For E equal to hexadecimal 12345678:

- H is hexadecimal 00123456.
- J is hexadecimal 00001234.
- H logically shifted right by seven is hexadecimal 00002468.
- J logically shifted right by seven is hexadecimal 00000024.

Their argument and address roles are not interchangeable.

### Selector reread and aliasing

If P + 1 contains 5, the next associated byte-write destination is address 22.

If an intervening dependency changes that selector to 9, the following associated write uses address 26. Likewise, if a preceding byte write aliases P + 1, the next selector read observes the resulting byte, subject to the actual memory model.

### Byte truncation

For X equal to hexadecimal 00000123, the first selector-based byte write stores hexadecimal 8C.

For X equal to hexadecimal 3FF00000, masking to 30 bits and shifting right by 20 produces hexadecimal 3FF. The corresponding byte write stores hexadecimal FF.

### Loop-head dispatch

Assume all required accesses succeed and all dependencies return through three complete iterations.

- Iteration one invokes F-d099-first at the loop head.
- Its tail installs the selection for F-d099-repeat.
- Iterations two and three invoke F-d099-repeat.

Equal carried values or equal fragments across iterations do not restore F-d099-first. This example constrains target selection only, not target implementation behavior.

### Loop fragment boundaries

The first F-d099 call receives zero as its second explicit argument because the initial fragment is at most 31.

Under a 32-bit bit-pattern interpretation of the tail shifts:

- A tail result of 1 produces a fragment of hexadecimal 04000000 and a next head argument of hexadecimal 00040000.
- A tail result of 64 produces a zero fragment because its set bit shifts beyond the 32-bit width.

Both cases still transition to, or remain at, F-d099-repeat on the next iteration. Neither produces a visible loop exit.

## Uncertainties

- **documented_only — representation validity:** instruction-data and unreachable-region warnings limit confidence in correspondence to executable behavior.
- **documented_only — intent:** no higher-level operational purpose is established.
- **documented_only — entry state:** origins and required values of E and the link-related input are unknown.
- **documented_only — secondary result:** S has no verified producer or ABI interpretation.
- **documented_only — signed arithmetic:** overflowing products and left shifts require machine-level confirmation, including address calculations and loop-state updates.
- **documented_only — address model:** address width, mappings, alignment, byte order, and fault behavior are unknown.
- **documented_only — dependencies:** implementations and external effects are unresolved. The loop-head target transition itself is established.
- **documented_only — completion:** no normal exit is visible, but faults and dependency-controlled transfers may prevent indefinite repetition.

## Evidence

The evidence is the supplied static representation and its accompanying warnings. Prior narrative and reviewer feedback are not independent behavioral evidence.

The representation supports the described explicit inputs, access widths, call ordering, argument and result relationships, repeated selector reads, unconditional loop, and distinct first-versus-later loop-head targets. In particular, the dispatch value established before loop entry differs from the value installed during the iteration tail.

It does not establish dependency implementations, valid memory layouts, ownership rules, higher-level intent, or executable fidelity.

No runtime tests, executable inspection, or external-source verification have been performed.
