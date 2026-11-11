<!-- Trusted provenance: F-7ecd62ba5e7c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7ecd62ba5e7c

## Purpose

F-7ecd62ba5e7c performs an ordered sequence of indirect calls, computed-address memory accesses, and a global assignment. Its established direct effects include six 32-bit stores through computed addresses, two byte lookups followed by byte stores, a byte clear, and a halfword clear. The named-global assignment is separate from those six computed-address stores.

The sequence ends by attempting a call through address zero with three zero arguments. A normal return is not established.

No application-level purpose is established. The supplied material does not demonstrate allocation, serialization, object initialization, or a conventional buffer-processing interface. Interpretations involving word indices or tagged values remain inferred intent, not requirements.

All behavior below is conditional on preceding calls returning and preceding memory operations completing. The description distinguishes explicit dataflow from unresolved dependency behavior and execution-environment requirements.

## Interface

### Explicit inputs

| Input | Represented width | Established use |
|---|---:|---|
| First argument | 32 bits, uninterpreted | Passed unchanged to F-00000001. |
| Second argument | Signed 32 bits | Multiplied by four, then increased by 20 bytes to address the first word store. |
| Third argument | Unsigned 32 bits | Has its bottom two bits cleared, then is increased by 20 bytes to address a later word store. |

The third argument is not multiplied by four. Clearing its bottom two bits selects the same aligned-down numerical value for each group of four consecutive unsigned values. This does not establish a valid object or allocation.

### Implicit execution inputs and auxiliary state

The representation additionally depends on:

- An incoming unsigned 32-bit execution-state word, reused for two word stores and one byte-lookup address.
- An incoming link-like value, represented as signed 32 bits, used to derive the first call target by adding 3,996.
- A global slot whose captured value supplies the base for a later word read.
- A second global slot assigned near the end.
- An auxiliary byte pointer exposed by the call sequence and used for a final byte clear.
- Any register, memory, or global state consumed by the dependencies.

The auxiliary pointer is not an established explicit argument. Its complete production and calling-convention contract are unavailable.

### Outputs and completion

There is no explicit return value. Observable outputs are direct writes, global-state changes, dependency effects, and attempted calls.

The final address-zero call may return, fail, or transfer control elsewhere; the supplied material does not establish which. If it returns normally, no further explicit memory operation appears before the nominal return.

### Widths, arithmetic, and layout

Direct word accesses are 32 bits; byte accesses are 8 bits; the final halfword store is 16 bits. Address displacements are expressed in bytes.

Two dependency results are represented as 64-bit quantities. Their lower and upper halves are extracted as 32-bit values. This establishes numerical half selection, not a memory layout, byte order, or aggregate-return ABI.

Some address calculations multiply signed 32-bit values by four, and some value calculations left-shift signed 32-bit values by two. Portable signed-overflow and signed-left-shift behavior is not established for problematic operands. Ordinary arithmetic examples below avoid those cases.

Explicit unsigned 32-bit masks and logical shifts operate on 32-bit bit patterns. Unsigned left shift by two discards bits shifted beyond that width. The unsigned addition used for the third argument’s aligned-down value is likewise a 32-bit operation; interpreting the result as an effective address remains environment-dependent.

No structure definition, allocation boundary, alignment guarantee, pointer-width contract, or memory-map validity follows from the repeated 20-byte displacement.

## Behavior

Dependencies are assigned opaque IDs F-00000001 through F-00000056 in invocation order. These identify call sites, not proven distinct implementations. A call with “no explicit arguments” has none shown in the supplied representation; it may still consume implicit state.

### Initial stores and result forwarding

1. Before any dependency call, write the incoming execution-state word to the address formed by multiplying the second argument by four and adding 20.

2. Invoke F-00000001 with the first argument. Its target depends on the incoming link-like value.

3. Invoke F-00000002 with no explicit arguments, then pass its 32-bit result to F-00000003.

4. Invoke F-00000004 with no explicit arguments, then pass its 32-bit result to F-00000005.

5. Invoke F-00000006, F-00000007, and F-00000008, in that order, with no explicit arguments.

6. Invoke F-00000009 with no explicit arguments. Interpret its result as a signed 32-bit index. Write the incoming execution-state word to that index multiplied by four, plus 20. Then invoke F-00000010 with the index multiplied by four, without the displacement.

7. Invoke F-00000011, F-00000012, and F-00000013, in that order, with no explicit arguments.

8. Invoke F-00000014 with no explicit arguments. Left-shift its signed 32-bit result by two. Write that scaled result to the third argument’s aligned-down value plus 20. Invoke F-00000015 with two explicit arguments, both equal to the scaled result.

These are the first three computed-address word stores. No direct read of their previous contents is shown.

### Zero store and retained scaled result

9. Invoke F-00000016 with no explicit arguments, then forward its 32-bit result to F-00000017.

10. Invoke F-00000018 with no explicit arguments, then forward its 32-bit result to F-00000019.

11. Invoke F-00000020, F-00000021, and F-00000022, in order, with no explicit arguments.

