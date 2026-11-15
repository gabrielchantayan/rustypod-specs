<!-- Trusted provenance: F-dc2ffa5794cb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dc2ffa5794cb

## Purpose

F-dc2ffa5794cb forwards an explicit input to an indirect dependency, performs a long ordered sequence of dependency invocations, and interleaves byte and 32-bit memory accesses. The supplied representation ends at an invalid-instruction boundary without a visible normal return.

No application-level purpose is established. Repeated scaling and bit extraction are consistent with manipulation of encoded quantities, but serialization, object construction, allocation, device configuration, and cleanup are not established purposes.

This contract distinguishes:

- **Established static evidence:** represented calls, dataflow identities, memory-access widths, and sequencing.
- **Conditional arithmetic consequences:** results under the 32-bit wrapping model stated below.
- **documented_only unknowns:** unresolved execution semantics, dependency contracts, and validity of recovered implicit-register dataflow.

The described behavior is conditional on execution reaching each operation. It is not proof that the entire represented sequence executes successfully.

## Interface

### Explicit input

One explicit argument is represented as a 32-bit quantity. Its signedness, semantic type, and permitted values are unknown.

The first indirect invocation receives that quantity unchanged. No subsequent direct use of the explicit argument is visible. The first invocation can nevertheless affect later execution through memory or other state.

### Implicit inputs and represented values

The sequence also depends on:

- An incoming return-link value used to construct the first indirect target.
- An incoming unsigned 32-bit **address seed**, used in early address calculations and logical shifts.
- An incoming 32-bit **stored word**, written near the beginning.
- Three separately represented unsigned 32-bit **secondary values** whose producers are unresolved.
- Results of indirect invocations, including one represented 64-bit result.
- Bytes and words read from calculated addresses.
- State observed or changed by dependencies.

The three secondary values have distinct roles:

| Represented value | Uses sharing that value |
|---|---|
| First secondary value | Middle argument of a three-argument invocation; later byte payload obtained by shifting right by 22 |
| Second secondary value | Word-store address selection; subsequent invocation arguments; later stored word |
| Third secondary value | Repeated invocation arguments; sixteen-times selector address; shifted byte payload |

Within each row, the uses share one represented value. Intervening calls do not introduce represented replacements for it. This establishes dataflow identity in the supplied representation, not the actual producer, ABI, or correctness of recovered register lifetimes.

### Output and termination

No public return value is declared, and no normal return is visible.

The 64-bit dependency result is an intermediate value. Its high and low 32-bit halves are used separately; it is not a return value from F-dc2ffa5794cb.

The terminal invalid-instruction indication establishes neither a recoverable error nor a particular hardware exception. Actual terminal behavior is **documented_only**.

### Execution requirements

For any prefix to complete, reached dependencies must be callable and return as needed, and reached memory accesses must be valid for their widths and access modes. Required incoming register state must also exist.

No explicit null, range, alignment, allocation-success, or dependency-error checks are visible. These requirements are not validated guarantees.

## Behavior

### Widths, arithmetic, and access layout

The representation uses 8-bit bytes, 32-bit quantities, and one 64-bit result.

The numerical consequences below assume 32-bit address arithmetic, multiplication, addition, and left shifts retain their low 32 bits. Right shifts of represented unsigned quantities are logical. Byte stores retain eight bits; selector-byte reads are zero-extended. Word stores preserve all 32 bits.

Some intermediates are represented as signed quantities. Wrapping is a conditional machine-arithmetic interpretation, not a claim that overflowing signed source-language arithmetic is defined.

Two recurring layouts are:

- **Scaled-word destination:** numerical address equal to four times a selecting value, plus 20 bytes; accessed as a 32-bit word.
- **Byte-selected destination:** read a selector byte, then write one byte at the selector’s numerical value plus 17.

The byte-selected destination is therefore between numerical addresses 17 and 272 inclusive. It is not represented as a field relative to the selector’s containing object.

