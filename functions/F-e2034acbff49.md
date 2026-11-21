<!-- Trusted provenance: F-e2034acbff49; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e2034acbff49

## Purpose

F-e2034acbff49 executes a sequence of dependency calls, field extractions, computed-address reads, and byte, halfword, and word stores. The supplied static evidence does not establish an application-level purpose such as parsing, allocation, initialization, or dispatch.

Its behavior depends on implicit entry state, dependency results, secondary result channels, and memory observations. Explicit arguments alone do not determine its addresses or stored values.

No conditional branches or loops are represented. Reaching each operation nevertheless requires preceding calls to return and preceding accesses to complete. The terminal operation is a call to address zero with three zero arguments; normal return is established only conditionally on that call returning.

## Interface

### Explicit inputs and output

The represented interface accepts two 32-bit quantities and declares no return value.

- The first explicit input has no established use.
- The second explicit input is forwarded as the first argument of F-opaque-001.
- Neither input is identified as an output buffer, length, error destination, or ownership-transfer parameter.
- No meaningful final return-register value is established.

This is not a complete machine-level calling convention.

### Implicit inputs and storage

Execution additionally consumes:

- Entry control-flow state used to determine the initial call target.
- An entry unsigned word, logically shifted right by 19 for the initial call.
- An entry address used for a selector-byte read at offset 1.
- An entry word used in the first explicit word store.
- Primary, secondary, and doubleword dependency results.
- Stack-resident storage whose address is passed externally.
- Shared storage, absolute low-address memory, and overlapping shared representations.

No object layout, allocation extent, or ownership relation is established merely because a numerical value is used as an address.

### Widths and arithmetic

Bytes, halfwords, words, and doublewords have widths of 8, 16, 32, and 64 bits respectively. An explicitly unsigned byte read produces a value from 0 through 255. A byte store retains the low eight bits of its supplied value.

Established unsigned word arithmetic uses 32-bit bit-vector semantics. Logical right shifts introduce zeros. Unsigned left shifts and multiplication retain the low 32 bits. A doubleword’s low and high words are its least and most significant 32 bits.

Some address expressions, multiplications, and left shifts involve signed representations. Their exact overflow and signed-shift semantics are **documented_only**. Arithmetic examples below use small nonnegative operands unless they concern explicitly unsigned operations.

Endianness, alignment requirements, address mapping, access permissions, and executable-target validity are not established.

### Preconditions and errors

There is no represented local validation, error-status return, recovery branch, or cleanup path. Continued execution requires all consumed implicit values and result channels to be meaningful under the actual calling convention.

An invalid address, inaccessible memory, alignment violation, invalid call target, or nonreturning dependency may prevent subsequent operations. The resulting fault, exception, termination, or recovery behavior is **documented_only**.

## Behavior

### Ordering and selected-byte operations

Descriptions below are conditional on execution reaching the relevant operations.

A recurring selected-byte update first reads an unsigned byte from a computed selector address, then stores one byte at the numerical address equal to that selector value plus 17. Its destination is therefore between addresses 17 and 272 inclusive. This is not an offset added back to the selector’s original base.

Every represented selector read is a separate observation. Repeated address expressions do not authorize reusing an earlier value. Dependencies may change selectors, selected destinations, or other storage.

The ordering below describes explicit access and call order. It does not establish atomicity, persistence, synchronization, or externally visible commit order.

### Initial calls and scaled word stores

F-opaque-001 is called at a target derived from entry control-flow state plus 4094. Its arguments are the second explicit input and the entry unsigned word logically shifted right by 19.

After intervening calls, a result determines a word destination at four times that result plus 20. The entry word is stored there before a call receiving four times the result.

A later result is multiplied by four, and that scaled value is stored at sixteen times the original result plus 20. The following argument-bearing call receives:

- The four-times-scaled value.
- A newly read unsigned word at the scaled value plus 20, shifted right by 19.
- Zero.

The store and read generally use different addresses. A subsequent result selects another destination at four times that result plus 20, where the retained scaled value is stored.

### Early fields and the first doubleword boundary

A later unsigned result supplies its four-times-scaled value and three fields extracted from its lower 30 bits:

- Right shift by 17: at most 13 significant bits.
- Right shift by 20: at most 10 significant bits.
- Right shift by 7: at most 23 significant bits.

