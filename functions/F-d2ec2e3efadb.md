<!-- Trusted provenance: F-d2ec2e3efadb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d2ec2e3efadb

## Purpose

The routine combines indirect calls, width-sensitive arithmetic, direct memory accesses, conditional record updates, and potentially repeated processing. Represented outcomes include a constructed 64-bit return, return through an unresolved post-interrupt transfer, repetition, and termination at invalid instruction data.

No application-level purpose is established. The evidence does not identify the routine as an allocator, parser, serializer, or device driver.

This contract describes established properties of the supplied static representation. Invalid instruction data and unresolved control flow prevent an unconditional claim that every represented path is executable.

## Interface

### Explicit inputs

| Argument | Width | Established use |
|---|---:|---|
| First | 32 bits | Shifted to form an early byte-read address contribution. |
| Second | 32 bits | No explicit use is represented. |
| Third | 32 bits | No explicit use is represented. |
| Fourth | 32 bits | Interpreted as signed for shifts; selects an early store address and later address bases. |

The apparently unused arguments are not proven irrelevant to the actual machine interface.

### Implicit inputs

Behavior additionally depends on:

- Incoming machine-register values, including a return-address-related value used in indirect-target selection.
- Incoming coprocessor-related values.
- Incoming stack words and accessible stack storage.
- Global values and current memory contents.
- Primary, secondary, and paired results of indirect calls.

The represented interface is insufficient to reconstruct these inputs from the four explicit arguments.

### Output and execution prerequisites

The apparent return width is 64 bits. For paired values, “low word” and “high word” mean the lower and upper 32 bits, not their order in memory.

The shared return produces:

- Low word: twice a selected 32-bit value, with 32-bit truncation.
- High word: a freshly read 32-bit global value minus 84, with 32-bit truncation.

Another path returns the 64-bit result of an unresolved transfer following software interrupt 74. Other paths may repeat, encounter invalid instruction data, or fail through an invalid access or transfer.

Normal execution requires valid memory accesses, indirect targets, and machine-state assumptions for the selected path. No general null, range, alignment, permission, or lifetime validation is represented. These prerequisites are not checks performed by the routine.

## Behavior

### Width and arithmetic rules

Word calculations and address-like values are represented as 32 bits unless otherwise stated. Byte, halfword, and doubleword accesses are 8, 16, and 64 bits respectively.

Arithmetic described as wrapped discards bits above bit 31. Narrow stores retain the low bits of their payload. Signed right shifts are represented as arithmetic shifts; the explicitly unsigned right shift is logical.

Signed overflow, unsigned carry, and ordinary comparisons are distinct predicates. They must not be substituted for one another.

One shift count is masked to eight bits. Semantics for resulting counts of 32 through 255 are not established. Likewise, the first argument’s left shift followed by arithmetic right shift must not be simplified to the original argument.

Memory byte order and architectural handling of overflow, unaligned accesses, and faults remain unresolved.

### Initial calls, store, and saved predicates

The routine first forms an address contribution by shifting the first argument left one bit, shifting that result arithmetically right one bit, and subtracting 82.

It calls an indirect dependency with two visible arguments, in order:

1. 45.
2. A global 32-bit value.

It then reads one byte at the derived contribution plus an implicit incoming address contribution plus 9. The second indirect call receives, in order:

1. An implicit incoming value minus 79.
2. The loaded byte.

After that call, a 32-bit value identifying the address of **F-0d1559cd79ec** is stored at the original fourth argument, arithmetically shifted right by 13, plus 74. This does not establish invocation of that function.

The fourth argument is then reduced to its sign word: zero for nonnegative inputs, or all one bits for negative inputs. This selects:

- An offset of zero or −131,072.
- A byte-read address of zero or −262,144.

An intermediate word is formed from the second call’s primary result minus 56, arithmetically shifted right by a count equal to the low eight bits of an implicit incoming value plus 21, then doubled.

Before further calls, the routine saves three predicates:

- Whether this word is unsigned-greater-than `0xFFFFFFCD`.
- Whether adding 50 causes signed overflow.
- Whether the wrapped sum with 50 is negative.

