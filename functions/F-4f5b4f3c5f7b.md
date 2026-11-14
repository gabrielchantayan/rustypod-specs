<!-- Trusted provenance: F-4f5b4f3c5f7b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4f5b4f3c5f7b

## Purpose

The supplied static representation describes a procedure that interleaves indirect dependency calls with 32-bit and 8-bit memory accesses. It propagates explicit inputs, constructs scaled values from dependency results, repeatedly clears selected locations, and reaches an unresolved invalid-instruction boundary.

No application-level purpose is established. Object construction, device initialization, collection management, and resource release are not supported interpretations.

This contract covers the represented execution prefix, conditional on accurate instruction recovery. It does not establish normal return or behavior beyond the terminal boundary.

## Interface

### Explicit inputs

| Input | Represented width | Established use |
|---|---:|---|
| First input | 32 bits | Written unchanged to the first computed word destination. |
| Second input | 32-bit unsigned | Determines that destination, is passed to the first dependency, and supplies a masked-and-shifted stored value. |
| Third input | 32 bits | Written unchanged to two computed word destinations. |

The first and third inputs are uninterpreted bit patterns. Pointer, integer, handle, and packed-field interpretations remain unestablished.

### Implicit inputs and preconditions

Execution additionally depends on:

- Incoming register-like state used to derive the first indirect call target.
- Incoming register-like state used to calculate the first destination for the third input.
- Memory contents at computed word and byte addresses.
- Dependency results, including one represented 64-bit result.
- A secondary register-like value exposed after a later call.
- The execution environment’s address interpretation, alignment rules, byte order, and fault behavior.

Reaching the complete visible sequence requires every preceding memory access to succeed and every preceding dependency to return control. No visible validation establishes those conditions.

### Outputs and widths

There is no explicit return value. Outputs comprise direct memory writes, dependency invocations, and whatever effects those dependencies produce.

A **word** means 32 bits; a **byte** means 8 bits. Byte selectors are read as unsigned values from 0 through 255. A store described as retaining a low byte writes the low eight bits, not a word.

Address expressions below denote byte addresses. Scaling by four or sixteen does not establish an array or object layout.

The representation mixes unsigned 32-bit arithmetic with signed 32-bit intermediates. Unsigned masks, shifts, and multiplication have 32-bit bit-vector behavior in that representation; overflowing unsigned multiplication retains the low 32 bits. Signed multiplication and left shifts cannot uniformly be assigned portable overflow semantics. Converting calculated integers into usable addresses also requires environmental interpretation. Examples involving machine address wraparound are therefore conditional.

## Behavior

### Ordering and dependency notation

Identifiers such as **F-D01.1** denote opaque dependency call sites, not established callee identities. Distinct identifiers may reach the same implementation.

A **four-call sequence** means four consecutive indirect calls with no explicit arguments shown; its final result is used only where specified. This does not exclude implicit machine-level arguments.

The following order is the represented observable order, conditional on execution reaching each operation. Dependencies may modify memory or global state. Repeated loads are separate observations; saved values are not implicitly reloaded.

### Initial word propagation

The first direct operation writes the first input as a word at four times the second input plus `0x14`.

F-D01.1 receives the second input. F-D01.2 supplies a word passed to F-D01.3. The four-call sequence F-D01.4–F-D01.7 supplies a selector. At four times that selector plus `0x14`, the procedure writes the second input masked by `0x3fffffff`, then logically shifted right by one.

F-D02.1 receives four times that selector. F-D02.2–F-D02.5 supply a new selector. Define its **link value** as four times that new selector. The procedure writes the link value at sixteen times the new selector plus `0x14`.

It next reads a word at the link value plus `0x14`, logically shifts the loaded word right by three, and passes that result to F-D03.1. This load is not generally a read-back of the preceding store: their address expressions differ.

F-D03.2–F-D03.5 supply a destination selector. The saved link value is written at four times that selector plus `0x14`.

F-D04.1 receives four times the destination selector. F-D04.2–F-D04.5 supply the **primary selector**. Define:

