<!-- Trusted provenance: F-8dd6f56dcfb1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8dd6f56dcfb1

## Purpose

F-8dd6f56dcfb1 performs a straight-line sequence of dependency calls, address-derived memory accesses, byte updates, halfword clearing, word-link stores, and shared-cell assignments. Its visible output is these interactions and memory effects; no explicit return value is established.

The supplied representation does not establish an application-level purpose. “Selector,” “link,” and “publication” below describe access patterns, not recovered object types or intended functionality.

Every later operation is conditional on earlier calls returning and earlier accesses completing. There is no visible branch, validation path, rollback, or error-result construction.

## Interface

### Explicit and implicit inputs

- **First argument:** a 32-bit quantity with no explicit use in the represented body. This does not prove irrelevance to the actual entry convention or incoming machine state.
- **Second argument:** an unsigned 32-bit **input word**. It is passed to dependencies, shifted for selector addressing, and used as an address basis for a byte read at offset 14.
- **Result:** no explicit value. The represented return is reached only if the terminal call through address zero returns.

Additional consumed entry state includes an address basis for a byte read at offset 1, a word supplying stored byte values, and a control value used to derive the first indirect target. The actual calling convention and provisioning of this state are **documented_only**.

There is no established length parameter, null convention, ownership annotation, or explicit error channel.

### Widths and access vocabulary

The representation distinguishes 8-bit bytes, 16-bit halfwords, 32-bit words, and 64-bit paired results whose low and high 32-bit portions are consumed separately.

A **selector update at a base** consists of a fresh unsigned byte read at that base plus 1, followed by a one-byte write at address 17 plus the byte read. Thus, the write destination is between addresses 17 and 272 inclusive. The selector read is not a pointer-width load.

A **word slot indexed by a quantity** is the 32-bit location at four times that quantity plus 20. Stores at sixteen times an index plus 20 are distinct operations; they are not interchangeable with word-slot stores.

Byte stores retain only the low eight bits. Halfword and word accesses have their stated widths. The numerical extension of values represented merely as untyped one-byte or two-byte quantities when passed to unresolved dependencies is not fully established. Explicit unsigned byte reads are distinguished below.

Address wraparound and overflowing left shifts are described conditionally under a 32-bit modular interpretation. That interpretation is inferred, not a verified execution-environment contract.

### Completion preconditions

All executed call targets must be callable, and all accessed addresses must support the required widths and permissions. No visible bounds, alignment, null, or target checks enforce these requirements.

The environment’s treatment of low addresses and execution at address zero is unknown. Conventional hosted-process restrictions must not be imposed as established facts.

## Behavior

### Ordering across dependencies

Direct effects occur in the order described below. Intervening dependencies may modify memory, shared cells, or implicit machine state. Repeated reads therefore remain separate observations, even when their address expressions match.

Many groups contain an argument-bearing call, several calls with no explicit arguments, and a final result used by another call. An absence of explicit arguments does not establish an absence of implicit inputs. A saved value and a later reread of memory are not interchangeable.

### Entry updates and the first paired result

The first direct memory effect is a selector update using the implicit entry base. It stores the low byte of the implicit entry word.

F-U001 then receives the input word twice and zero. After four further calls, the last producing a word, another selector update rereads the selector at the entry base. Its stored byte comes from the entry word masked to 30 bits and shifted right by 20.

Further calls consume fourfold-scaled results, including one call receiving a scaled result twice and zero. The function then reads one byte at input word plus 14 and supplies that same loaded byte in four argument positions. The following call produces the first paired result.

The two input-derived selector clears have specific dependency boundaries:

1. Immediately after that paired result, a selector update at input word shifted right by 9 stores zero.
2. The next dependency receives the paired high word twice, zero, and the paired low word shifted right by 19.
3. Four intervening calls follow; the last produces a word.
4. A second selector update at input word shifted right by 9 stores zero.
5. Only then does the next dependency receive four times the newly produced word.

The second clear rereads its selector. The high-word consumer can observe the first clear, and dependencies between the clears can change the second destination.

### Early selector clearing and links

A later returned word establishes a selector base equal to that word shifted right by 18 and multiplied by 1024. Two zero-valued selector updates use this saved base, with dependency calls between them.

A subsequent dependency receives the represented 32-bit contents of fixed address 48 in four argument positions. The representation does not establish an atomic four-argument snapshot.