Endianness, alignment requirements, object boundaries, and address-space interpretation are unknown. Separate represented reads remain separate even when their addresses are equal.

### Initial forwarding and first word store

The first indirect invocation receives the explicit input. Its target is constructed from the incoming return-link value.

Four more indirect invocations follow. The last returns a selecting value. The incoming stored word is then written at that value’s scaled-word destination.

The next invocation receives four times that selecting value. Four further indirect invocations occur before the next visible memory access; the last supplies the first packed value.

Calls without displayed arguments remain observable invocations. They must not be eliminated or treated as pure.

### First packed-byte group

The sequence:

1. Reads a selector byte at the address seed plus one.
2. Writes the low byte of four times the first packed value to the selected destination.
3. Invokes a dependency with the address seed shifted right by 12.
4. Executes intervening indirect invocations.
5. Reads the selector address again.
6. Writes packed bits 20 through 27 to the newly selected destination.

The second payload is represented by masking the packed value to 30 bits, shifting right by 20, then truncating to a byte. Bits 28 and 29 do not survive byte truncation.

The two selector reads are distinct. Intervening dependencies can change memory, so the selected destinations need not agree.

Subsequent invocations receive a returned value shifted left by two and, later, another returned value shifted left by two together with the retained second selector.

### Collapsed-address group and first secondary value

The address seed is shifted left by 26. Two separate 32-bit reads at that shifted value plus 28 supply two argument positions of one invocation. Their relative evaluation order is not established.

Multiplying the shifted value by 1,024 produces a further address component. Under 32-bit wrapping, this component is always zero: the combined left shift exceeds the available width.

The following sequence is represented:

- After another invocation, read a selector at that component plus one and write zero to its selected destination.
- Invoke a dependency with zero, the **first secondary value**, and the component.
- Execute intervening invocations.
- Read the same selector address again.
- Write the low byte of the **same first secondary value** shifted right by 22.

Thus, under wrapping, the selector reads use numerical address 1, and the three-argument invocation receives zero in its first and third positions.

The middle argument and later shifted byte payload share one represented unsigned 32-bit value. Its producer and recovered register lifetime remain uncertain; the representation does not assign a distinct replacement between these uses.

Later calls receive a newly returned value shifted left by two and, subsequently, the collapsed component shifted left by 26 together with another returned value shifted left by two.

### Direct dependency and second secondary value

An indirect result is retained and passed twice to F-674e75a953e8. Call this the **retained direct-call input**; the name describes its use, not an application meaning.

After that direct invocation:

1. One indirect invocation occurs.
2. The retained direct-call input is stored at the **second secondary value’s** scaled-word destination.
3. A dependency receives that same secondary value and its low 30 bits shifted right by 10.
4. Further indirect invocations produce a new selecting result.
5. The same second secondary value is stored at that new result’s scaled-word destination.
6. The next invocation receives four times the new selecting result.

The address selector in step 2, both argument derivations in step 3, and the stored word in step 5 share one represented value. They are not independently unconstrained register observations.

Neither this identity nor the equal arguments to F-674e75a953e8 establishes that any dependency preserves actual registers, globals, or memory. Validity of the recovered dataflow remains a separate uncertainty.

### Middle packed-byte group

A new indirect result supplies a packed value.

The first selector is read at eight times the retained direct-call input plus one. Its selected destination receives the low byte of four times the packed value. The next invocation receives that full 32-bit scaled value.

After intervening calls, another selector is read at 64 times the packed value plus one. Its selected destination receives packed bits 20 through 27.

Subsequent invocations receive:

- A returned value shifted left by two.
- Another returned value shifted left by two together with the retained second selector.
- The retained direct-call input shifted left by 10 in both argument positions.

After another invocation, a selector is read at numerical address 1 and zero is written to its selected destination. Here the constant address follows from a separately initialized zero multiplier, not the earlier shift-and-multiply collapse.

### Third secondary value and explicit-zero group

The next dependency receives the **third secondary value** twice, followed by sixteen times that same value.

