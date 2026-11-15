<!-- Trusted provenance: F-90c4ecaebbe5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-90c4ecaebbe5

## Purpose

F-90c4ecaebbe5 performs an ordered sequence of unresolved indirect calls, byte selections, scaled-address calculations, and byte or 32-bit memory stores. Results from dependencies determine later arguments, addresses, and stored values.

The supplied static evidence does not establish an application-level purpose. The stores do not, by themselves, establish initialization, allocation, serialization, resource destruction, or hardware control.

The recovered representation ends at an invalid-instruction boundary rather than a normal return. This contract describes the visible execution prefix, conditional on faithful recovery and on preceding operations completing. It does not establish behavior beyond that boundary.

## Interface

### Explicit inputs

Four 32-bit input slots are displayed:

| Input | Established role |
|---|---|
| First input | Passed to the first indirect call. Its signedness is not established. |
| Second input | No explicit consumption appears in the visible prefix. |
| Third input | No explicit consumption appears in the visible prefix. |
| Fourth input | Interpreted as unsigned, logically shifted right by three, and stored as a 32-bit word. |

The lack of explicit consumption of the second and third inputs does not establish their invisibility to dependencies through the underlying calling convention.

### Additional effective inputs

Execution also depends on machine state outside those four slots:

- An entry machine word used to form the first indirect call target.
- An **entry base**, used in address calculations and scaling.
- An **entry payload**, stored as a 32-bit word near the beginning.
- Memory contents at computed addresses.
- Dependency results, including a displayed 64-bit result and a value exposed through a secondary result register.

These effective inputs are not initialized by the visible prefix before their first use.

### Outputs and completion

No normal return or returned value is established. Observable behavior includes indirect calls, their explicit arguments, direct memory accesses, dependency effects, and the execution outcome at or before the terminal boundary.

No success status, exception class, termination mechanism, or guaranteed continuation is specified.

### Widths and arithmetic

Byte reads are unsigned and produce values from 0 through 255. Byte stores retain the low eight bits. Word accesses occupy 32 bits. One dependency result is displayed as 64 bits and is consumed as separate low and high 32-bit halves.

For compact address descriptions:

- \(S_k(x)\) denotes multiplication by \(2^k\), retaining the low 32 bits.
- \(H(x)=(x\mathbin{\&}\text{0x3fffffff})\gg20\).
- \(J(x)=(x\mathbin{\&}\text{0x3fffffff})\gg1\).
- \(T(x)=x\mathbin{\&}\text{0x003fffff}\).

The right shifts in these definitions are logical. Address additions below use the same apparent 32-bit bit-pattern model. Thus, for example, \(S_4(x)+\text{0x14}\) describes a sixteen-times-scaled address with a byte offset of 0x14, not an array index.

Some displayed expressions have signed intermediate types. The modulo-width notation describes apparent machine-width behavior; it does not establish portable source-language semantics for signed overflow or signed left shifts. Exact execution semantics remain **documented_only** where they depend on unresolved platform details.

No byte order, alignment guarantee, or accessible address range is supplied. Completing an access requires that the effective address support its stated width on the actual platform.

## Behavior

### Ordering model

The visible prefix contains no explicit branch or loop. Calls can nevertheless fail to return, and memory operations can interrupt execution.

A **five-call run** means five consecutive indirect calls in displayed order. Unless otherwise stated, only the first has explicit arguments shown; the relevant result is from the fifth. A **two-call sequence** similarly consists of a first call with stated arguments followed by a call with no explicit arguments shown; its relevant result is from the second.

These groupings describe ordering, not common dependency identity. No explicit arguments shown does not mean argument registers are cleared, nor does it establish a complete calling convention.

Each described memory read is a separate access at its stated point. In particular, reads separated by calls cannot be treated as interchangeable: dependency memory effects are unknown. Retained values below identify values carried forward in the recovered representation, rather than promises that dependencies preserve arbitrary globals or memory.

### Initial stores

The initial five-call run begins with the first input. Its result selects a destination at four times that result plus 0x14, which receives the entry payload as a 32-bit word.

