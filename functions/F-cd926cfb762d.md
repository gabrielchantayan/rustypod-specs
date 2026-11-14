<!-- Trusted provenance: F-cd926cfb762d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cd926cfb762d

## Purpose

The established behavior is an unconditional sequence of computed calls, packed-value transformations, and memory updates. The function writes and clears 32-bit slots, uses bytes as address selectors, copies the low byte of several 32-bit loads, publishes values to a global word, and finally attempts a call through address zero.

No application-level purpose is established. Allocation, initialization, queue management, supervisor operations, and cleanup are not justified descriptions without dependency contracts and an address-space model.

This contract describes static evidence only. It does not establish that the function can complete in an ordinary process environment.

## Interface

### Explicit inputs and result

The displayed interface has three 32-bit inputs:

- **Input A:** an uninterpreted value forwarded to the initial computed call.
- **Input B:** an uninterpreted value with no explicit use in the visible body.
- **Input C:** an unsigned value whose upper nine bits participate in a word store and a selector lookup.

There is no declared return value. The visible return is reached only if the final zero-address call returns.

The absence of an explicit use of Input B does not prove that dependencies cannot observe its incoming machine state. Displayed argument lists are not complete calling-convention specifications.

### Ambient inputs

Execution also depends on:

- An incoming link-register value used to construct the first target.
- An unexplained incoming 32-bit register value used as a stored payload.
- Computed-call results, including three results represented as 64-bit quantities.
- Several distinct secondary-register outputs.
- A byte at absolute address 1.
- A global 32-bit control location.
- A global pointer-like value.
- A separate three-byte global value associated with address 9.

The secondary-register outputs must remain distinct in this contract:

- **Early secondary value:** an unsigned 32-bit value used for an early slot clear, a special-call argument, and a later masked word-address calculation.
- **Four-argument secondary value:** a separate 32-bit output repeated in the first three positions of a subsequent special call.
- **Retained secondary value:** a later signed 32-bit output captured for repeated and shifted arguments.
- **Final secondary pointer:** a later byte pointer used for the final byte clear.

Equality between these values is not established. In particular, the four-argument secondary value is not shown to be the early secondary value.

### Widths and address conventions

A **word slot** is a 32-bit location at address `4 × index + 20`. A **wide-stride slot** is a 32-bit location at `16 × index + 20`; the stored payload remains 32 bits.

A **selector** is an unsigned byte loaded from memory and zero-extended. A selector-directed destination is at `selector + 17`, ranging from address 17 through 272. The selector is not a pointer-sized value and does not trigger another pointer dereference.

“Low byte” means the least significant eight bits of a value. Splitting a 64-bit result into low and high halves means numerical extraction of its lower and upper 32 bits, not a claim about memory byte order.

Unsigned 32-bit arithmetic is subject to its width. Some index arithmetic and left shifts instead operate on displayed signed 32-bit values. Overflow and invalid signed shifts are not established as portable operations; unrestricted wrapping behavior must not be assumed for them.

## Behavior

### Execution and dependency notation

There are no visible loops or conditional branches. The following order applies if every preceding call returns and every preceding access completes.

Opaque dependency IDs identify call classes, not necessarily individual functions:

- **F-000000000001:** the initial link-relative call.
- **F-000000000002:** the recurring computed calls using an anchor displacement of 4,094.
- **F-000000000003:** the other nonzero computed calls.
- **F-000000000004:** the final zero-address call.

Calls within one class may reach different targets and have different conventions. Intervening calls remain part of the behavior even when their results are unused. They cannot be discarded, merged, or treated as memory-preserving operations.

### Initial forwarding and early slot writes

F-000000000001 is called with Input A at a target formed from the incoming link register plus 3,896.

An F-000000000002 result is forwarded to another call in that class. Input C is logically shifted right by 23 to obtain a nine-bit value.

After further F-000000000002 calls, a returned index selects a word slot. The unexplained incoming register payload is stored there as a complete 32-bit value. Four times that index, without the 20-byte slot displacement, is then passed to a dependency.

Following more calls, the early secondary value selects another word slot, which is cleared to zero. That same early secondary value is passed to F-000000000003.

These writes are separate events; their destinations are not established to be different.

### Upper-bit storage and first selector-directed write

An ordinary result is obtained and forwarded. Further calls produce an index whose word slot receives the upper nine bits of Input C.

The function then reads a 32-bit word at:

`20 + ((early secondary value AND 0x1fffffff) logically shifted right by 19)`.

The offset is between zero and 1,023, without multiplication by four. The word access can therefore be unaligned.

After intervening calls, the loaded word is logically shifted right by 22, leaving a ten-bit value. The function reads a selector at `1 + (Input C logically shifted right by 23)`, then writes the low byte of the ten-bit value at `selector + 17`.

The selector source ranges from address 1 through 512.

A subsequent F-000000000003 call receives, in order:

1. The four-argument secondary value.
2. That same four-argument secondary value.
3. That same four-argument secondary value.
4. Zero.

This is a separate secondary-register output from the early secondary value used for the slot clear and masked word load. The visible representation does not establish equality, continuity, or interchangeability between them.