- **Primary base:** sixteen times the primary selector.
- **Reduced selector:** the primary selector masked by `0x3fffffff`, then logically shifted right by one.
- **Reduced fragment:** the reduced selector masked by `0x3fffff`.

These names express arithmetic relationships, not application types.

### Third-input propagation and primary byte updates

The third input is written as a word at the incoming implicit destination selector multiplied by `0x80000`, plus `0x24`.

F-D05.1 receives the primary base and the primary selector shifted left by two. **Four** calls follow: F-D05.2, F-D05.3, F-D05.4, and F-D05.5. Only after F-D05.5 returns does the procedure write the third input at the primary base plus `0x24`. All four intervening calls may have observable effects.

The procedure then reads a word at the primary base plus `0x74` and passes it to F-D06.1. F-D06.2–F-D06.5 supply a saved **shift source**.

A byte read at the primary base plus one selects a destination by addition of `0x11`. The procedure writes there the low byte of the primary selector masked by `0x3fffffff` and logically shifted right by twenty.

The selector is a byte value used directly in address arithmetic. The selected destination is between `0x11` and `0x110`, inclusive; no word-sized pointer is loaded.

F-D07.1 receives the shift source shifted left by two. F-D07.2 supplies a value passed to F-D07.3. F-D07.4–F-D07.7 supply another selector.

The byte at the primary base plus one is read again. Zero is written at this newly read value plus `0x11`. Its destination may differ from that of the preceding byte store because dependencies intervene.

F-D08.1 receives the latest dependency selector shifted left by two. **Four** calls, F-D08.2–F-D08.5, follow before the next direct memory operation.

### Reduced-selector clearing and saved-byte addressing

After F-D08.5 returns, zero is written to the byte at the reduced selector plus `0x0f`.

Define **scaled base** as the reduced selector multiplied by `0x400`, retaining the represented unsigned 32-bit result. A byte read at the scaled base plus one selects another destination by addition of `0x11`; that byte destination is cleared.

F-D09.1 receives four arguments: the shift source shifted left by twenty-three in each of the first three positions, and the shift source shifted left by twenty-five in the fourth position. F-D09.2 supplies a value passed to F-D09.3. F-D09.4–F-D09.7 supply another selector.

The byte at the scaled base plus one is read again and retained as the **saved byte selector**. The byte at that selector plus `0x11` is cleared.

F-D10.1 receives the latest dependency selector shifted left by two. **Four** calls, F-D10.2–F-D10.5, follow. After the fourth returns, two word stores occur in this order:

1. Zero at the saved byte selector plus `0x44`.
2. Zero at absolute address `0x24`.

The saved byte selector is not reloaded after these calls. Its word-store destination ranges from `0x44` through `0x143`; natural word alignment is not guaranteed.

F-D11.1 receives, in order, zero, `0x98`, scaled base, and `0x98`. The argument pattern does not establish copying, filling, or allocation. F-D11.2 supplies a value passed to F-D11.3. F-D11.4 supplies a value passed to F-D11.5.

F-D11.6–F-D11.9 supply a selector. A word at four times that selector plus `0x44` is cleared, followed by another word clear at absolute address `0x24`.

### Split result and further word clears

F-D12.1 receives four times the preceding selector. F-D12.2–F-D12.5 follow, with F-D12.5 supplying a represented 64-bit result.

F-D13.1 receives the low 32-bit half shifted left by two. After that call returns, the procedure clears a word at twice the previously captured high 32-bit half plus `0x44`, then clears the word at absolute address `0x24`.

The halves are extracted from the saved result rather than reobtained from a dependency. Their subsequent signed interpretation and overflowing arithmetic remain subject to the representation’s limitations.

F-D14.1 receives zero, a secondary register-like value associated with F-D13.1, and that secondary value shifted left by two. The secondary value is not established as an ordinary declared return.