A second run begins with that four-times-scaled address base. Afterward, the function stores \(S_{19}(\text{entry base})\) at \(S_{21}(\text{entry base})+\text{0x14}\).

A third run begins with three copies of that stored value. Its result selects a destination at four times the result plus 0x14. That destination receives the unsigned fourth input shifted right by three.

A fourth run begins with the preceding four-times-scaled address base. Its result is the **early result**. The function retains its four-times-scaled value as the **early scaled value**.

### Early byte selection

After those calls, an unsigned byte is read at entry base plus one. The low byte of the early scaled value is stored at the selected byte value plus 0x11.

The next five-call run begins with three arguments:

- That byte shifted right by three.
- The early scaled value.
- Four times the entry base.

Its final result will supply the four-times-scaled argument for the following run. Before that following run, however, entry base plus one is read again. This second byte is the **retained selector**. The function stores the low byte of \(H(\text{early result})\) at retained selector plus 0x11.

After the following run, a word at four times the retained selector plus 0x14 receives the early scaled value.

Another run begins with three copies of the retained selector and a fourth argument equal to that selector shifted right by one. Its final result selects a four-times-scaled destination plus 0x14, again receiving the early scaled value.

The retained selector is the second byte read. Later changes to its original memory location do not constitute a refresh of this retained value.

### Scaled-address propagation

The next three runs propagate words through distinct scale relationships:

| Run | First-call argument | Store after the run |
|---|---|---|
| First | Four times the preceding result | Four times the new result, at sixteen times that new result plus 0x14 |
| Second | The word just stored | That same word, at four times the new result plus 0x14 |
| Third | Four times the preceding result | Four times the new result, at sixteen times that new result plus 0x14 |

Retain the third run’s stored word as the **propagated word**.

A two-call sequence begins with the propagated word. Its second result becomes the first argument of a five-call run. That run’s result selects a destination at four times the result plus 0x14, receiving \(J(\text{propagated word})\).

The next run begins with that destination’s four-times-scaled base. Its result produces another store: four times the result is written at sixteen times the result plus 0x14.

A subsequent unsigned word read instead uses four times this result plus 0x14. This is not generally the address of the immediately preceding store. The read value, logically shifted right by three, becomes the first argument of another run. Its result selects a four-times-scaled destination plus 0x14, receiving the previously calculated four-times-scaled value.

### Main result and selector-dependent state

Another run begins with the preceding destination’s scaled base. Its result is the **main result**.

The retained selector is stored as a 32-bit word at:

\[
S_{19}(\text{retained selector}\gg3)+\text{0x24}.
\]

The function retains sixteen times the main result as the **main base**, and \(J(\text{main result})\) as the **derived quantity**.

A run begins with the main base and four times the main result. After that run, main base plus 0x24 receives the retained selector as a word.

The function then reads a word at main base plus 0x74. That read value is the first argument of another run, whose result is the **late shift result**.

A fresh byte read at main base plus one selects a destination at the byte value plus 0x11. This destination receives the low byte of \(H(\text{main result})\).

A two-call sequence begins with four times the late shift result. Its second result feeds a five-call run. Afterward, main base plus one is read again, and the newly selected byte destination at offset 0x11 receives zero. Dependencies may have changed the selector; the two destinations are not required to match.

### Byte clearing and retained clear selector

A five-call run begins with four times the preceding run’s result. After it completes, the function clears one byte at derived quantity plus 0x0f.

The **clear base** is \(S_{10}(\text{derived quantity})\). A byte read at clear base plus one then selects another byte destination, at the selected value plus 0x11, which receives zero.

The next two-call sequence begins with four arguments: three copies of \(S_{23}(\text{late shift result})\), followed by \(S_{25}(\text{late shift result})\). Its second result feeds a five-call run.

After that run, clear base plus one is read again. This byte becomes the **clear selector**, and clear selector plus 0x11 is cleared.

Another run begins with four times the preceding result. Afterward, a 32-bit zero is stored at clear selector plus 0x44, followed by a 32-bit zero at absolute address 0x24.

