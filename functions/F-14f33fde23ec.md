<!-- Trusted provenance: F-14f33fde23ec; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-14f33fde23ec

## Purpose

F-14f33fde23ec accepts two opaque 32-bit inputs and performs an extensive, ordered sequence of external calls and memory accesses. Its directly represented effects include selector-directed byte writes, indexed word writes, accesses at fixed offsets, and publication to two global state slots.

No application-level purpose is established. The evidence does not justify identifying the function as an initializer, allocator, decoder, supervisor service, or cleanup routine.

The supplied representation contains no explicit conditional branches or loops. Its last call targets address zero with three zero arguments. A nominal void return follows only if that call returns.

This contract distinguishes:

- **Established static evidence:** operations, widths, dependencies, and ordering represented in the supplied material.
- **Inferred intent:** possible interpretations without contractual force.
- **documented_only:** unresolved behavior requiring information not supplied.

All descriptions of later behavior are conditional on preceding calls returning and preceding accesses completing.

## Interface

### Explicit inputs and output

The visible interface accepts two 32-bit values. Their signedness and application meaning are unspecified.

The entry dependency, F-unknown-001, receives four argument positions: first input, second input, second input, first input.

No return value is declared. There is no established success flag, error code, returned length, or returned pointer. Successful arrival at the nominal return is not guaranteed.

### Implicit inputs

Execution also depends on:

- An incoming link-like execution value used to derive the entry call target.
- An incoming 32-bit value used in early byte stores.
- An incoming address-like value used for early selector reads.
- Primary external-call results, including quantities represented as 64 bits.
- Additional result channels represented as 32-bit words or a pointer.
- Absolute-address state, notably the byte at address 1, the word global at address 0x30, and a 24-bit quantity beginning at address 9.
- Two global locations called **state slot A** and **state slot B** in this contract.
- A stack-resident address exposed to an external dependency.

These are execution dependencies, not additional declared parameters.

The call involving address 0x30 contains four direct argument expressions referencing that 32-bit global. Unlike nearby byte reads explicitly captured and reused, the representation does not establish one sampled word duplicated four times. Argument evaluation order and physical load count are **documented_only**.

### Widths, layouts, and arithmetic

The represented memory accesses have widths of 8, 16, and 32 bits. A byte store retains the low eight bits of its source. A 64-bit result is separated into low and high 32-bit portions where specified.

The following descriptive layouts recur:

| Description | Address or access relationship |
|---|---|
| Selector update using a base | Read the unsigned byte at base + 1; write a byte at the captured selector’s numeric value + 0x11 |
| Indexed word location | 4 × index + 0x14 |
| Expanded word location | 16 × index + 0x14 |
| Wider-stride word location | 256 × index + 0x14 |

Selector destinations range from 0x11 through 0x110. They are not generally base + 0x11. A selector clear still performs the selector read before writing zero.

Unsigned right shifts are logical. Unsigned 32-bit calculations retain 32-bit results; no unbounded arithmetic is implied. The representation also uses signed-looking multiplication and left shifts. Exact executable overflow, pointer conversion, and signed-shift behavior remain **documented_only**. Bit-vector deductions below are qualified where they depend on interpreting such operations as 32-bit machine arithmetic.

### Preconditions and errors

No validation or explicit error-handling branch is represented. Progress requires accessible memory of the requested width, usable implicit result channels, valid call targets, any required alignment, and returning dependencies.

These requirements are not checked by this function. Faults, exceptions, nonreturning calls, and dependency-defined error signaling are unresolved. No rollback is represented.

## Behavior

### Observation and retention rules

External calls separate many otherwise similar accesses. Their implementations are unavailable; they cannot be assumed to preserve globals, pointed-to memory, or exposed stack storage.

Accordingly:

- Repeated memory reads are distinct observations unless the representation explicitly retains an earlier value.
- Repeated selector updates reread their selector.
- A retained scalar used as an address remains that captured scalar; it does not imply a fresh global read.
- A later overwrite does not make an earlier store unobservable.
- An earlier global clear does not establish the value of a later global read across unknown calls.

Order within an expression, including the four direct address-0x30 arguments, is not established merely by the ordering of surrounding statements.

### Entry and early byte updates

After F-unknown-001, another dependency is called. The function then performs a selector update using the incoming address-like base, storing the low byte of the incoming 32-bit value.