After intervening calls:

- A selector byte is read at sixteen times the same third secondary value plus one.
- Its destination receives the low byte of that value shifted right by 22.
- A dependency receives a newly returned value shifted left by two.
- Following more calls, another dependency receives the retained selector first and another returned value shifted left by two second.

The repeated call arguments, selector-address calculation, and shifted payload all derive from one represented unsigned 32-bit value. Unknown dependency effects do not turn these into independent observations.

A separately initialized zero is shifted left by 10 and passed in both argument positions to another invocation. Both explicit arguments are zero.

### Split 64-bit result

The next invocation supplies a represented 64-bit result:

- Its high 32 bits select a scaled-word destination receiving zero.
- The next invocation receives that high half.
- After further invocations, the high half is stored at a newly returned value’s scaled-word destination.
- The next invocation receives four times that new selecting value.

The low half is retained for later calculations.

After more calls, a word is stored at 268,435,456 times the low half plus 20. Its payload is 67,108,864 times the low half. Under wrapping, the address component depends only on the low four bits, while the payload depends only on the low six bits.

The following invocation receives that scaled payload. After further calls, the low half masked to 29 bits and shifted right by nine is stored at another scaled-word destination. The next invocation receives four times that destination’s selecting value.

### Five scaled-word groups

Each group begins with a newly returned selecting value. Four times that value forms a **payload**. The first store writes the payload at sixteen times the selecting value plus 20.

This destination differs from payload plus 20: the payload itself is only four times the selector.

The groups then differ as follows:

1. **Two-read group.** Two separate 32-bit reads occur at payload plus 20. One read, shifted right by 12, supplies the first argument of a dependency; the payload supplies the second; the other read, shifted left by two, supplies the third. Read evaluation order is unresolved. After intervening calls, the retained payload is stored at a new scaled-word destination.

2. **Split-low argument group.** A dependency receives the payload twice and the retained low half of the 64-bit result shifted left by 26. After intervening calls, the payload is stored at a new scaled-word destination.

3. **Single-payload argument group.** A dependency receives the payload once. After intervening calls, the payload is stored at another scaled-word destination.

4. **Retained-read group.** A word is read once at payload plus 20, before the dependency receiving the payload. That read result is retained across subsequent calls. Its logical right shift by 12 is then stored at a newly selected scaled-word destination. Both that selecting value and its four-times address component remain relevant to the next group.

5. **Cross-group argument group.** After the initial payload store, a dependency receives three arguments: the preceding group’s four-times address component masked to six bits and shifted left by 14; the new payload; and the preceding group’s selecting value shifted left by 30. After intervening calls, the new payload is stored at another scaled-word destination.

After each group’s later store, a dependency receives the associated four-times selecting value before calls progress toward the next group or final byte group.

The fourth group’s retained word must not be reloaded after intervening calls. Conversely, the first group’s two reads must not be consolidated.

### Final packed-byte group

A final indirect result supplies a packed value.

A selector is read at 268,435,456 times the selecting value retained from the fourth scaled-word group, plus one. Its selected destination receives the low byte of four times the final packed value.

The next invocation receives four times the packed value twice, followed by sixteen times the packed value. Its target construction uses a displacement 18 bytes smaller than the repeated construction used for most earlier indirect invocations. Target equivalence is not established.

A later indirect result is forwarded unchanged to another dependency. After additional calls, the earlier selector address is read again. The newly selected destination receives final packed bits 20 through 27.

Further invocations receive a returned value shifted left by two and, later, another shifted result together with the retained second selector. One final indirect invocation precedes the invalid-instruction boundary.

## Side effects

Established direct effects include byte and word reads, byte and word writes, numerous indirect invocations, and one direct invocation of F-674e75a953e8.

Dependencies can have additional effects. None is established as pure, nonblocking, allocation-free, or state-preserving.

Ownership is **documented_only**. No explicit allocation, release, ownership transfer, or lifetime validation is visible. Passing a quantity does not establish ownership acquisition; absence of a visible release does not establish a leak.