The word destination uses the retained clear selector. It does not involve another read of clear base plus one.

### Repeated word clearing and wide result

A four-call sequence follows. Its first call receives zero, 0x98, the clear base, and 0x98. The second call’s result is supplied to the third; the fourth call’s result feeds a five-call run.

That run’s result selects a word destination at four times the result plus 0x44. It receives zero, followed by another zero word store at absolute address 0x24.

The next run begins with the preceding four-times-scaled base and produces the displayed 64-bit result. Four times its low 32 bits is passed to an additional indirect call. After that call, a word at twice the retained high 32 bits plus 0x44 receives zero. Absolute address 0x24 then receives its third explicit zero word store.

The next call receives zero, a secondary result-register value exposed by the preceding additional call, and four times that secondary value. One more call follows. No meaning or ownership is established for the secondary value.

Another five-call run begins with \(S_7(T(\text{derived quantity}))\). Its result selects a word destination at four times the result plus 0x44, receiving zero. Absolute address 0x24 then receives its fourth explicit zero word store.

A run begins with the preceding four-times-scaled base. Four times its result is passed to an additional call. That additional call’s result selects a word destination at four times the result plus 0x14, receiving zero.

### Final word propagation

The next run begins with three copies of that additional call’s result and a fourth argument of \(S_7(T(\text{derived quantity}))\). Its result selects another four-times-scaled destination plus 0x14, receiving zero.

Three more runs reproduce the earlier propagation pattern:

- The first begins with the preceding scaled base, then stores four times its result at sixteen times that result plus 0x14.
- The second begins with that stored word, then stores the same word at four times its own result plus 0x14.
- The third begins with the preceding scaled base, then stores four times its result at sixteen times that result plus 0x14.

A two-call sequence begins with four times and sixteen times the last result. Its second result feeds a five-call run whose first call additionally receives \(S_4(T(\text{derived quantity}))\).

That run’s result selects a destination at four times the result plus 0x14. The stored word is \(S_7(T(\text{derived quantity}))\).

### Final byte operations and boundary

A run begins with the preceding scaled base. Its result is the **first final result**.

The function reads a byte at \(S_{26}(\text{derived quantity})+1\), then writes the low byte of four times the first final result at the selected byte value plus 0x11.

A two-call sequence beginning with four times the first final result feeds another run. The same selector address is then reread. Its newly selected destination receives the low byte of \(H(\text{first final result})\).

A run begins with four times the preceding run’s result. Its result is the **second final result**. The same selector address is read a third time, and the selected destination receives the low byte of four times the second final result.

A two-call sequence now begins with that third selected byte value—not the second final result or its scaled value. Its second result feeds another five-call run.

Afterward, a different selector address is read: eight times the second final result plus one. The byte destination it selects, at offset 0x11, receives the low byte of \(H(\text{second final result})\).

A final five-call run begins with four times the preceding run’s result. The recovered prefix then ends at the invalid-instruction boundary. No normal return is visible.

## Side effects

All accesses and calls are conditional on execution reaching them.

No explicit rollback operation is shown in the recovered prefix. Earlier effects may remain observable after interrupted execution; whether dependencies restore or compensate for them is unknown.

The direct memory effects have these properties:

- Selector-plus-0x11 byte destinations range from 0x11 through 0x110.
- Clear-selector-plus-0x44 word starting addresses range from 0x44 through 0x143. Their four-byte extents can reach 0x146.
- Selector-derived word destinations need not be aligned.
- Absolute address 0x24 receives four explicit 32-bit zero stores, separated by calls and other accesses.
- Separate address expressions may identify overlapping or identical memory.
- A byte store may change a later word access through overlap.
- A store may also affect a later selector read if their addresses overlap.
- Distinct reads remain distinct even when their address expressions are equal.

The direct word stores express neither atomicity nor synchronization guarantees.

No allocation, release, ownership transfer, reference counting, or resource lifetime is established. An argument’s numerical appearance does not determine whether it is a pointer, handle, size, or ordinary scalar. Dependencies may have additional memory, global-state, ownership, or compensating effects.