After intervening calls, a second selector update uses the same base expression. Its stored value is the low byte of the incoming word masked with 0x3fffffff and shifted right by 20. The selector is read again; the destination may differ.

Early primary results are scaled by four and passed to dependencies, sometimes in two equal argument positions and sometimes followed by zero.

A byte at offset 0x0e from an additional-result-channel address is explicitly captured and supplied in four argument positions. A following 64-bit result provides its high half twice, zero, and its low half shifted right by 19 to another call. A selector clear occurs between obtaining that result and the call consuming its portions.

Further selector clears use additional result words shifted right by 9. Another repeated clear uses a retained base formed by shifting an unsigned primary result right by 18 and multiplying by 0x400. Calls intervene between the repeated selector reads.

Later, all four argument expressions of a call reference the word global at absolute address 0x30. No common sampled value or equality of observations under changing memory is guaranteed by the representation. Further calls and selector clears follow.

### Initial word linking and extraction

An additional-result-channel word is stored at its own indexed word location. A dependency then receives that retained word and a word read from a separately derived location, logically shifted right by 19.

Subsequent writes propagate the retained word to another indexed location. A newly returned index supplies both an expanded word destination and the value four times that index. Later indexed writes propagate the retained scaled value.

A primary unsigned result then supplies:

- A selector byte store containing the low byte of four times the result.
- A later selector byte store containing its masked extraction shifted right by 20.
- Two retained extractions formed by masking with 0x3fffffff and shifting right by 17 and by 7.

A later dependency receives seven argument positions: four copies of a primary result, the two extractions, and a code-associated value. No semantic role is established for the final argument.

### Mixed-width reads and call results

Following another 64-bit result, selector clears use the retained masked extraction shifted right by 7. Calls with differing target derivations intervene.

The low 32-bit portion of that 64-bit result supplies its top four bits to a call, a selector base obtained by shifting right by 9, and a byte-read address formed by multiplying by two. The byte is explicitly widened without sign extension and passed four times.

A later call consumes an additional result channel shifted right by 18. Subsequent processing retains another 64-bit result’s low word and its logical extraction shifted right by 19.

A word read uses the retained low word plus that extraction shifted left by 31. Under 32-bit bit-vector interpretation, this added offset is either zero or 0x80000000; it is not an unrestricted array index.

Further operations include:

- Another byte read at offset 0x0e.
- Selector clears using direct and shifted result-derived bases.
- A selector clear using a retained low result multiplied by 0x1000.
- A call receiving the same stack-resident address in all four argument positions.
- A captured unsigned byte read at a retained low result plus an additional-result-channel offset.
- A halfword read at the sum of that byte’s numeric value and a later result-channel word shifted right by 19.

The halfword is supplied in four argument positions. Its byte order and argument-extension convention are not established.

### Fixed offsets and expanded linking

A subsequent 64-bit result provides a retained low address-like word and a high index-like word. A retained extraction is stored at the high word’s indexed location and, after calls, at another indexed location.

At offset 0x0d from the retained low address, the function first stores the low byte of twice that address value. After intervening calls and a word read, it writes byte value 4 to that same retained address plus 0x0d.

Two word-read formulas occur in this sequence:

- Four times a returned unsigned word, plus 0x44.
- That word masked with 0x3fffffff, shifted right by 20, plus 0x44.

They are distinct address calculations.

A wider-stride write uses 256 times a returned index plus 0x14 and stores four times a retained word. Another indexed write stores an extraction masked with 0x03ffffff and shifted right by 13.

Several chains alternate expanded-location writes of four-times-index values with indexed writes propagating those retained values. Calls also consume high-bit extractions and left-shifted quantities.

Later, a zero word is stored at offset 0x24 from a scale-16-derived base, immediately followed by a word read at offset 0x74 from that base. The loaded address is retained across calls. A later unsigned result shifted right by 20 is written at its offset 0x24; a word at its offset 0x74 is then read and passed onward.

These accesses establish individual offsets, not a complete object layout.

### Byte extraction and further linking

The unsigned result already shifted right by 20 is subsequently shifted right by 21 for call arguments. This second extraction is necessarily zero: the first shift leaves at most 12 significant bits.

Three separate byte reads occur at offset 0x15 from independently supplied additional-result-channel addresses. Their values are passed onward. The third is explicitly widened without sign extension and stored as a 32-bit word at an indexed location.

A different result-channel word is masked with 0x1fffffff and shifted right by 18. The resulting value has at most 11 significant bits. It later supplies a left shift by 13 to a call and an indexed word store.