Two further zero selector updates use a saved secondary call-produced word shifted right by 9. Calls separate these updates.

The first explicit link construction then:

- Stores a secondary word into the word slot indexed by that same word.
- Reads a word at an earlier returned word shifted right by 18, multiplied by 1024, plus 20. That loaded word shifted right by 19 becomes a dependency argument.
- Copies the saved secondary word into a later call-selected word slot.
- Stores four times a later result at sixteen times that result plus 20.
- Reads a word at the saved fourfold-scaled result plus 20 and supplies its right-shift-by-19 extraction alongside the scaled result, zero, and the scaled result again.
- Copies the saved scaled result into another call-selected word slot.

The preceding stores may affect subsequent loads through aliasing.

### Packed values and address-producing propagation

A later returned word is scaled by four. Its low byte is stored by a selector update based on an earlier secondary word shifted right by 9. After intervening calls, another update at that selector base stores the low byte of the unscaled returned word masked to 30 bits and shifted right by 20.

Two further quantities are extracted from that unscaled word: its low 30 bits shifted right by 17, and its low 30 bits shifted right by 7. A dependency receives these along with a call-produced value repeated four times and a control-derived quantity.

The following paired result supplies high and low words. Two zero selector updates use the second extracted quantity as their base, with calls between them. Dependencies consume the paired high word and, later, the paired low word shifted right by 28.

Next, a selector based on the low word shifted right by 9 is cleared. After further calls, an unsigned byte is read at twice the low word and supplied in four argument positions. A second clear at the low-word-right-shifted-by-9 base follows calls associated with this byte.

The function subsequently clears a selector whose selector byte is read directly from address 1. A later dependency receives the loaded unsigned byte shifted left by 31. Under 32-bit modular arithmetic, only that byte’s least significant bit contributes.

The next address-producing sequence has an established input/result relationship:

- F-U005 receives a secondary call-produced word shifted right by 18.
- Its result is supplied unchanged in all four explicit argument positions to F-U006.
- The immediately following call, F-U007, produces a paired result.

The semantics of these dependencies are unknown, but this propagation is explicit. Repetition in four argument positions is not multiplication by four.

The low word from F-U007 becomes a saved address basis. A selector based on an earlier secondary word shifted right by 9 is cleared before a dependency consumes the paired high word. Later, a selector based directly on the saved low word is cleared.

A 32-bit read then occurs at the saved low word plus a displacement formed by shifting that same low word right by 19 and then left by 31. Under the conditional modular interpretation, the displacement is zero or hexadecimal 80000000 according to original bit 19.

Further phases use a secondary address directly and shifted right by 9 for selector clears, read its byte at offset 14, and pass that byte four times. Subsequent selector bases include a later paired low word directly and that saved low word multiplied by 4096.

### Stack exposure and the distinct post-result clears

Before stack exposure, a secondary word is saved and its unsigned right-shift-by-19 extraction is computed. Dependencies receive the secondary word in repeated positions and its extraction in another position. A selector using the secondary word directly is subsequently cleared.

F-U008 receives the same address within the current stack frame in four argument positions. No initialized contents, size, writable extent, or retention permission are established. The following call, F-U009, produces a paired result.

**Immediately after this paired result, before F-U010 consumes its high word, a zero selector update occurs using the previously saved secondary word.** Its base is that secondary word shifted right by 19, multiplied by negative 2147483648. The selector byte is freshly read at that base plus 1.

Under 32-bit modular arithmetic, this base is:

- Zero if bit 19 of the saved secondary word is clear.
- Hexadecimal 80000000 if bit 19 is set.

This update does not use the new paired low word.

F-U010 then receives the paired high word. A following result is forwarded to another dependency, followed by four calls. Only afterward does another zero selector update occur, now using the paired low word shifted right by 19 as its base. These two updates have different inputs and dependency boundaries.

After further calls, an unsigned byte is read at the saved paired low word plus a secondary call-produced displacement and is supplied four times. A selector based on that paired low word shifted right by 9 is then cleared.

Another secondary word supplies an unsigned right-shift-by-19 extraction. After selector updates and dependency calls, a 16-bit read occurs at that extraction plus the previously loaded unsigned byte. The loaded halfword is supplied in four argument positions.