12. Invoke F-00000023 with no explicit arguments. Interpret its result as a signed 32-bit index. Write a zero word to that index multiplied by four, plus 20. Invoke F-00000024 with the index multiplied by four.

13. Invoke F-00000025, F-00000026, and F-00000027, in order, with no explicit arguments.

14. Invoke F-00000028 with no explicit arguments and obtain a 64-bit result. Interpret its lower half as signed 32 bits and left-shift it by two. Interpret its upper half as a signed 32-bit destination index. Store the scaled lower half at the upper-half index multiplied by four, plus 20.

15. Read and retain the value of the global base slot. This occurs after the preceding store and before F-00000029.

16. Invoke F-00000029 with four arguments, in this order: the scaled lower half, the unscaled upper half, the scaled lower half again, and the scaled lower half again.

17. Invoke F-00000030, F-00000031, and F-00000032, in order, with no explicit arguments.

18. Invoke F-00000033 with no explicit arguments. Interpret its result as a signed 32-bit index. Store the retained scaled lower half from F-00000028 at that index multiplied by four, plus 20.

19. Read a 32-bit word at the previously captured global base plus 20. Retain this word for both later byte stores.

20. Invoke F-00000034 with the index from F-00000033 multiplied by four.

Steps 12, 14, and 18 supply the remaining three computed-address word stores, bringing that total to six.

The base-slot read and the word read through that base are distinct events. Changes to the base slot after step 15 do not, in the represented dataflow, redirect step 19. Intervening calls or direct writes can nevertheless change the contents at the captured address before the word is read.

### Byte lookups and extraction

21. Invoke F-00000035, F-00000036, and F-00000037, in order, with no explicit arguments.

22. Invoke F-00000038 with no explicit arguments and obtain a 64-bit result. Retain its lower half as an unsigned 32-bit value and its upper half as a 32-bit value.

23. Logically shift the incoming execution-state word right by 23 bits, then add one. Read an unsigned byte at that numerical address. Add 17 to the byte’s value and use the result as an absolute destination address. Write the bottom eight bits of the word captured at step 19 there.

24. Invoke F-00000039 with four arguments: the upper half from F-00000038 three times, followed by the lower half left-shifted by two as an unsigned 32-bit value.

25. Invoke F-00000040 with no explicit arguments, then pass its 32-bit result to F-00000041.

26. Invoke F-00000042, F-00000043, and F-00000044, in order, with no explicit arguments.

27. Invoke F-00000045 with no explicit arguments and retain its signed 32-bit result.

28. Clear the top two bits of the retained lower half from F-00000038, logically shift right by 21, and add one. Read an unsigned byte at that numerical address. Add 17 to the byte’s value and use the result as an absolute destination address. Store bits 22 through 29, inclusive, of the captured word there.

29. Invoke F-00000046 with the result from F-00000045 left-shifted by two.

Both byte-lookup addresses range from 1 through 512 inclusive. Each looked-up byte selects a destination from 17 through 272 inclusive. These are numerical ranges, not proof of accessible memory.

The lookups occur separately at their specified positions. Even if their calculated lookup addresses coincide, the second byte read is not replaced by reuse of the first result. Intervening dependencies or writes may change that location.

### Final calls and clears

30. Invoke F-00000047, F-00000048, and F-00000049, in order, with no explicit arguments.

31. Invoke F-00000050 with no explicit arguments. Left-shift its signed 32-bit result by two and invoke F-00000051 with two arguments, each equal to that scaled value.

32. Invoke F-00000052 with no explicit arguments.

33. Invoke F-00000053 with no explicit arguments, then pass its 32-bit result to F-00000054.

34. Invoke F-00000055 with no explicit arguments. Assign its 32-bit result to the second global slot.

35. Write a zero byte through the auxiliary pointer exposed by the call sequence. Its complete producer contract is unresolved.

36. Clear the top two bits of the retained lower half from F-00000038 and logically shift right by seven. Use that numerical result directly as an address and write a zero halfword there.

37. Invoke F-00000056 through address zero with three explicit zero arguments.

The final halfword address ranges from zero through 8,388,607 inclusive. It need not be two-byte aligned. There is no special case avoiding address zero.

## Side effects

### Direct memory effects

Conditional on reaching and completing every operation, the direct sequence contains:

| Effect | Count |
|---|---:|
| 32-bit stores through computed addresses | **6** |
| Read of the global base slot | 1 |
| 32-bit read through the captured base | 1 |
| Byte reads used to select byte destinations | 2 |
| Byte stores containing extracted data | 2 |
| Assignment to the second global slot | 1 |
| Byte clear through the auxiliary pointer | 1 |
| Computed-address 16-bit clear | 1 |

The six computed-address word stores occur at behavior steps 1, 6, 8, 12, 14, and 18. The global assignment at step 34 is separately counted, not a seventh computed-address store.

Dependency effects are additional and unrestricted by the supplied material.

### Ordering and aliasing

The listed order is material. No transaction, rollback, validation barrier, or deferred commit is shown. A fault or non-returning call may leave earlier effects in place.