Further chains propagate retained indices, scaled indices, result words, and masked extractions. Selector updates remain interleaved. Some use a statically zero base and therefore read address 1, but the selector byte itself is still read at the represented point.

### State slot A clear and intervening observations

State slot A receives an explicit 32-bit zero store.

Subsequent operations include calls, indexed zero stores, expanded linking stores, and word reads supplying shifted call arguments.

A later selector update reads absolute address 1 and stores the low byte of a scaled unsigned result at the selected destination. Another update reads address 1 through a base expression known to be zero and stores a masked extraction.

F-unknown-002 receives two equal words. After it and another dependency, the function again reads the byte at address 1 to select a zero-byte destination.

The function subsequently captures a byte at offset 1 from a retained primary result. Across intervening calls, that captured byte becomes the address base for another byte read at captured value + 1. It is not a repeated read from the original primary-result address.

Several associated extractions are necessarily zero because their retained precursors are explicitly zero. This does not eliminate the represented calls, selector reads, or stores.

### Distinct offset-9 bases

A late byte-sized call result is shifted left by two and stored through a selector. Its signed promotion and left-shift semantics are **documented_only**.

The later offset-9 writes use two independently derived bases:

- The **first base** is a previously obtained primary result already scaled by four. The zero-byte destination is that scaled value multiplied by 0x2000, plus 9.
- The **second base** is a different external call’s primary result. It is captured before the zero-byte write and retained across subsequent calls. The later byte write stores this second result’s low byte at the second result’s own address plus 9.

Equality of these bases is not established. The later write is not necessarily an overwrite of the zero-byte destination.

The following word read at offset 0x34 uses the **second base**. That same second-base value is propagated through subsequent indexed word stores.

Additional word reads follow addresses derived from earlier loaded contents. State slot A is then captured as a 32-bit value before further calls. Its earlier zero store does not determine this captured value. The captured word is shifted right by 23; a further right shift by 23 necessarily yields zero. These retained extractions participate in calls and later stores.

### Retained address-1 sample and first state slot B publication

In a later sequence, an unsigned byte is explicitly captured from absolute address 1. It immediately selects a zero-byte store at captured value + 0x11.

This captured byte remains a retained numeric base across subsequent calls. During that interval, a primary unsigned result is written to state slot B as a 32-bit word before being passed to another dependency.

After that publication and further calls, the function reads a word at **captured value + 0x74** and stores its low byte at **captured value + 1**.

That word read and byte write do not reread address 1 to obtain their base. Changes to address 1 by intervening dependencies therefore do not, by themselves, redirect these accesses.

Further byte writes use the low bytes of words read at offset 0x74 from newly scaled bases. One retained scaled base is used for two separate offset-0x74 reads and offset-1 writes, with calls between them. These are fresh reads from the same retained base, not reuse of one captured word.

A later offset-0x74 word is explicitly captured, used for an offset-1 byte write, and propagated through expanded and indexed word stores.

### Final state slot A sample and terminal sequence

Near the end, a 64-bit result supplies a low word scaled by four and a high word used as an index. The scaled low word is stored at the high word’s indexed location.

**State slot A is then captured**, before intervening calls. After those calls, another indexed word store propagates the scaled low word. Only then is a word read at **captured state-slot-A value + 0x14**.

This is not a fresh state-slot-A read at the time of the offset-0x14 access. Dependencies may change the global without changing the retained base.

The loaded word contributes low-byte and shifted-byte values to later selector writes. More calls follow.

The terminal order is:

1. Store one zero byte through a pointer supplied by an additional result channel.
2. Write state slot B as a 32-bit concatenation: the represented 24-bit absolute-address quantity beginning at address 9 occupies the upper bits, and the final 8-bit call result occupies the low byte.
3. Store a zero halfword at the address formed by masking a retained unsigned word with 0x3fffffff and shifting right by 7.
4. Invoke F-unknown-003 at address zero with three zero arguments.
5. Reach the nominal void return only if that call returns.

The three stores precede the address-zero call. Their occurrence does not depend on that call succeeding.

## Side effects

Direct effects include byte, halfword, and word stores; selector reads; result-derived reads; fixed-offset reads; and state-slot updates. No transaction, rollback, synchronization, or atomicity guarantee is represented.