It reads a byte from the sign-selected byte-read address. This saved byte participates in the direct processing-body entry described below, even though no argument explicitly passes it to the next call.

An indirect call returns a pair. The routine writes one byte at the pair’s low word plus the sign-selected offset. Its payload is the offset’s low byte, which is zero for both signs.

Only after that call and store does it read a global 32-bit base. This base is a saved observation, not an entry-time snapshot.

The extended preparation branch is selected when the saved sign of the sum differs from the saved signed-overflow predicate. A separate dispatch working word initially equals 13.

### Extended preparation

On the extended branch, the sign word selects a base of zero or `0xF0000000`. The routine reads an unsigned byte at that base plus 21, then calls an indirect dependency.

It subsequently performs these operations in order:

1. Read a signed halfword at the selected base plus the saved byte.
2. Sign-extend that halfword and use it as an offset from the saved global base to read another halfword.
3. Store the saved byte, widened to 32 bits, at the saved global base plus 84.
4. Call an indirect dependency with the second halfword.
5. Read a 32-bit word at the saved byte plus 52.
6. Perform a coprocessor operation using the signed halfword and incoming coprocessor-related state.

A following call supplies a halfword result and a secondary address result. The halfword is stored at that address plus 60.

Two further calls precede an unsigned-halfword read at a secondary-result address plus 36. Two more calls follow; the latter supplies a value passed to another call together with an incoming stack word.

A subsequent paired result supplies both address words and the full payload for this ordered store sequence:

1. Store the saved unsigned halfword at the pair’s high word plus 36.
2. Store the entire 64-bit pair at the pair’s low word.
3. Store the saved word read at the byte-derived address plus 52 at the pair’s low word plus 8.
4. Store the saved unsigned halfword, zero-extended to 32 bits, at the pair’s low word plus 12.
5. Store the saved global base at the sign-selected base plus 100.

These addresses need not be disjoint. In particular, the first store may overlap subsequent stores.

A call then receives the pair’s low word, its high word, and another incoming stack word. Another call supplies a secondary address from which the routine reads a 32-bit record address.

The record-related sequence is:

1. Call an indirect dependency.
2. Store a zero byte at record offset 11.
3. Call another dependency.
4. Read and save the byte at record offset 15.
5. Call another dependency.
6. Store a secondary-result byte at record offset 19.
7. Call another dependency, obtaining a base.
8. Read a 32-bit word at that base plus the saved record byte, without scaling the byte.
9. Perform two further calls, saving the latter primary result.

These accesses establish offsets and widths, not a record type.

### Signed-subtraction-overflow alternatives

The routine tests signed overflow when subtracting the saved record address from the saved word loaded through the byte-indexed base.

If overflow is reported, it doubles the latest primary result with truncation and forms a byte-derived quantity equal to the saved record byte plus 18. That quantity ranges from 18 through 273; it is not immediately narrowed to a byte.

A call receives the latest primary result and that result arithmetically shifted right by 25. At the returned address plus 100, the routine stores the wrapped subtraction result. It then reads a word at the returned address plus `0x07000000`.

The carry-selected branch is taken if either:

- Adding the byte-derived quantity to the loaded word produces unsigned carry; or
- Adding a Boolean to that wrapped sum produces unsigned carry.

The Boolean equals one exactly when the returned address is unsigned-greater-than `0xF8FFFFFF`.

On this branch, indirect calls alternate with these accesses, in order:

1. Read a byte at the returned address plus `0x07000008` plus the byte-derived quantity.
2. Store the doubled primary result as a word at the returned address plus `0x07000054`.
3. Store the loaded byte, widened to a word, at the returned address plus `0x07000044`.
4. Store the low byte of the returned address plus `0x07000008` at the loaded byte plus the byte-derived quantity.

Further calls include one receiving a preceding primary result minus 166. Software interrupt 74 follows. An unresolved indirect transfer then receives four visible arguments: a global address-like value, a byte read at a global-derived address plus 9, another global value, and a stack-area address. If this transfer returns as represented, its 64-bit result is returned directly.

The source does not establish evaluation order or snapshot consistency among the separate global expressions used for these arguments.