### Selector-directed clears and retained secondary state

Further calls include an explicit zero argument obtained by logically shifting a locally established zero.

After more calls, the function reads a selector at `1 + 4 × ten-bit value`, using the ten-bit value from the preceding phase, and clears the byte at `selector + 17`. The selector source ranges from 1 through 4,093 in increments of four.

Subsequent dependencies receive a four-times-scaled index once and another four-times-scaled index twice. Another 32-bit result is obtained and forwarded.

The function then reads the byte at absolute address 1 and clears the byte at that selector plus 17. It captures the retained secondary value and passes that value three times to F-000000000003.

A later dependency receives a newly obtained result together with the upper nine bits of the previously obtained 32-bit result.

Following more calls, the function reads address 1 again, retains this newly read selector, and clears the byte at that selector plus 17.

These are two distinct reads of address 1. Neither the first read nor the first selector-directed clear fixes the value of the second read: intervening calls or overlapping writes may change it.

### Global publication and repeated word-to-byte copies

Further calls receive scaled indices. A later call produces a 32-bit value that is written in full to the global control location before being forwarded to another dependency.

After another call, the function uses the selector retained from the second address-1 read to:

- Read a 32-bit word at `retained selector + 116`.
- Write that word’s low byte at `retained selector + 1`.

The word source ranges from address 116 through 371. No alignment guarantee is established.

F-000000000003 then receives the retained secondary value shifted left by 11.

A subsequent call returns a represented 64-bit quantity. The next call receives its low 32 bits, its high 32 bits, and the upper nine bits of the locally retained value previously published to the global control location. The third argument is not a fresh global read.

Further calls produce an index. Four times this index becomes a retained base. The function reads a word at `base + 116`, stores its low byte at `base + 1`, and passes the base to a dependency.

Another call sequence produces a second index and a separately retained four-times-scaled value. The function nevertheless performs the next word-to-byte copy using the **previous base** again. It freshly reads `previous base + 116` and writes the resulting low byte at `previous base + 1`.

F-000000000003 then receives:

1. The retained secondary value shifted left by 12.
2. The second index scaled by four.
3. The same scaled value again.
4. The second index shifted left by 18.

### Full-word propagation and link-like writes

A result is obtained, forwarded through a special call, followed by another obtained result and ordinary forwarding.

Further calls produce an index. The function reads a 32-bit payload at `4 × index + 116`, writes its low byte at `4 × index + 1`, and passes the scaled index to a dependency. The complete payload is retained for later use.

A subsequent index selects a wide-stride slot, which receives that complete payload.

A special call receives the earlier separately retained scaled value logically shifted right by two and then left by 16. The operation must not be simplified to an unrestricted transformation of the original index: scaling may already have discarded high bits under the applicable arithmetic interpretation.

After more result forwarding and calls, the same retained payload is written to a conventional word slot.

Further calls produce another index. Its four-times-scaled value is stored as a 32-bit word at:

`(earlier retained scaled value AND 0xfffffffc) + 20`.

A special call receives the newly scaled value twice.

Another forwarding sequence follows, after which a newly selected word slot is cleared to zero. Its scaled index is passed onward before the next result-producing sequence.

### Paired-result stores and global-derived payload

A second represented 64-bit result is obtained. Its low half is shifted left by two; its high half selects a word slot. The scaled low half is written to that slot.

The function then reads and retains the global pointer-like value. The following call receives the scaled low half, the unscaled high half, and the scaled low half twice more.

After additional calls, another word slot receives that same scaled low half. The function then reads a 32-bit word at `retained global pointer + 20`. The pointer is the earlier captured value, not a refreshed read of the global pointer location.

Following additional calls, a third represented 64-bit result is obtained. Its low half becomes a new packed value; its high half supplies repeated arguments.

Before the next special call, the function reads a selector at:

`1 + (retained full-word payload logically shifted right by 23)`.

It writes the low byte of the global-derived word at `selector + 17`.

The special call receives the latest high half three times, followed by the latest low half shifted left by two. This last shift is displayed on an unsigned 32-bit value.

### Final selector write and termination attempt

After another forwarding sequence and further calls, the function obtains a selector from:

`1 + ((new packed value AND 0x3fffffff) logically shifted right by 21)`.

The selector source ranges from address 1 through 512. The destination at `selector + 17` receives the low byte of the global-derived word logically shifted right by 22.

Further calls receive a scaled index once and a later scaled index twice. An additional ordinary call is followed by a differently offset computed call producing a 32-bit result, which is forwarded through an ordinary call.

The last ordinary call produces an 8-bit result and exposes the final secondary pointer. The remaining visible effects occur in this order:

1. Clear one byte through the final secondary pointer.
2. Write the global control word with a composite numerical value: the final 8-bit result supplies the low byte, and the separate three-byte global value supplies the upper 24 bits.
3. Clear a 16-bit location at `(new packed value AND 0x3fffffff) logically shifted right by 7`.
4. Call F-000000000004 at address zero with three displayed zero arguments.
5. Return only if that call returns.