The selector at the entry address plus 1 is read, and the low byte of the scaled value is stored at selector plus 17. After calls, the selector is read again, and the low byte of the right-by-20 field is stored at the newly selected destination. That field can exceed 255 and therefore can truncate.

A later seven-argument call receives one retained result four times, followed by the right-by-17 field, the right-by-7 field, and a represented control-flow-derived value.

The ordering around the following doubleword result is material:

1. F-opaque-004 returns the first doubleword described here. Its low word is retained for later calculations; its high word supplies later call arguments.
2. The routine reads the selector at the retained lower-30-bits-right-by-7 field plus 1.
3. It clears the byte at that selector value plus 17.
4. Only then does F-opaque-005 receive the doubleword’s high word twice and the earlier retained result used in the seven-argument call.
5. Further calls occur, including a call producing a word and another receiving that word.
6. After additional calls, the routine freshly reads the same field-plus-1 selector expression and clears the byte at this newly observed selector plus 17.
7. A following call receives a four-times-scaled intervening result.

The first clear must not be moved after F-opaque-005. That dependency may observe the cleared location or change the selector before the second read. The two clears need not target the same address.

Further selected-byte clears use shifted retained results and absolute address 1, with calls between repeated observations.

### Byte, word, halfword, and stack observations

With calls and selected-byte clears interspersed, significant observations occur in this order:

- An unsigned byte at twice a retained unsigned word is passed four times to a dependency.
- A result obtained from a dependency receiving a secondary unsigned result shifted right by 18 is passed four times.
- A word at a retained unsigned base plus a high-bit contribution is passed four times.
- An 8-bit value at offset 14 from a secondary result is passed four times.
- A returned word is passed four times.
- The same stack-storage address is passed four times.
- An unsigned byte at a retained base plus a secondary result is passed four times.
- A halfword at the preceding byte value plus a secondary unsigned result shifted right by 19 is passed four times.

The high-bit contribution to the word-read address is formed by shifting the retained unsigned base right by 19 and then left by 31. Under 32-bit unsigned semantics, this contribution is either zero or 0x80000000.

No explicit initialization of the exposed stack storage is established before its address is passed. Its extent, contents, retention requirements, and subsequent treatment are unresolved.

After the halfword-related call, a new doubleword result supplies a high word used to select a destination at four times that high word plus 20. The retained secondary result shifted right by 19 is stored there. After calls, the same retained value is stored at another result-derived destination.

### Direct offset-13 writes and further scaling

The low word of that doubleword is retained as an address. Two byte writes target its offset 13:

- First, the low byte of twice the retained address.
- Later, after calls, the constant 4.

Between these writes, a word at four times another unsigned result plus 68 is read and passed onward. After the second write, a separate word read uses that unsigned result’s lower 30 bits shifted right by 20, plus 68.

Further word stores include:

- A four-times-scaled unsigned result at 256 times another unsigned result plus 20.
- A lower-26-bit field shifted right by 13 at four times a later result plus 20.
- Four-times-scaled results at sixteen times their original values plus 20.
- Retained scaled values at destinations selected by later results multiplied by four, plus 20.

These are computed accesses, not evidence of a particular table, list, or object representation.

### Offset-36 and offset-116 state

A sixteen-times-scaled result region receives a zero word at offset 36. A word is then read at offset 116 of that region and retained as another address.

Following calls, an unsigned result shifted right by 20 is stored at offset 36 from that retained address. A word at offset 116 from the retained address is then read and passed to a dependency.

A selector at retained address plus 1 selects a byte destination for a truncated field store. After calls, the same selector expression is freshly observed and the newly selected destination is cleared.

The unsigned right-by-20 result is at most 4095. Its subsequent right shift by 21 is necessarily zero; this conclusion requires no assumption about memory preservation.

Three distinct unsigned byte reads at offset 21 from secondary results follow. All feed calls; the third byte is also retained and later stored as a zero-extended word at a four-times-derived destination plus 20.

Another secondary unsigned result contributes its lower 29 bits shifted right by 18. This field is at most 2047 and participates in subsequent scaling, stores, and selector-address calculations.

### Intermediate shared and low-address state

Further stores publish retained results, secondary results, scaled fields, and zeros.

A shared word, designated **shared word A**, is explicitly set to zero. Calls and other operations precede later reads of that word. Each later read is a fresh observation; the explicit zero does not determine its value after unknown dependency effects.