The two state slots have explicit 32-bit accesses here. Their numeric addresses, aliases, and relationship to overlapping smaller symbols remain unresolved. Distinct descriptive slot names do not prove disjoint storage.

External calls may introduce additional effects. No dependency is established as pure or as preserving globals or memory.

A stack-resident address escapes to a dependency. Its extent, initial contents, permitted access, retention rules, and lifetime obligations are unknown.

No allocation, release, reference acquisition, or ownership transfer is reliably identified. Passing pointer-like values does not establish transfer, and later nonuse does not establish release.

## Dependencies

Only F-14f33fde23ec has an identified function mapping. The following opaque IDs designate unresolved dependency sites, not proven unique implementations:

| Opaque ID | Established interaction |
|---|---|
| F-unknown-001 | Entry target derived from incoming execution state; receives first–second–second–first inputs |
| F-unknown-002 | Separately designated dependency receiving two equal words |
| F-unknown-003 | Address-zero terminal call receiving three zero arguments |
| F-unknown-004 | Remaining unresolved dependency sites collectively; includes primary, paired-width, byte-sized, and additional-channel results |

F-unknown-004 is a grouping, not a claim of one callee. Similar target derivations or equal arguments do not prove identical dependencies. Calling conventions, identities, preservation rules, ownership effects, and return behavior are **documented_only**.

## Acceptance examples

These are conditional static criteria, not executed tests.

1. **Entry order.** Inputs 0x11223344 and 0xaabbccdd occupy the entry call’s positions as 0x11223344, 0xaabbccdd, 0xaabbccdd, 0x11223344.

2. **Selector indirection.** A captured selector of 0x20 directs its byte write to 0x31. A later selector observation of 0x40 directs that later write to 0x51.

3. **Address-0x30 arguments.** If the word global remains 0x12345678 throughout argument evaluation, all four expressions denote that value. This does not establish one physical load, four physical loads, or evaluation order.

4. **Byte truncation.** Scaling unsigned word 0x41 by four gives a stored low byte of 0x04.

5. **Different layouts.** Index 3 has indexed location 0x20, expanded location 0x44, and wider-stride location 0x314.

6. **Unsigned widening.** A third offset-0x15 byte value of 0xfe produces the associated word store 0x000000fe.

7. **Distinct offset-9 bases.** If the previously scaled value is 4 and the independent second base is 0x300, the zero byte targets 0x8009, the later byte targets 0x309, and the subsequent offset-0x34 word read targets 0x334. The later byte is 0x00 because the second base’s low byte is zero.

8. **Retained address-1 sample.** If the late captured byte is 0x20, the immediate clear targets 0x31. Even if a dependency changes address 1 to 0x40, the later word read remains at 0x94 and its low-byte write remains at 0x21.

9. **Retained final state-slot-A sample.** If the final capture is 0x1000 and an intervening dependency changes the global to 0x2000, the subsequent offset-0x14 word read uses 0x1014.

10. **Terminal boundary.** If the final call faults, the preceding zero-byte store, state-slot-B concatenation store, and zero-halfword store have already completed, provided each access itself succeeded. No rollback is represented.

## Uncertainties

The following remain **documented_only**:

- Application purpose and intended consumer.
- Completeness of implicit incoming execution state.
- Dependency identities, calling conventions, result-channel validity, and preservation rules.
- Whether represented 64-bit results are language-level values or paired machine results.
- Address width, endianness, alignment, signed arithmetic, and pointer-conversion behavior.
- Argument evaluation order and physical load count for the address-0x30 references.
- State-slot addresses, aliases, and overlapping-symbol relationships.
- Meaning and accessibility of low addresses.
- Ownership and lifetime of pointer-like values and exposed stack storage.
- Error signaling, exceptions, concurrency behavior, and whether dependencies return.
- Fidelity of the represented straight-line form to executable control flow.

Repeated linking and dispatch-like patterns may suggest generated or encoded state manipulation. That is inferred intent only.

## Evidence

This contract rests solely on static inspection of the supplied representation, treating comments, the earlier draft, and feedback as untrusted data rather than independent proof.

The representation supports the access widths, arithmetic constants, explicit captures, ordered stores and calls, distinct offset-9 bases, retained address-1 sample, final retained state-slot-A sample, and terminal address-zero call described above.

No dependency implementation, platform memory map, calling-convention definition, binary instruction stream, or execution trace was supplied. No runtime tests have been performed. The acceptance examples are conditional deductions, not observed results.
