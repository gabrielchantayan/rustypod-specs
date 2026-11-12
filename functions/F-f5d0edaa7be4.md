<!-- Trusted provenance: F-f5d0edaa7be4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f5d0edaa7be4

## Purpose

F-f5d0edaa7be4 performs an ordered sequence of mixed-width memory observations and publications, prepares selected words in stack-relative storage, and invokes two dependencies. If both dependencies return and execution continues, it obtains a callable target from a terminal operation and invokes that target without a represented return path.

No application-level purpose is established. Recurring offsets suggest structured mutable state, but do not establish a record type, allocation scheme, protocol, or device interface.

This is a **documented_only** contract derived from static evidence. It does not establish that the represented entry boundary is a complete callable interface or that a valid invocation can reach every described operation.

## Interface

### Explicit inputs

| Input | Represented width | Uses |
|---|---:|---|
| Primary input | 32 bits | Address base for word reads at offsets 20, 100, and 116; destination base for a byte write at offset 1; operand of left shifts by 4 and 8 |
| Secondary input | 32 bits | Address contribution for the initial byte read; source of an initial narrowed byte write |

Neither input is validated. Neither is established to identify an allocated object.

### Additional entry state

Execution also consumes:

- An entry base used for addressing and self-valued publications.
- An entry index/base used for addressing and stored data.
- A saved entry word used as data and later as an address base.
- An entry-supplied indirect callable target.
- Five distinct global word locations encountered during the update sequence.
- A separate global location supplying an argument to F-cdec09f5d432.
- Existing memory at the derived addresses.
- Stack-relative storage, including bytes not initialized before exposure to a dependency.

The explicit inputs alone therefore do not specify the invocation state.

### Widths and arithmetic

Byte, halfword, and word accesses have represented widths of 8, 16, and 32 bits. Address-bearing integer operations are represented at 32-bit width. All offsets below are byte offsets.

Halfword observations are signed and widen with their sign. For example, a halfword pattern of `0xFFFE` contributes −2 to subsequent arithmetic.

The initial byte observation has unresolved signedness when widened. A pattern of `0x80` can therefore contribute either 128 or −128. This affects addresses derived from that observation and its later halfword publication.

Narrow writes retain the low destination-width bits; they do not saturate. Arithmetic is not unlimited precision. General wrapping behavior, signed-overflow behavior, and the validity of left shifts of negative values are not established. Examples involving derived addresses are conditional on valid arithmetic and accessible memory.

### Completion and errors

No ordinary return value or normal return path is represented.

An invalid access, arithmetic fault under the applicable execution semantics, or nonreturning dependency may prevent subsequent effects. No validation, error status, recovery branch, cleanup, or rollback is represented.

## Behavior

### Ordering and retained observations

The following ordering describes the function’s own represented accesses. Different address expressions are not assumed to designate disjoint storage.

An explicitly retained observation remains that value despite later memory changes. An explicit reread observes memory at its later position in the sequence. Byte and halfword publications may affect subsequent word observations through aliasing.

Each of the five global-selection stages contains two observations of its global location: one selects the address-bearing value, and another supplies the word published into an offset-68 field. Their equality is not guaranteed by the supplied representation.

Primary offset 100 is read exactly four times:

| Observation | Subsequent byte use |
|---|---|
| First | Early publication at entry-base offset 17 |
| Second | Publication after the second global selection |
| Third | Publication after the fourth global selection |
| Fourth | Two publications separated by intervening memory updates |

There is no fifth observation and no refresh between the two uses of the fourth observation.

### Initial publications and first global selection

The initial observations occur in this order:

1. A word at primary offset 100.
2. A signed halfword at the entry base plus the entry index/base.
3. A byte at the secondary input plus that signed halfword.

The widened byte supplies a further address base. The function then publishes, in order:

- The entry index/base’s low byte at primary offset 1.
- The secondary input’s low byte at entry-index/base offset 13.
- The saved entry word at entry-index/base offset 68.
- The entry index/base’s low byte at byte-derived-base offset 13.
- The widened byte, narrowed to a halfword, at the entry index/base plus the signed halfword.

It next rereads entry-index/base offset 68. The intervening narrow publications can overlap that word, so this observation is not interchangeable with the saved entry word.

Entry-base offset 84 then receives the entry base itself. The first global address is selected; a separate observation of that global is published at byte-derived-base offset 68. The selected global address receives the retained readback at offset 20. Finally, entry-base offset 17 receives the low byte of the first primary-offset-100 observation.