Separately retained arithmetic zeros produce zero arguments and selector addresses equal to absolute address 1. These consequences follow from retained values, not from shared word A remaining zero. The byte at address 1 can change between observations.

F-opaque-002 receives the same retained unsigned word twice. Later accesses include:

- An unsigned byte at that retained word plus 1.
- An 8-bit value at the preceding unsigned byte value plus 1.
- An unsigned byte at a further field-derived address plus 1.

The second address is between 1 and 256. The access is byte-sized, not a pointer-sized link traversal.

Some later extracted fields are necessarily zero because their retained inputs are already zero or sufficiently narrow. This does not eliminate calls or memory reads in the represented sequence.

### Late address-derived updates

A byte-sized dependency result is shifted left by two for a selected-byte store. Its signed promotion and shift behavior are **documented_only**.

Other late operations include:

- Clearing a byte at a four-times-scaled result multiplied by 8192, plus 9.
- Writing the low byte of a retained address at that address plus 9.
- Reading a word at offset 52 from that address and supplying it with split doubleword results to a dependency.
- Storing a word at four times a secondary result plus 20, then separately reading a word at the secondary result plus 20.
- Reading words through right-shift-derived addresses and using newly observed words in later destinations.

A fresh read of shared word A is logically shifted right by 23, leaving at most nine significant bits. Shifting that retained result right by another 23 yields zero regardless of the value originally observed.

### Final shared state and word-to-byte transfers

A second shared word, **shared word B**, receives an unsigned dependency result before that result is passed to another dependency.

A base retained from an unsigned byte read at absolute address 1 is used for a word-to-byte transfer: a word at base plus 116 is read, and its low byte is stored at base plus 1. The base is a byte-derived numerical address, not an established object pointer.

Similar transfers use four-times-scaled dependency results. One retained base is used twice, with calls between the word reads. Both reads remain separate observations. Another transfer retains the complete loaded word for later word stores.

Additional stores include retained full words, a scaled result at a destination formed by clearing another retained value’s low two bits and adding 20, a zero word at a computed destination, and a scaled doubleword low word at a destination derived from its high word.

Shared word A is read again. After intervening operations, a word is read at that retained shared-word value plus 20.

### Terminal sequence

Near termination:

1. A selected byte receives the low byte of the word obtained through the fresh shared-word-A observation.
2. Calls intervene.
3. Another selected byte receives the low byte of that retained word shifted right by 22.
4. Further calls produce an 8-bit result.
5. A byte is cleared through a pointer supplied by a secondary result channel.
6. Shared word B receives a constructed word whose low byte is that 8-bit result and whose upper 24 bits come from a separately represented shared 24-bit value.
7. A halfword is cleared at an address formed from a retained unsigned word’s lower 30 bits shifted right by 7.
8. F-opaque-003 is called at address zero with three zero arguments.
9. Return follows only if F-opaque-003 returns.

The pointed-byte clear precedes use of the shared 24-bit value in construction of shared word B. Aliasing can therefore affect the construction. Overlapping shared representations must not be presumed independent.

## Side effects

Established direct effects include byte, halfword, and word stores; shared-word assignments; absolute low-address accesses; and exposure of a stack address to a dependency.

In particular, the first field-selected clear after F-opaque-004 occurs before F-opaque-005. Unknown dependency effects do not permit reversing that order or merging it with the later clear.

No local rollback, synchronization, release operation, or error-status publication is established. Dependency effects may overwrite, restore, retain, or otherwise affect storage; this contract assigns no narrower effect set.

**Ownership is documented_only.** No dependency is established as an allocator, destructor, retaining consumer, or releasing consumer. Passing an address or storing it does not establish transfer of ownership.

**Persistence and external visibility are documented_only.** The evidence establishes neither atomicity nor non-atomicity, and does not establish what an observer sees following a failed access or nonreturning call.

## Dependencies

These are specification-local opaque IDs, not verified dependency identities.

| ID | Established role | Unresolved contract |
|---|---|---|
| F-opaque-001 | Initial call through entry-control-state-derived target | Calling convention, target validity, effects, and return behavior |
| F-opaque-002 | Fixed-target call receiving the same unsigned word twice | Purpose, effects, preservation, and ownership |
| F-opaque-003 | Terminal call to address zero with three zero arguments | Callability, effects, and return behavior |
| F-opaque-004 | Produces the first doubleword described in the early-field phase | Meaning and validity of both result words; other effects |
| F-opaque-005 | Receives that high word twice and an earlier retained result, after the first field-selected clear | Effects on the cleared destination, selector storage, and other state |