No disjointness checks are shown. Computed word destinations, global storage, lookup bytes, byte destinations, and final clear destinations may overlap where their addresses coincide. Distinct global symbols also do not prove distinct storage.

The captured data word is read once and reused. A dependency changing its original memory after step 19 does not cause a direct reread in the represented dataflow.

If the two byte destinations coincide, the second direct byte store overwrites the first at that point in the sequence. This does not establish the final function-wide value: later calls or clears may write the same location.

### Ownership and failure

No allocation, release, reference-count operation, ownership transfer, or lifetime extension is established by the direct operations. No dependency is known to preserve globals, retain arguments, own passed values, or release memory.

There is no explicit status result, error branch, or recovery path. Accessible memory, acceptable alignment, valid call targets, and suitable arithmetic are prerequisites for continuing through the corresponding operations.

## Dependencies

| Opaque IDs | Established visible role |
|---|---|
| F-00000001 | Receives the first argument; target is derived from incoming execution state. |
| F-00000002–F-00000008 | Two result-forwarding pairs, then three calls without explicit arguments. |
| F-00000009–F-00000015 | Produce indices or scaled data associated with the initial stores and receive the specified scaled arguments. |
| F-00000016–F-00000027 | Two forwarding pairs, intervening calls, and the zero-store index producer and consumer. |
| F-00000028–F-00000034 | Produce the first split result and a later index; receive repeated values and scaled indices. |
| F-00000035–F-00000046 | Surround the byte stores, produce the second split result, and consume its halves or other returned values. |
| F-00000047–F-00000054 | Final intervening calls, a repeated scaled-argument call, and a result-forwarding pair. |
| F-00000055 | Produces the word assigned to the second global slot. |
| F-00000056 | Address-zero target, invoked with three zero arguments. |

Except for the final zero target, targets are represented by execution-derived or code-associated address calculations. Resolved identities, complete signatures, implicit inputs, auxiliary outputs, and semantic contracts are not independently established.

## Acceptance examples

These are conditional arithmetic and ordering examples, not executed tests. They assume relevant accesses are valid, preceding calls return, and signed arithmetic is defined for the selected values.

1. **Initial store:** A second argument of 3 selects address 32. The incoming execution-state word is stored there before F-00000001.

2. **Aligned-down third argument:** Third-argument values 256, 257, 258, and 259 all select address 276 for the scaled result from F-00000014.

3. **Split result:** If F-00000028 returns lower half 5 and upper half 8, step 14 writes 20 at address 52. F-00000029 receives 20, 8, 20, and 20.

4. **Captured base:** If step 15 captures base 1,024, step 19 reads address 1,044. Changing only the global base slot between those steps does not redirect that read. Changing memory at address 1,044 can change its result.

5. **Extracted bytes:** A captured word of hexadecimal A5C0007E supplies hexadecimal 7E for the first byte store and hexadecimal 97 for the second.

6. **Coincident byte destinations:** If both lookup reads produce 40, both byte stores target address 57. The second store replaces the first there when reached; later effects may still change it.

7. **Separate lookup observations:** If both lookup addresses are equal but an intervening dependency changes that byte from 40 to 41, the destinations are respectively 57 and 58.

8. **Final halfword clear:** A retained lower half of 256 selects address 2. A lower half of zero selects address zero.

9. **Partial execution:** If F-00000015 is entered and does not return, the first three computed-address word stores have completed. The remaining three, both extracted-byte stores, global assignment, and final clears have not been reached.

10. **Store accounting:** If execution reaches the terminal call after completing every direct access, it has performed six computed-address 32-bit stores and one separately classified global assignment.

## Uncertainties

The following remain **documented_only**:

- Application-level purpose and validity of the low numerical addresses.
- Accuracy and completeness of the represented interface and dependency signatures.
- Calling-convention status of the incoming execution-state and link-like values.
- Effective-address width, pointer conversion, and execution-level overflow behavior.
- Behavior of problematic signed multiplication, addition, and left shifts.
- Endianness, alignment requirements, and fault handling.
- Whether different call sites resolve to different implementations.
- Dependency return behavior and all unshown memory, global, ownership, or control-flow effects.
- The auxiliary byte pointer’s precise producer, validity, and lifetime.
- Platform-specific meaning or overlap of the global slots.
- Whether address-zero execution is supported and whether nominal return is reachable.

An inferred interpretation is that some quantities are word indices or tagged values. That inference does not establish an object layout or override the explicit access calculations.

## Evidence

Established behavior comes solely from the supplied static representation: access widths, masks, shifts, displacements, result-half extraction, argument forwarding, call order, and explicit reads and writes.

Counting the explicit computed-address word writes yields six. The later global assignment is a distinct operation and is not included in that count.

The supplied representation warns that some global symbols overlap smaller symbols at the same address; symbolic distinctions therefore do not establish storage separation.

No runtime tests have been performed. No dependency implementations, executable instruction stream, platform memory map, or external calling-convention contract were available for independent validation. The acceptance examples express conditional consequences of the static representation, not observed executions.