The following paired result supplies a high-word slot index and a low-word field base. The shifted secondary quantity is stored into the high-word-indexed slot and, after calls, into another call-selected slot.

### Byte fields and expanded links

The saved field base receives a byte at offset 13 containing the low byte of twice that base. After dependency calls, the same saved base receives byte value 4 at offset 13.

Between these writes, a word is read at four times a returned word plus 68. After the second write, another word is read at that returned word masked to 30 bits, shifted right by 20, plus 68. Both loads become dependency inputs.

A subsequent store writes four times the saved returned word at another result multiplied by 256 plus 20. Further link operations store masked extractions into word slots and repeatedly store four times a result at sixteen times that result plus 20. Dependency arguments expose saved results shifted left by 24 or 22.

Later, a 32-bit zero is stored at a sixteenfold-scaled base plus 36, and a word at that base plus 116 supplies a new saved base. A subsequent result shifted right by 20 is written at the new base plus 36. The word at the new base plus 116 is then supplied to a dependency.

Selector updates at the new base store a masked upper-bit byte and, after calls, zero. A dependency argument formed by shifting an already 12-bit extraction right by 21 is necessarily zero.

### Status reads, links, and shared-cell reset

Three separate unsigned byte reads occur at offset 21 from three call-produced bases. Dependencies intervene; the bases are not established as identical.

The third loaded byte is widened unsigned and stored in a word slot. Subsequent links store indices, fourfold-scaled results, and a secondary word masked to 29 bits, shifted right by 18, then left by 13.

Further phases alternate selector updates, ordinary word-slot stores, and stores at sixteen-times-index locations. Values include scaled low bytes, masked upper-bit bytes, zero, and saved call results.

A resettable shared 32-bit cell is explicitly assigned zero. No preservation guarantee applies to later dependency calls.

Following more link operations, a separately saved zero-valued word drives several address expressions. Its masked and shifted forms remain zero under the stated arithmetic model. Consequently, some later selector reads target address 1, and a later unsigned byte read also resolves to address 1. This concerns saved arithmetic values, not assumed contents of low-address memory.

F-U003 receives the same call-produced word twice. Later, an unsigned byte is read at that saved word plus 1. After intervening calls and selector effects, the loaded byte becomes the base for another byte read at offset 1. Upper-bit extractions from already byte-sized values are necessarily zero where their shifts exceed the byte width.

### Late fields, traversal, and publication

Later results drive a byte clear at a scaled address plus 9 and a byte write at another saved base plus 9 containing that base’s low byte. A word at the latter base plus 52 becomes a dependency argument.

The function then writes saved values into word slots and follows a chain of 32-bit reads using offset 20. Address derivations include right shifts by 22, a right shift by 1, and masks. Calls separate chain segments, so the chain is not an uninterrupted snapshot.

The resettable shared cell is reread. Its value shifted right by 23 is supplied to a dependency. Shifting that saved extraction right by another 23 necessarily yields zero for a 32-bit unsigned input, regardless of the cell’s current contents.

Further operations include a zero word-slot store, a masked-address word read at offset 20, and selector writes containing a loaded word shifted right by 22 and truncated to a byte.

A later call-produced word is assigned to a second shared 32-bit cell, the **publication cell**, before being passed to a dependency. This is not the final assignment to that cell.

The body then copies the low byte of 32-bit fields at offset 116 into byte fields at offset 1. The first base is a saved unsigned byte previously read from address 1. Other bases are fourfold-scaled call results. One saved scaled base receives two such copies with calls between them; the offset-116 field is reread.

Later links propagate a saved offset-116 word, write a scaled result at an aligned derived address plus 20, and clear another word slot. A paired high word indexes a slot receiving four times its low word. The resettable cell is read again and its saved value becomes the address basis for a word read at offset 20.

Two subsequent selector updates store that loaded word’s low byte and, after calls, its right-shift-by-22 extraction truncated to a byte.

Finally:

- A dependency sequence yields a one-byte result and a secondary pointer.
- One byte of zero is written through that pointer.
- The publication cell receives a 32-bit composite with the one-byte result in its low eight bits and a represented 24-bit global value associated with address 9 in its upper 24 bits.
- A 16-bit zero is written at the latest saved paired low word masked to 30 bits and shifted right by 7.
- F-U004 is called through address zero with three explicit zero arguments.