## Dependencies

The only supplied opaque function identity is **F-90c4ecaebbe5**. No separate opaque IDs are provided for its indirect dependencies. Assigning identities to those targets would fabricate provenance.

Dependency identity and implementation contracts are therefore **documented_only**.

Established call-site evidence includes:

- The first target depends on an entry machine word.
- Subsequent targets are formed from displayed anchor values and offsets, including some pointer-form anchors.
- Repeated five-call patterns establish neither shared targets nor interchangeable behavior.
- Most consumed results are displayed as 32 bits; one is displayed as 64 bits.
- One subsequent call consumes a value exposed through a secondary result register.
- Arguments and result relationships are established only as described in the visible prefix.

No dependency is established to return normally, preserve globals, preserve addressed memory, retain or release resources, or refrain from restoring earlier modifications.

## Acceptance examples

These are conditional static examples, not executed tests. They assume the relevant prefix is reached and the stated accesses complete.

1. **Unsigned input transformation.** A fourth input of 0xffffffff produces a stored word of 0x1fffffff at its designated destination.

2. **Byte versus word truncation.** An early result of 0x41 gives an early scaled value of 0x104. The first selected byte destination receives 0x04; later word stores of that retained scaled value receive 0x00000104.

3. **Masking before extraction.** For a result of 0xffffffff, \(H\) is 0x3ff, so its byte store receives 0xff. \(J\) is 0x1fffffff.

4. **Fresh early selector.** If entry base plus one first yields 0x20 and later yields 0x30, the corresponding byte destinations are 0x31 and 0x41. Reusing the first read for both destinations would not match the visible accesses.

5. **Retained selector.** If the second early selector is 0x30, its later four-times-scaled word destination is 0xd4. A subsequent change to the original selector memory does not itself change that retained destination.

6. **Different address scales.** For a propagation result of two, sixteen times the result plus 0x14 is 0x34, whereas four times the result plus 0x14 is 0x1c. A read from 0x1c cannot be assumed to retrieve the word stored at 0x34.

7. **High half is not replaced by a later result.** If the displayed 64-bit result has high half 3, the associated post-call clearing destination is 0x4a under the stated width model. The additional call may have other effects, but its ordinary result does not select this destination.

8. **Low-byte wraparound.** For a second final result of 0x40, the corresponding four-times byte store receives 0x00. The separate selector read based on eight times that result uses address 0x201.

9. **Interrupted execution permits compensation.** If a dependency is entered after a word store, then restores that word before failing or not returning, this contract does not prohibit the restoration. If no restoration occurs, the earlier store may remain observable. Neither outcome is guaranteed.

10. **No completion implication.** Reaching the final call sequence does not establish return to the caller, success, or a particular fault at the terminal boundary.

## Uncertainties

The following remain **documented_only**:

- Whether the recovered control flow faithfully represents executable behavior.
- Application purpose, data structures, and object boundaries.
- Complete calling convention and hidden inputs.
- Dependency identities, memory effects, ownership rules, compensation, and failure behavior.
- Exact platform arithmetic where signed expressions or address conversion matter.
- Memory accessibility, alignment rules, byte order, and memory type.
- Whether low addresses are ordinary memory, mapped devices, or invalid.
- Atomicity, concurrency interaction, and visibility guarantees.
- Exact behavior at and beyond the invalid-instruction boundary.

An initialization-like interpretation is possible inferred intent, not an established requirement. Repeated clearing also does not establish that affected locations remain zero after later dependency effects.

## Evidence

This contract uses only the supplied static representation: displayed input widths, effective entry-state consumption, access widths, arithmetic relationships, explicit call arguments, result consumers, operation ordering, and the terminal control-flow boundary.

The representation warns about invalid instruction data and overlapping global symbols. These warnings limit confidence in recovered control flow and prevent treating named storage as necessarily independent objects.

No dependency bodies, verified dependency identities beyond the subject function, runtime traces, platform memory map, or complete calling-convention documentation were supplied. No runtime tests have been performed.