No rollback is visible. A fault or nonreturning dependency can leave earlier effects in place. Atomicity, synchronization, volatile semantics, and concurrency guarantees are unresolved. Computed destinations may alias selector storage, word-read locations, or dependency-visible state.

## Dependencies

| Opaque identifier | Established role | Unresolved contract |
|---|---|---|
| F-indirect-001 | First invocation; receives the explicit 32-bit input | Target identity, complete ABI, effects, return behavior |
| F-indirect-002 | Collective designation for the remaining ordered indirect invocation sites | Distinct target identities, implicit inputs, result conventions, effects, errors, ownership |
| F-674e75a953e8 | Direct invocation receiving the retained direct-call input twice | Return behavior, register and memory effects, ownership, errors |

F-indirect-002 denotes an unresolved set of sites, not one shared implementation. No-argument displays do not establish absence of implicit inputs. The terminal invalid-instruction boundary is not established as an ordinary dependency invocation.

## Acceptance examples

These are conditional static examples, not executed tests. They assume valid reached accesses, returning dependencies, and the stated arithmetic model.

1. **Byte truncation.** A packed value of 64 produces a first byte payload of zero, because four times 64 is 256. A corresponding full-width argument remains 256.

2. **Upper-field extraction.** A packed value of hexadecimal FFF00000 produces a bits-20-through-27 byte payload of 255, not a 10-bit store.

3. **Selector reread.** Selector results of 3 and then 9 produce byte writes at numerical addresses 20 and 26. Retaining the first selector for both writes would violate the represented reads.

4. **Collapsed address.** Any 32-bit address seed shifted left by 26 and multiplied by 1,024 yields zero under wrapping. The resulting selector address is 1.

5. **First secondary identity.** If the first secondary value is hexadecimal 00800000, the relevant invocation receives that value as its middle argument, and the later byte payload is 2. Intervening calls do not justify substituting an unrelated represented value.

6. **Second secondary identity.** If the second secondary value is 3, the retained direct-call input is stored at address 32; the subsequent arguments are 3 and zero; and the later dependency-selected word store contains 3.

7. **Third secondary identity.** If the third secondary value is hexadecimal 00400002, the repeated arguments both equal that value, the third argument is hexadecimal 04000020, the selector address is hexadecimal 04000021, and the shifted byte payload is 1.

8. **Different destination scales.** A group selector of 3 produces payload 12 and an initial word destination of address 68. A scaled-word destination selected by 3 is instead address 32.

9. **Split-result distinction.** High half 2 and low half 5 produce an initial zero store at address 28. The low-half-scaled store uses address hexadecimal 50000014 and payload hexadecimal 14000000.

10. **Partial execution.** If a dependency does not return after a store, later writes, rollback, and a synthesized status result are not required.

## Uncertainties

The following remain **documented_only**:

- Validity of the instruction interpretation throughout the represented region.
- Actual ABI, implicit inputs, and indirect target identities.
- Producers and correctness of recovered secondary-register dataflow.
- Whether numerical addresses designate ordinary memory, devices, or inaccessible regions.
- Endianness, alignment, fault behavior, and address-space rules.
- Relative evaluation order of separate reads used within one invocation.
- Dependency effects on memory, globals, registers, ownership, and control flow.
- Possible execution paths outside the represented sequence.
- Actual behavior at the terminal boundary.

Secondary-value producer uncertainty does not erase established represented identity within each value’s uses. Conversely, represented identity does not prove real dependency register preservation.

## Evidence

This contract derives solely from the supplied static representation, including its invalid-instruction warning.

Established evidence includes the explicit 32-bit input, incoming register dependencies, three shared secondary-value dataflows, ordered invocations, byte and word access widths, scaled addresses, bit operations, split 64-bit result, and absence of a visible normal return.

Zero-collapse and numerical examples are conditional arithmetic deductions. Application intent remains inferred or unknown.

No runtime observations or runtime tests have been performed.