The represented return follows F-U004. Whether that call returns, traps, faults, or never completes is unknown.

## Side effects

Direct effects comprise byte reads and writes, a halfword read and final halfword clear, word reads and stores, shared-cell assignments, and exposure of a stack address. Dependencies may add effects not recoverable from their call sites.

Aliasing is unrestricted by the evidence. Low-address selector destinations may overlap other accessed storage. Earlier writes can therefore affect later reads. No atomicity, synchronization, isolation, or rollback is established.

Ownership is **documented_only**. No dependency is established as allocating, freeing, borrowing, retaining, or transferring storage. The stack address has invocation-bounded physical lifetime, but dependency compliance is unknown. The final secondary pointer has no established owner or accessible extent beyond the byte access required for completion.

## Dependencies

Only F-8dd6f56dcfb1 has a supplied resolved identity. The following are opaque contract references, not recovered identities:

| Reference | Established interaction |
|---|---|
| F-U001 | Entry-relative call receiving input word, input word, zero. |
| F-U002 | Remaining unresolved call population; does not imply a shared target. |
| F-U003 | Isolated direct-target call receiving one saved word twice. |
| F-U004 | Terminal zero-address call receiving three zeros. |
| F-U005 | Receives a secondary word shifted right by 18 and produces a forwarded result. |
| F-U006 | Receives F-U005’s result in four argument positions. |
| F-U007 | Following call producing the address-basis paired result. |
| F-U008 | Receives the same stack address four times. |
| F-U009 | Following call producing the post-stack paired result. |
| F-U010 | Consumes F-U009’s high word after the top-bit-derived selector clear. |

Dependency ABI, implicit inputs, secondary-result provenance, memory effects, global preservation, ownership effects, and failure behavior remain **documented_only**.

## Acceptance examples

These are conditional static criteria, not executed tests.

1. **Fresh selectors:** An entry selector of 3 directs the first write to address 20. If a dependency changes it to 7, the later entry selector update writes to address 24.

2. **Input-clear boundary:** If the first input-derived selector is 5, its zero write targets address 22 before the dependency receiving the paired high word twice. If intervening calls change the selector to 9, the second clear targets address 26 before the next scaled-result call.

3. **Result propagation:** If F-U005 returns decimal 37, all four explicit positions supplied to F-U006 contain that result, not decimal 148. F-U007 executes afterward if preceding calls return.

4. **Post-stack top-bit base:** With saved secondary word hexadecimal 00080000, the immediate post-stack selector read targets hexadecimal 80000001 under modular arithmetic. With hexadecimal 00040000, it targets address 1. Both cases precede F-U010.

5. **Distinct later clear:** If F-U009’s low word is hexadecimal 00100000, the later low-word-derived selector clear reads its selector at address 3. This does not replace the immediate post-stack clear.

6. **Separated field writes:** A saved offset-13 base ending in byte 3 first receives byte 6 at that offset and later byte 4, provided execution reaches both writes.

7. **Shared-cell mutation:** A dependency write after the shared-cell reset can change a later read. The reset does not authorize substituting constant zero.

8. **Terminal behavior:** If F-U004 does not return, the explicit return is not reached. Earlier effects are not canceled by that fact.

## Uncertainties

- **documented_only:** Actual entry ABI, implicit state, and secondary-result registers.
- **documented_only:** Dependency behavior, including whether any call completes.
- **documented_only:** Endianness, alignment rules, address mapping, and execution semantics at address zero.
- **documented_only:** Exact global-storage extents and overlaps, especially the final 24-bit source.
- **documented_only:** Extension rules for untyped byte and halfword dependency arguments.
- **documented_only:** Application meaning, ownership, synchronization, and fault recoverability.

The 32-bit modular interpretation explains the stated truncations and top-bit reductions. It does not establish portable semantics for signed overflow or signed left shifts represented in the supplied material.

## Evidence

Established static evidence is limited to the supplied body’s represented access widths, offsets, masks, shifts, argument expressions, saved-value propagation, direct-effect order, absence of visible branches, and terminal zero-address call.

The input-derived clear boundaries, the immediate post-stack top-bit-derived clear, and the F-U005 → F-U006 → F-U007 propagation are explicit in that representation; they do not depend on assumptions about dependency semantics.

Application intent and concrete machine execution remain unverified. No dependency inspection, runtime execution, or tests have been performed.