On the non-carry alternative, the routine stores the low byte of the word-plus-quantity-plus-Boolean sum at a global-derived address plus 17. Representation ends at invalid instruction data; no normal result is established.

If signed subtraction overflow was absent, the routine instead:

- Replaces the saved dispatch predicate with an unsigned comparison of the latest primary result, arithmetically shifted right by 9, against `0xFFFFFFBE`.
- Forms a pair with that primary result as low word and its shifted value plus 65 as high word.
- Replaces the dispatch working word with the saved record address.

### Common dispatch

The initial path and non-overflow extended path converge at a call receiving the current pair’s low and high words. This call returns the pair used for dispatch.

If the saved dispatch predicate is false, execution enters the common tail.

Otherwise, the routine stores the low halfword of the dispatch working word at the returned low word plus 42. It compares:

- Whether the dispatch working word is negative as signed 32-bit data.
- Whether subtracting 47 from the returned high word causes signed overflow.

Equal Boolean results select the common tail. Unequal results select the processing body.

For this direct body entry only:

- The address-like state is the returned high word.
- The first call’s working argument is the returned low word logically shifted right by 25.
- The pending offset-62 store payload is an implicit incoming word minus the saved sign-selected byte, or minus the saved record address on the extended path.

### Processing body and entry-dependent payload

Every body entry starts with a call receiving the current working word, the current address-like word, and the address of a local stack area. Another call returns a pair.

The routine forms the current address-like word plus 10 and stores the entry’s pending 32-bit payload at the original address-like word plus 62. This store may be unaligned.

The payload is not invariant across iterations:

- Direct entry from conditional dispatch uses the difference described above.
- Every entry from the common tail uses the wrapped four-times-square value computed by that immediately preceding tail.
- This also applies when execution initially bypasses the body, enters the tail, and then reaches the body for the first time.

If the wrapped product of the incremented address-like word and 65,536 is negative, execution takes the shared return. The selected return input is the most recent paired call’s low word.

Otherwise, the routine:

1. Stores that pair’s low halfword at its high word plus 60.
2. Performs two indirect calls.
3. Stores halfword value 38 at the latest returned address plus 2.
4. Forms a temporary word by multiplying the incremented address-like word by `0x10000000`, truncating, then arithmetically shifting right by 9.
5. Forms the wrapped sum of that temporary word and the incremented address-like word.
6. Reads and saves a global 32-bit XOR operand.
7. Calls a dependency with twice the latest returned address and the incremented address-like word multiplied by 65,536 minus 81.
8. XORs the saved sum with the saved global operand to obtain a new address-like word.
9. Adds 47 to the temporary word.
10. Saves the new address-like word in an incoming stack slot.
11. Calls a dependency with fixed address-like constant `0x085734E8`, the byte at the new address plus 1, the adjusted temporary word, and a secondary result of the preceding call.
12. Stores the adjusted temporary word at the new address plus 36.
13. Performs another call.

The global XOR operand is read before the doubled-address call, but the XOR occurs after it. A callee’s modification of that global does not replace the saved operand. The stack-slot update precedes the following byte-read/call expression and the subsequent store and call.

The pair entering the common tail then has the new address as low word and the latest secondary result plus 48 as high word.

### Common tail and repetition

For the current pair, the tail computes:

- A pending payload equal to four times the square of the high word, wrapped to 32 bits.
- A working word equal to twice the low word, wrapped to 32 bits.

It then calls a dependency with the low word plus an implicit incoming value, calls another with a global word, and performs a third call.

After those calls it tests a termination global against zero. The access width of this global is not established by the supplied representation.

If the observed value is zero, a secondary result from the third call is arithmetically shifted right by 9 and used as the shared return input.

If nonzero, the routine stores the pending square-derived payload as a word at twice the earlier high word plus 52. It replaces the address-like state with that third call’s secondary result and enters the processing body. The working word remains twice the tail pair’s low word, and the body’s offset-62 payload is this tail’s square-derived payload.

No iteration bound is represented.

## Side effects

The routine accesses byte, halfword, word, and doubleword storage through low addresses, large offsets, globals, stack areas, and call results. Accesses may alias or be unaligned.

Indirect calls occur between accesses. No dependency is assumed to preserve globals, reachable memory, exposed stack storage, machine state, or resource ownership. Saved observations are distinct from later memory reads.