F-D14.2 is invoked without explicit arguments. F-D14.3 receives the reduced fragment shifted left by seven. F-D14.4–F-D14.7 supply another selector. The word at four times that selector plus `0x44` is cleared, followed by another clear at absolute address `0x24`.

There are thus four direct word clears at absolute address `0x24`: after F-D10.5, after F-D11.9, after F-D13.1, and after F-D14.7. Intervening dependencies may observe or alter that location; these are not interchangeable with one final clear.

F-D15.1 receives four times the latest selector. F-D15.2–F-D15.5 supply a new selector. F-D16.1 receives that selector shifted left by two. Its result selects a word at four times the result plus `0x14`, which is cleared.

### Link construction and reduced-fragment propagation

F-D17.1 receives the F-D16.1 result in each of its first three argument positions and the reduced fragment shifted left by seven in its fourth. F-D17.2–F-D17.5 supply a selector. The word at four times that selector plus `0x14` is cleared.

F-D18.1 receives four times that selector. F-D18.2–F-D18.5 supply another selector. Four times this new selector is written at sixteen times the selector plus `0x14`.

F-D19.1 receives the value just written, without a memory read-back. F-D19.2–F-D19.5 supply a destination selector. The same saved value is written at four times that destination selector plus `0x14`.

F-D20.1 receives four times the destination selector. F-D20.2–F-D20.5 supply a new selector. Four times this selector is written at sixteen times the selector plus `0x14`.

F-D21.1 receives four times this selector and sixteen times this selector. F-D21.2 supplies a value passed to F-D21.3 together with the reduced fragment shifted left by four.

F-D21.4–F-D21.7 supply another destination selector. The procedure writes the reduced fragment multiplied by `0x80` at four times that selector plus `0x14`.

The reduced fragment is at most `0x3fffff`. Its shifts by four and seven, including multiplication by `0x80`, fit within 32 unsigned bits. This does not establish that destination-address calculations fit or identify accessible memory.

### Final byte updates and termination boundary

F-D22.1 receives four times the latest selector. F-D22.2–F-D22.5 supply the **first final value**, represented as 32-bit unsigned.

Define **final base** as the reduced selector multiplied by `0x4000000`, retaining the unsigned 32-bit result. A byte read at the final base plus one selects a destination by addition of `0x11`. The procedure writes there the low byte of the first final value shifted left by two.

F-D23.1 receives the first final value shifted left by two. F-D23.2 supplies a value passed to F-D23.3. F-D23.4–F-D23.7 supply another selector.

The byte at the final base plus one is read again. At its value plus `0x11`, the procedure writes the low byte of the first final value masked by `0x3fffffff` and logically shifted right by twenty.

F-D24.1 receives the latest dependency selector shifted left by two. F-D24.2–F-D24.5 supply the **second final value**, represented as 32-bit unsigned.

Another byte read at the final base plus one supplies a saved selector. At that selector plus `0x11`, the procedure writes four times the low byte of the second final value, retaining the low eight bits. F-D25.1 then receives the saved selector zero-extended to 32 bits.

F-D25.2 supplies a value passed to F-D25.3. F-D25.4–F-D25.7 supply another selector.

The final direct memory update uses a different selector source: a byte is read at eight times the second final value plus one. At that byte’s value plus `0x11`, the procedure writes the low byte of the second final value masked by `0x3fffffff` and logically shifted right by twenty. The source-address arithmetic uses the represented unsigned 32-bit result.

F-D26.1 receives the latest dependency selector shifted left by two. **Four** calls follow, F-D26.2–F-D26.5. Only after F-D26.5 returns does represented control flow reach the invalid-instruction boundary. Reaching F-D26.4 does not establish completion of the visible call sequence.

## Side effects

Direct side effects are the ordered word and byte accesses above. Computed locations may alias each other, selector-source bytes, or memory affected by dependencies. No non-aliasing guarantee is supported.

No explicit allocation, deallocation, ownership transfer, reference-count adjustment, synchronization, or rollback operation is established. Dependencies may nevertheless perform any of those actions.