### Progression to the second selection

The second primary-offset-100 observation is followed by:

- A word observation at entry-base offset 68, supplying a loaded field base.
- A signed halfword at the entry base plus the first selected global address.
- A word at offset 116 of the earlier readback, supplying a traversal base.
- A word at entry-base offset 20, supplying the current destination.

The traversal base’s low byte is written at loaded-field-base offset 13. The loaded field base is then narrowed to a halfword and written at the signed-halfword value plus the traversal base.

A word at traversal-base offset 68 is retained for the next global publication. The current destination receives itself at offset 84.

The second global address is selected. A separate global observation is published at loaded-field-base offset 68, followed by publication of the retained traversal word at selected-global offset 20. The current destination then receives the second primary observation’s low byte at offset 17.

### Progression to the third selection

The function reads current-destination offset 68 and a signed halfword at the current destination plus the second selected global address.

Before traversing offset 116 of the newly loaded field base, it:

- Writes the second selected global address’s low byte at offset 21 of the word retained for the preceding offset-20 publication.
- Stores that retained word at second-selected-global offset 68.

The new offset-116 observation supplies a traversal base. The loaded field base is narrowed into a halfword at the signed-halfword value plus this traversal base. Traversal-base offset 68 is then observed.

A current-destination self-publication at offset 84 precedes the third global selection. The separate global-content publication goes to loaded-field-base offset 68; the traversal word goes to selected-global offset 20.

Current-destination offset 17 then receives the low byte of twice that destination. This byte does not come from primary offset 100.

### Progression to the fourth selection

The third primary-offset-100 observation precedes a current-destination offset-68 observation and a signed-halfword observation at the current destination plus the third selected global address.

A fresh word at offset 20 of the signed-halfword-derived base replaces the current destination. This address-producing read precedes the following publication group:

1. The loaded offset-68 base receives itself at offset 84.
2. The saved entry word is stored at the third selected global address plus the carried traversal word.
3. Third-selected-global offset 20 receives the loaded base.
4. Saved-entry-word offset 68 is read, and its low byte is written at third-selected-global offset 13.
5. The loaded base’s low byte is written at the replacement destination’s offset 13.
6. The loaded base is narrowed to a halfword at the signed-halfword value plus the third selected global address.

Third-selected-global offset 68 is then read. The replacement destination receives itself at offset 84.

The fourth global selection follows. Its separate global-content observation is published at loaded-base offset 68, and the preceding global-derived offset-68 observation is published at selected-global offset 20. The replacement destination receives the third primary observation’s low byte at offset 17.

### Progression to the fifth selection

The fourth and final primary-offset-100 observation occurs first; its low byte is retained for two publications.

The function next reads current-destination offset 68 and a signed halfword at the current destination plus the fourth selected global address. It then:

1. Stores the current destination at offset 84 of the carried word used in the preceding global offset-20 publication. This is a cross-base publication, not a self-publication.
2. Writes the retained primary byte at current-destination offset 17.
3. Stores the fourth selected global address at the newly loaded base’s offset 68.
4. **Reads a fresh word at offset 20 of the signed-halfword-derived base, replacing the current destination.**
5. Publishes the newly loaded base into its own offset 84.

The fresh offset-20 read is therefore after the offset-68 store and before the loaded base’s self-publication. If the store aliases its source field, the read observes the resulting memory state. It is not a reuse of the destination established before the fourth selection.

The second saved-word publication group then:

- Stores the saved entry word at the fourth selected global address plus the carried word.
- Stores the newly loaded base at fourth-selected-global offset 20.
- Freshly reads saved-entry-word offset 68 and publishes its low byte at fourth-selected-global offset 13.
- Publishes the newly loaded base’s low byte at the **replacement destination’s** offset 13.
- Publishes the newly loaded base, narrowed to a halfword, at the signed-halfword value plus the fourth selected global address.

Fourth-selected-global offset 68 is then observed. The replacement destination receives itself at offset 84.

The fifth global address is selected. Its separate global-content publication goes to the newly loaded base’s offset 68; the preceding global-derived observation goes to selected-global offset 20. The retained fourth-primary-observation byte is finally written at the **replacement destination’s** offset 17.

Thus the fresh offset-20 observation determines the subsequent offset-13 byte destination, offset-84 self-publication destination, final retained-byte destination, and destination carried into the arithmetic-dependent portion.

### Arithmetic-dependent addressing