Stack visibility is significant: the body exposes a local stack-area address, updates an incoming stack slot before another call, and the post-interrupt path exposes a different stack-area address.

No allocation, release, reference-count change, or ownership transfer is positively identified. An address-like paired result is not evidence of allocation. No cleanup or rollback guarantee is established.

The coprocessor operation and software interrupt may have effects beyond ordinary memory; their meanings are unresolved.

## Dependencies

- **F-0d1559cd79ec**: its address is stored as a 32-bit value. A call to it is not established.
- **F-882dff43a182**: identifies the mapped shared return location. The represented transfer there is internal control flow, not an established ordinary call.
- **F-000000000001**: documentation-only grouping ID for unresolved indirect dependencies, including the post-interrupt transfer. It does not assert that distinct call sites share a target.

Unresolved dependencies have unknown calling conventions, preservation rules, failure behavior, ownership effects, and secondary-result conventions. Indirect-target selection depends on incoming or path-dependent address-like state.

## Acceptance examples

These are conditional static obligations, not executed tests.

1. **Second-call argument order.** Given a loaded byte of 7 and an implicit first-argument source of 100, the second call receives 21 first and 7 second.

2. **Early function-address store.** Given valid preceding operations, a fourth argument of zero selects address 74; −1 selects address 73. The stored value is 32 bits and identifies F-0d1559cd79ec.

3. **Sign-selected byte write.** Both signs select a zero payload. The negative-case offset is −131,072 relative to the nonnegative-case offset. Complete destinations differ by that amount only if the paired call returns the same low word in both executions.

4. **Unscaled indexing.** A saved record byte of 3 and returned base B select a word read at B + 3, not B + 12.

5. **Promoted byte arithmetic.** A saved record byte of 255 produces quantity 273 after adding 18, not 17.

6. **Signed-overflow selection.** A saved loaded word of `0x80000000` minus record address 1 selects the signed-overflow branch. An unsigned-borrow test is not equivalent.

7. **Tail-supplied body payload.** If a tail pair has high word 3 and the termination observation is nonzero, its square-derived payload is 36. That tail stores 36 at address 58; the next body stores 36 at its current address-like word plus 62, assuming intervening execution reaches that store.

8. **First body entry through the tail.** If dispatch bypasses the body and a tail with high word 4 observes nonzero termination, the first body entry uses payload 64, not the direct-dispatch difference.

9. **Saved XOR operand and exposed stack.** If the XOR operand is read as 1 and the following callee changes its global storage to 2, the address calculation still uses 1. The resulting address is saved in the incoming stack slot before the next call.

10. **Zero termination observation.** Given a zero termination value and secondary result `0x00000400`, the shared return low word is 4. The high word separately depends on the return-time global minus 84. No byte-width assumption is required.

11. **Invalid-data endpoint.** Given signed subtraction overflow and false combined carry, the global-derived byte store occurs before the represented invalid-instruction endpoint. No successful return is specified.

## Uncertainties

The following remain **documented_only**:

- Executability and reachability of represented paths.
- Complete ABI, implicit inputs, and indirect-target identities.
- Shift semantics for masked counts of 32 or greater.
- Termination-global access width.
- Endianness, alignment handling, address translation, and fault delivery.
- Validity and meanings of global, low-address, and large-offset storage.
- Coprocessor and software-interrupt semantics.
- Dependency ownership, lifetime, preservation, and error behavior.
- Effects of aliasing and callee changes to exposed memory.
- Exact evaluation order of expressions where the representation supplies no sequencing.

**Inferred intent:** offset-based accesses may concern structured state, and repetition may implement state processing. Neither interpretation is required by this contract.

## Evidence

The sole evidence is the supplied static representation and opaque dependency mapping. It establishes the represented widths, arithmetic relationships, branch predicates, access sequencing, stack update, and termination forms described above.

Its invalid-instruction and unresolved-control-flow warnings limit claims about actual execution. The termination comparison supplies no explicit load width.

No implementation access, runtime execution, or runtime tests have been performed. Acceptance examples express conditional consequences of static evidence, not observed outcomes.