The contract does not grant ownership of computed locations or authorize their use as ordinary object fields. In particular, absolute address `0x24` and the low byte-selected destinations require a suitable memory environment.

No visible error-status inspection or recovery path is present. An inaccessible address, invalid indirect target, or nonreturning dependency prevents establishment of subsequent effects. Earlier writes are not shown to be undone.

## Dependencies

Every F-D identifier denotes an unresolved indirect call occurrence. No external callee identity is established.

The represented interfaces include zero through four explicit arguments, forwarded dependency results, loaded words, scaled selectors, masked fragments, a 64-bit result, and a secondary register-like output.

No dependency is established as pure, non-throwing, memory-preserving, global-preserving, ownership-preserving, or guaranteed to return. A result not subsequently used does not make its producing call removable. This applies specifically to the fourth calls F-D05.5, F-D08.5, F-D10.5, and F-D26.5.

## Acceptance examples

These are conditional static criteria, not executed tests.

1. **First store.** With second input three and a writable destination, the first input’s complete word pattern is written at `0x20` before F-D01.1.

2. **Masking.** A second input of `0xffffffff` produces `0x1fffffff` for the word store following F-D01.7.

3. **Distinct link addresses.** If F-D02.5 supplies two, the link value is eight. The store targets `0x34`; the following word load targets `0x1c`. They are not automatically the same location.

4. **Third-input ordering.** After F-D05.1, all four calls F-D05.2–F-D05.5 precede the third-input store at primary base plus `0x24`. An effect of F-D05.5 occurs before that store.

5. **Reload-sensitive selection.** A primary selector-source byte of `0x05` selects destination `0x16`. If intervening effects change it to `0x09`, the later clearing store selects `0x1a`.

6. **Reduced-clear ordering.** The clear at reduced selector plus `0x0f` follows F-D08.5, not merely F-D08.4.

7. **Saved selector and word ordering.** If the saved byte selector is `0xff`, the word clear targets `0x143` even if dependencies subsequently change its source byte. That clear follows F-D10.5 and precedes the clear at `0x24`.

8. **Byte truncation.** A first final value of `0x41` produces byte `0x04` in its first final byte store. The separately supplied shifted argument is `0x104`.

9. **Repeated absolute clears.** A store-sensitive observer must retain all four direct word clears at `0x24` in their specified positions.

10. **Conditional address wraparound.** Under 32-bit wrapping machine address interpretation, a second input of `0x40000000` selects initial destination `0x14`. This does not establish that the address is writable.

11. **Visible completion.** Returning from F-D26.4 still leaves F-D26.5 pending. Even reaching the terminal boundary after F-D26.5 does not establish normal return.

## Uncertainties

The following remain **documented_only**:

- Accuracy of instruction recovery and the represented function boundary.
- Complete calling convention and required incoming register state.
- Actual indirect targets and dependency effects.
- Address-space mapping, byte order, alignment enforcement, and volatility.
- Machine interpretation of overflowing signed arithmetic and integer-derived addresses.
- Meaningful object layouts, if any, behind the computed accesses.
- Origin and lifetime of the secondary register-like output.
- Ownership and resource lifetime across dependencies.
- Actual failure signaling and behavior beyond the invalid-instruction boundary.

Repeated scaled stores suggest structured state manipulation, but that is inferred intent, not an established data model.

## Evidence

**Established static evidence:** three explicit 32-bit inputs; ordered word and byte accesses; masks and shifts; dependency argument relationships; a represented 64-bit result split; four separate absolute-address word clears; four no-explicit-argument calls after each of F-D05.1, F-D08.1, F-D10.1, and F-D26.1; and a terminal invalid-instruction boundary.

**Evidence limitations:** the supplied representation reports problematic instruction recovery and overlapping global symbols. Its types, boundaries, and terminal behavior are not independently verified machine behavior.

**Inferred intent:** no application-level interpretation is required. “Selector,” “base,” and “link value” name arithmetic relationships only.

**Runtime verification:** no runtime tests have been performed.