The remaining call occurrences are designated **F-opaque-rest-\***, with a distinct unresolved occurrence identity rather than an assumed common callee.

Represented call interfaces include empty explicit argument lists, repeated words, zeros, scaled values, stack addresses, seven arguments, secondary result channels, and doubleword results. An empty explicit argument list does not establish absence of machine-level inputs. Similar target calculations do not establish interchangeable dependencies.

## Acceptance examples

These are conditional analytical criteria, not executed tests. Each assumes execution reaches the relevant operations and the stated accesses complete.

1. **Initial forwarding.** For second input 0x12345678 and entry unsigned shift-source 0xFFFFFFFF, F-opaque-001 receives 0x12345678 and 0x00001FFF. The first explicit input has no established contribution.

2. **Selected address and truncation.** A selector value of 200 selects numerical address 217. An early unsigned result of 0x00123456 scales to 0x0048D158, whose byte store writes 0x58.

3. **Ten-bit field.** Masking 0x3FF00000 to its lower 30 bits and shifting right by 20 gives 1023. A byte store of this field writes 0xFF.

4. **Clear before high-word forwarding.** Suppose the retained right-by-7 field is 128 and the selector at address 129 initially contains 10. Immediately after F-opaque-004 returns, the routine reads address 129 and clears address 27 before calling F-opaque-005. If F-opaque-005 reads address 27 before changing it, with no intervening external modification, it observes zero.

5. **Fresh selector after dependency effects.** Continue the preceding example with F-opaque-005 changing the selector at address 129 to 20. If subsequent calls leave that selector unchanged, the second selector observation selects address 37 for clearing. Reusing the first selector or postponing the first clear until after F-opaque-005 would not preserve the specified behavior.

6. **Different scales.** For a relevant result of 3, sixteen-times scaling plus 20 gives address 68; four-times scaling plus 20 gives address 32. A store to the former cannot generally replace a read from the latter.

7. **Width-derived zeros.** An unsigned word shifted right by 20 and then by 21 becomes zero. An unsigned word shifted right by 23 twice also becomes zero. Neither conclusion assumes shared word A remains zero.

8. **Fresh shared observation.** If a dependency changes shared word A and that change is available to a later read, the later calculation uses the changed value, not the earlier explicit zero.

9. **Terminal composition and boundary.** If the final 8-bit result is 0xAB and the shared 24-bit value observed after the pointed-byte clear is 0x123456, the constructed word is 0x123456AB. Normal return remains conditional on the subsequent call to address zero returning; persistence of earlier stores is unspecified.

## Uncertainties

- **documented_only:** Complete calling convention, implicit entry requirements, and validity of secondary result channels.
- **documented_only:** Individual unresolved dependency identities, effects, preservation, failures, and return behavior.
- **documented_only:** Signed overflow, signed shifts, and byte-promotion semantics where signed representations occur.
- **documented_only:** Endianness, alignment, address mappings, permissions, and executable low-address mappings.
- **documented_only:** Complete shared-storage layout, overlap, and alias relationships.
- **documented_only:** Stack-storage extent, initialization, lifetime, retention, and ownership.
- **documented_only:** Concurrency, atomicity, transactions, fault handling, persistence, and external visibility.
- **Inferred intent:** Repeated stores may publish internal state. No application-level or data-structure interpretation is established.

These unknowns do not obscure the established ordering of the first selector read and clear before F-opaque-005, followed by a separate selector read and clear after additional calls.

## Evidence

Established claims derive from explicit access widths, argument positions, result usage, arithmetic masks and shifts, access/call ordering, and the supplied warning about overlapping shared representations.

The early doubleword boundary establishes a selector read and byte clear before high-word forwarding, followed by another selector read and clear after intervening calls. No preservation assumption is needed for that ordering.

Arithmetic conclusions use stated widths and unsigned semantics. Dependency behavior, signed-operation details, ownership, and environmental guarantees remain unresolved rather than inferred from patterns.

No runtime execution or runtime tests have been performed. Acceptance examples are analytical conditions, not observed outcomes.