The function reads current-destination offset 68, then a signed halfword at the current destination plus the fifth selected global address.

It performs this pair twice consecutively:

- Store the signed-halfword value at offset 84 of the address represented by that value.
- Write the current destination’s low byte at fifth-selected-global offset 17.

Both pairs remain represented effects even when their addresses and values coincide.

The signed-halfword value is multiplied by the preceding current-destination offset-68 observation. From that product-derived address, successive word reads at offsets 68, 116, and 20 produce a field base. A further word at field-base offset 20 is retained, and that field is replaced with twice the current destination.

The replaced word remains an address-bearing observation. Its offset-20 word is read and multiplied by the signed-halfword value to obtain another product base.

Before reading this product base’s offset 116, the function again performs the signed-base offset-84 self-publication and writes the replaced word’s low byte at fifth-selected-global offset 17.

The resulting offset-116 observation is assigned to stack-relative offset 264 and retained for later publications. The product base is multiplied by the signed-halfword value again. Another signed-base self-publication follows, then the retained offset-116 word’s low byte is written at fifth-selected-global offset 17.

The remaining ordered effects are:

1. Read the latest product base’s offset 68.
2. Read primary offset 116.
3. Store the retained arithmetic-tail offset-116 word at fifth-selected-global offset 100.
4. Write the earlier product-field observation’s low byte at fifth-selected-global offset 9.
5. Reread product-base offset 68 and publish its low byte at that product base’s offset 17.
6. Perform another signed-base offset-84 self-publication.
7. Write the retained arithmetic-tail offset-116 word’s low byte at twice the product base plus 17.
8. Multiply the product base by the signed-halfword value and read the resulting address’s offset 68.
9. Read primary offset 20 to obtain a final destination.
10. Write the primary-offset-116 observation’s low byte at offset 1 of the retained arithmetic-tail offset-116-derived base.
11. Write the final product-derived offset-68 observation’s low byte at final-destination offset 17.
12. Write twice the preceding product base, narrowed to a byte, at final-destination offset 9.

There are five signed-base offset-84 self-publications in this portion. Fifth-selected-global offset 17 receives four byte publications: the current-destination byte twice, the replaced-word byte, and the retained offset-116 byte.

### Stack-relative preparation

All listed assignments are 32-bit words. Offsets are relative to the pointer subsequently passed to F-000000000001.

| Offsets | Assigned value |
|---|---|
| 16, 36 | Primary input shifted left by 4 |
| 52, 76, 316, 392 | Primary input shifted left by 8 |
| 264, 296, 380 | Retained arithmetic-tail offset-116 word |
| 96, 168, 228, 440 | Zero |

Offset 264 is assigned during the arithmetic-dependent portion. The final batch assigns offsets 16 and 52 first, then zeros at 228, 96, 168, and 440, then assignments at 36, 76, 296, 316, 380, and 392.

The assigned extent reaches byte 443. This does not establish an allocation size, complete record layout, or initialization of gaps. These are ordered assignments, not unconditional final contents when other accesses can alias the area.

### Dependencies and terminal transfer

F-000000000001 receives the stack-relative pointer and two zeros. No result is explicitly consumed.

Only after it returns are these stack-relative words assigned, in order:

- Zero at offset 8.
- An opaque fixed code-location word at offset 412.
- Zero at offsets 256, 288, and 320.

F-cdec09f5d432 then receives a word read from the separate global argument location and three zeros. This argument is obtained after the first dependency returns, not from a pre-call snapshot.

If F-cdec09f5d432 returns, F-000000000002 receives zero and a fixed location-bearing operand. Its result supplies the callable target invoked as F-000000000003. No explicit invocation arguments are represented. The resulting invocation is marked nonreturning.

## Side effects

The function publishes bytes, halfwords, and words through entry-supplied, loaded, summed, multiplied, doubled, global-derived, and stack-relative addresses.

Disjointness is not established between destinations, later sources, global locations, or the exposed stack area. Repeated accesses and apparently redundant publications therefore remain significant. This does not establish volatile, atomic, or device-memory semantics.

Alignment requirements, byte order, unaligned-access support, access atomicity, and fault handling are unresolved. No alignment repair is represented.

No allocation or release is directly identifiable. Dependencies are not constrained to preserve globals, retain allocations, or avoid allocation and release.

Passing the stack-relative pointer establishes neither ownership transfer nor permission to retain it. Dependency copying, asynchronous access, lifetime requirements, and release behavior remain unknown. Failures can leave earlier publications observable; rollback is not represented.