The halfword address ranges from zero through 8,388,607. It is not guaranteed to be even.

## Side effects

Explicit accesses include byte, halfword, and word stores, word loads, selector-byte loads, and global accesses. Computed addresses include very low addresses; no valid heap object, allocation extent, or bounds guarantee is established.

The initial full-word global publication and final composite publication are distinct observable events. Dependencies execute between them and may observe or modify global storage. The function nevertheless uses some locally retained earlier values rather than refreshing them from globals.

The static representation warns of overlapping global views. Accordingly:

- Word stores may change byte-sized views.
- Selector-directed byte writes may change later word loads.
- Different symbolic descriptions do not prove disjoint storage.
- Distinct calculated addresses may alias.
- Fresh reads must remain fresh even when their addresses repeat.

No explicit allocation, release, reference-count adjustment, or ownership transfer is identifiable. Dependencies may perform such operations; their behavior is unknown.

There is no visible validation branch, error return, rollback, or recovery handler. A failing access or non-returning dependency prevents later operations under the applicable runtime rules. Earlier effects are not visibly undone.

## Dependencies

F-000000000001 uses incoming control-flow state to form its target.

F-000000000002 groups recurring computed calls with displacement 4,094. Displayed arguments and result widths vary, including paired-word results.

F-000000000003 groups other nonzero computed calls. Observed displacements include 3,896, 4,076, 3,976, 4,016, 3,988, 3,996, 3,984, and 176. Group membership does not establish common identity, semantics, or calling convention.

F-000000000004 is an explicit call through address zero. Its target is not established as executable memory, an invalid address, a firmware entry, or an intentional trap.

For all dependency classes, global preservation, memory ownership, release behavior, and return behavior are unknown. Displayed arguments do not establish the complete machine-level interface. Distinct secondary-register outputs cannot be conflated merely because they occupy the same architectural register at different points.

## Acceptance examples

These are conditional static expectations, not executed tests. Each assumes preceding operations complete and the applicable arithmetic and memory accesses are supported.

1. **Input boundary:** Input C equal to zero gives upper-nine-bit value zero and selector source address 1. Input C equal to `0xffffffff` gives value 511 and selector source address 512.

2. **Distinct secondary outputs:** If the early secondary value is 4, its early clear targets address 36 and its masked word load targets address 20. If the separate four-argument secondary value is 9, the subsequent special call receives 9, 9, 9, and 0—not 4, 4, 4, and 0.

3. **Byte truncation:** A first masked-load payload of `0xffffffff` becomes 1,023 after shifting right by 22. The selected byte receives 255.

4. **Selector interpretation:** A selector byte of 200 directs the selected-field write to address 217. No pointer-sized load replaces that byte selection.

5. **Repeated fresh load:** If an intervening dependency changes the word at the retained base plus 116, the second copy to the retained base plus 1 uses the changed word’s low byte.

6. **Payload preservation:** A retained payload of `0x123456ab` produces byte `0xab` in the nearby byte copy and complete value `0x123456ab` in later wide-stride and conventional word-slot stores.

7. **Composite publication:** Upper-three-byte value `0x123456` and final byte result `0xab` produce numerical control value `0x123456ab`. This does not specify physical byte order.

8. **Packed-value boundary:** New packed value `0xffffffff` selects final selector source address 512 and final halfword-clear address 8,388,607.

9. **Aliasing and failure:** Coincident destinations do not permit removing either write. If the zero-address call fails, the preceding pointer-byte clear, composite global write, and halfword clear have already been attempted in that order.

## Uncertainties

The following are **documented_only**, not acceptance guarantees:

- Application-level intent and object types.
- Exact dependency identities, signatures, and calling conventions.
- Origins and validity of ambient register values.
- The dependency-level production rules for secondary-register outputs.
- Any equality between distinct secondary-register outputs.
- Behavior of signed overflow and invalid signed left shifts.
- Address wrapping, endianness, alignment support, and permissions.
- Whether low addresses represent memory, hardware, or representation artifacts.
- Exact physical overlap of global views.
- Volatility, concurrency, interrupts, and external memory-order requirements.
- Dependency-induced allocation, release, or ownership changes.
- Successful execution or return of the zero-address call.

The environment must support each call and access as reached. The function does not visibly establish those prerequisites.

## Evidence

**Established static evidence:** three 32-bit explicit inputs; no declared return value; an unconditional sequence of computed calls; three represented 64-bit results; distinct secondary-register outputs; explicit byte, halfword, and word operations; packed-value masks and shifts; global accesses; and a final zero-address call.

The early secondary value is reused for the early slot clear and masked word-address calculation. A separately represented secondary output supplies the three repeated arguments of the later four-argument special call. Their equality is not established.

**Inferred intent:** repeated scaling and fixed offsets suggest structured storage; upper-bit extraction and selector tables suggest packed representations. Neither inference establishes domain semantics.

**Unknowns:** missing dependency contracts and execution-environment details prevent stronger claims about validity, ownership, error behavior, or successful completion.

No runtime tests have been performed.