## Dependencies

| Opaque function ID | Established role |
|---|---|
| F-000000000001 | Entry-supplied indirect target receiving a stack-relative pointer and two zeros |
| F-cdec09f5d432 | Dependency receiving a post-first-call global observation and three zeros |
| F-000000000002 | Terminal target-producing operation receiving zero and a fixed location-bearing operand |
| F-000000000003 | Produced target, invoked without represented explicit arguments and marked nonreturning |

The synthetic IDs distinguish unresolved roles, not verified implementation identities. No dependency has an established successful-return, ownership, global-preservation, or memory-effect contract.

## Acceptance examples

These are conditional static criteria, not executed tests.

1. **Narrowing:** A first primary-offset-100 observation of `0x123456AB` supplies `0xAB` to its early offset-17 publication.

2. **Signed halfword:** An initial halfword pattern of `0xFFFE` contributes −2 to the secondary-input-based byte address.

3. **Unresolved byte widening:** An initial byte pattern of `0x80` does not establish whether the later halfword publication contains `0x0080` or `0xFF80`.

4. **Initial reread:** If an intervening narrow publication changes entry-index/base offset 68, the subsequent word observation uses the resulting memory, not necessarily the saved entry word.

5. **Four primary observations:** A fifth explicit primary-offset-100 read is outside the represented behavior. Changes after the fourth read do not refresh either retained-byte publication.

6. **Fresh fifth-stage destination:** If the store of the fourth selected global address into the newly loaded base’s offset 68 aliases the signed-halfword-derived offset-20 field, the following fresh word read obtains the post-store value. That value supplies the later offset-13, offset-84, and retained-byte offset-17 destinations. The earlier offset-17 publication still uses the preceding destination.

7. **Later aliasing does not replace the retained destination:** If the loaded base’s subsequent offset-84 self-publication changes the field from which the replacement destination was read, the already observed destination remains in use; no additional refresh is represented there.

8. **Fresh saved-word observation:** A change to saved-entry-word offset 68 between the two publication groups can change the second group’s byte publication.

9. **Duplicate effects and rereads:** Both consecutive arithmetic publication pairs remain present. A later explicit product-field reread observes intervening aliased writes rather than reusing the first observation.

10. **Partial call area:** With nonoverlapping storage, the listed pre-call assignments have occurred before F-000000000001. Gaps are not promised to be zero, and the listed post-call assignments have not yet occurred.

11. **Dependency effects:** If F-000000000001 changes the separate global argument location and returns, F-cdec09f5d432 receives the later observation. If the first dependency does not return, subsequent assignments and calls are not reached.

12. **Terminal behavior:** If both ordinary dependencies return and terminal processing proceeds, the produced target is invoked instead of an ordinary success value being returned.

## Uncertainties

The following remain **documented_only**:

- Complete calling convention and validity of the represented entry boundary.
- Whether additional entry state is intentional or incompletely recovered.
- Initial byte signedness and overflowing arithmetic semantics.
- Valid address domain, alignment, byte order, and fault behavior.
- Global stability, initialization, and alias relationships.
- Object boundaries and meanings of recurring offsets.
- Complete stack layout and significance of fixed location-bearing operands.
- Dependency memory effects, ownership, retention, and return behavior.
- Concrete terminal outcome, including whether it corresponds to a trap, handler transfer, halt, or another mechanism.
- Whether the arithmetic-derived addresses occur in intended reachable execution.

The supplied representation identifies omitted unreachable regions without supplying their contents. No behavior is specified for those regions.

## Evidence

**Established static evidence:** ordered mixed-width accesses; signed halfword observations; four primary-offset-100 observations; five global-selection stages with separate selection and publication observations; alias-sensitive rereads; two saved-word publication groups; and the fresh fifth-stage offset-20 destination read after the loaded-base offset-68 store and before its offset-84 self-publication.

Also established are repeated arithmetic publications, product-derived traversal, partial stack-relative assignments, two ordinary dependency calls, and separate terminal target production and invocation.

**Inferred intent:** recurring offsets may describe structured mutable state; the exposed stack-relative area may serve as an argument record. Neither interpretation is required by the contract.

**Unknown:** application purpose, complete ABI, valid invocation domain, dependency semantics, actual reachability, and concrete terminal outcome.

No runtime tests have been performed. No invocation, memory trace, return, fault, or terminal transfer has been observed.
