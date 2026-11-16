<!-- Trusted provenance: F-9a4fcf4dfd54; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9a4fcf4dfd54

## Purpose

F-9a4fcf4dfd54 invokes an ordered sequence of unresolved dependencies, transforms their result bit patterns, modifies storage through calculated addresses, and reaches a final unresolved indirect transfer.

Its application-level purpose is not established. The literal values, repeated offsets, and self-referential stores do not establish text generation, object initialization, collection management, or a particular protocol.

This contract distinguishes three evidence levels:

- **Established:** behavior explicitly represented in the supplied static material.
- **[INFERENCE]:** interpretation requiring an additional machine model or semantic assumption.
- **documented_only:** behavior left unresolved rather than assigned an unsupported rule.

The ordering described below is the ordering in the supplied representation. It is not an independent guarantee about processor-level ordering, atomicity, or compiler treatment of ordinary memory.

## Interface

### Inputs and outputs

The represented interface has no explicit parameters and no explicit return value. It exposes no status code, output-buffer argument, length, or error object.

Its effective inputs nevertheless include:

- Incoming control context determining the first indirect invocation target.
- Primary and secondary result channels from dependencies.
- Existing bytes and words in absolute and indirectly addressed storage.
- A stored 32-bit value subsequently used as an address base.
- Hidden state read by unresolved dependencies.

The visible outputs are dependency invocations, memory-access effects, and four arguments supplied to the final transfer. A return is represented after that transfer; successful return is not established.

### Widths and address interpretation

| Access or value | Represented width | Required distinction |
|---|---:|---|
| Byte | 8 bits | A byte store retains the byte-sized portion of its value. Explicit unsigned byte reads produce values from 0 through 255. |
| Halfword | 16 bits | Not interchangeable with a byte or word access. |
| Word | 32 bits | Used for stored values, address calculations, and retained results. |
| Combined result | 64 bits | May be separated into low and high 32-bit portions or shifted before conversion. |

Offsets in this contract are byte offsets. An “address base” is an integer-derived address quantity, not evidence of a valid object or pointer-typed field.

Byte order, alignment, address-space mapping, and access atomicity are unresolved. Consequently, the byte representation of a stored word cannot be specified unconditionally.

Many calculations have signed-looking 32-bit operands. Portable signed-overflow behavior is not established. **[INFERENCE]** Under a 32-bit bit-vector model, multiplication and left shifting retain the low 32 bits, with address arithmetic interpreted consistently with that model. Numerical examples relying on truncation are conditional on this interpretation.

### Preconditions and ownership

Progression requires every reached indirect target to be executable under the actual calling convention and every reached memory access to be valid at its represented width.

No visible validation establishes bounds, nullability, alignment, object lifetime, or permissions. A zero-derived base does not select a graceful rejection path.

No explicit allocation, release, or ownership transfer is identifiable in the visible body. Dependencies may allocate, release, mutate globals, or invalidate storage; their behavior is unknown. Normal execution therefore requires suitable dependency and memory-lifetime conditions, not an assumption that dependencies preserve them.

## Behavior

### Execution structure and dependency results

The body represents a fixed sequence of indirect invocations followed by direct memory mutation and a final indirect transfer. No explicit conditional branch, input-controlled loop, retry, or validation path appears.

Each subsequent region is reached only if preceding calls and accesses permit execution to continue. This does not constrain branching or termination inside dependencies.

The first invocation target depends on incoming control context plus displacement 0xffe. Later pre-mutation targets use embedded control references with the same displacement. The actual callee identities and signatures remain unresolved.

Calls expose between zero and four arguments. Recurring argument relationships include:

- A primary result shifted left by 8.
- Low and high 32-bit portions of a combined result passed separately.
- A high result portion passed unchanged or after shifting or masking.
- One transformed value occupying multiple argument positions.
- Earlier results retained across intervening invocations.
- Explicit zero arguments alongside unresolved secondary-channel values.

A call with no visible arguments may still consume hidden inputs and have side effects. Ignoring a result does not make the invocation removable. Similar argument shapes do not establish identical callees.

The represented 64-bit result may be a scalar result or a representation of multiple machine result channels; the actual convention is **documented_only**.

### Arithmetic and field boundaries

The represented masks include fields of 24, 22, 21, 20, 19, 18, 17, 16, 15, 14, 13, 12, and 11 bits, together with individual-bit selection.

Their placement relative to shifts matters:

- Selecting 24 bits and then shifting left by 6 produces at most a 30-bit unsigned result.
- Selecting 22 bits before shifting left by 7 or 8 produces at most 29 or 30 bits.
- Selecting 20 bits, logically shifting right by 10, and then shifting left by 5 or 6 extracts a 10-bit field before repositioning it.
- Logical right shifts of unsigned 32-bit values insert zeros.
- Right shifts of complete 64-bit results by 34, 36, or 44 extract information from the upper portion before conversion.
- An intervening 32-bit left shift can discard bits permanently. A later right shift is not generally its inverse.

Some retained quantities are explicitly zeroed before later shifts. Their derived values remain zero in the represented dataflow where those operations are defined. This does not imply that dependency result channels or memory remain zero.

### Initial direct-memory effects

The transition from calls to memory mutation has a significant access order.

First, a word is written at offset 0x14 from the last combined result’s high-word base. A byte is then written at offset 9 from a retained shifted-result base. A word store follows at offset 0x54 from an address formed by multiplying the last result’s low word by 256. The offset-9 byte is then overwritten.

Next, a word at offset 0x14 from another scaled base is cleared, and an unsigned byte is read from that base plus 5. A word at offset 0x14 from a small derived base is cleared before the retained byte is widened and stored as a word at offset 0x34 from the scaled base.

The scaled base’s offset-0x14 word is then read. The small derived base’s offset-0x14 word is cleared again, followed by two consecutive byte stores at its offset 0x15.

The small derived base is twice an extracted quantity. **[INFERENCE]** Under 32-bit bit-vector arithmetic, the extracted quantity is a multiple of four from 0 through 508, making the base a multiple of eight from 0 through 1016. This range does not establish allocation or validity.

These operations cannot generally be reduced to final memory contents. Aliasing can make earlier writes affect later reads, and byte accesses can overlap word accesses.

### Address-derived and link-like updates

The function next reads an unsigned byte at absolute address 5 and uses its value as an address base. It also writes a byte to absolute address 1.

The following region uses byte offsets including 1, 5, 9, 0x0d, 0x11, and 0x15, and word offsets including 4, 0x14, 0x24, 0x34, 0x44, 0x54, 0x64, and 0x74. These offsets describe accesses, not a recovered record type.

Established effects include:

- A byte written at offset 0x11 and subsequently cleared.
- An offset-0x54 word read before replacement.
- Offset-0x74 words used as bases for further stores.
- Self-valued word stores at offsets 0x24 or 0x14.
- Repeated offset-9 byte stores and offset-0x24 word stores.
- A doubled value written as a word to absolute address 0x54.
- A byte written to absolute address 0x15.
- Further address and value calculations using factors of 2, 4, and 8.

Some unsigned byte reads supply bases for later offset-0x74 word reads. No pointer validation intervenes. A nested update also represents multiple uses of an offset-0x74 memory read; it must not be presumed to use one cached read without an additional memory-semantics justification.

**[INFERENCE]** The self-valued fields and paired updates resemble link manipulation. List membership, linkage invariants, reference counts, and ownership consequences remain unresolved.

### Literal stores and repeated publication

A later region stores 0x3a through three distinct accesses: a byte at a fourfold-scaled address, a halfword at an eightfold-scaled address, and a word at offset 0x20 from the first address. Nothing establishes a string operation.

For the remainder of this section, the **literal base** means the address whose offset-0x20 word receives the following ordered values:

1. 0x20.
2. 0x2f.
3. The zero-extended byte read from the literal base plus 4.
4. 0x20 again.

The byte read occurs after the first two word writes and before its corresponding word copy. Aliasing and access-observable memory may make these intermediate operations significant.

**After the final offset-0x20 store, the function reads the stored address base used for publication. Only after that read does it write the word 0x20 to absolute address 0x60.** The stored-base read must not be moved after that absolute-address write.

The resulting **publication base** is retained for a long sequence of repeated groups. Each group has this access order:

- A word store at offset 0x54 from a scaled base.
- A byte store at publication-base offset 0x19.
- A byte store at literal-base offset 0x11.
- A word store at publication-base offset 0x34.

The first group uses a different retained scaling input from the subsequent groups. The two byte values are retained byte portions of the literal and publication bases respectively. The offset-0x54 stored value is a retained fourfold-scaled link value.

Repetition is fixed in the representation, not input-controlled. The exact group count is **documented_only** in this contract; no exact-count acceptance claim is made. Repeated stores remain distinct represented effects, even when their addresses and values repeat.

### Final mutation: opening order

For this section, the **doubled-link base** is twice the loaded link whose fourfold value was retained for publication.

The final region begins in this order:

1. A 32-bit word is read at absolute address 0xa9.
2. A word is read at offset 0x74 from the retained fourfold-link base.
3. A byte is written to absolute address 0x3a, derived from eight times the adjusted scaling input plus 0x30.
4. A word is read at doubled-link-base offset 0x14.
5. A word is read at literal-base offset 4.

The absolute-address byte write precedes both latter reads. Their values must not be specified as though sampled before that write.

The offset-0x14 word is retained for later restoration and for the final transfer’s third argument. The literal-base offset-4 word is retained for a later publication-base offset-0x74 store.

Subsequent byte writes place retained byte portions at doubled-link-base offset 0x0d and literal-base offsets 0x0d and 9, and place 0x6c at literal-base offset 8. A word store temporarily places the doubled-link base into its own offset-0x54 location.

### Absolute address 0x67: three separate reads

The low byte of the previously loaded absolute-address-0xa9 word is written to absolute address 0x67.

**The next relevant read is the word at doubled-link-base offset 0x44. Only afterward is address 0x67 read for the byte write to absolute address 0x21.**

The loaded offset-0x44 word then participates in a literal-base offset-0x10 byte store, a publication-base offset-0x54 word store, and a shifted byte store at doubled-link-base offset 1.

That shift uses the low eight bits of the literal base as its count. A count in 0 through 255 is not necessarily valid for a 32-bit shift. Signed right-shift behavior and counts of 32 or greater are **documented_only**.

Further operations replace the publication-base offset-0x54 word with the literal base, read the literal-base offset-0x40 word, publish the retained offset-4 word at publication-base offset 0x74, copy the offset-0x40 word to literal-base offset 0x10, and store the doubled-link base at publication-base offset 0x44.

Only then does a **second, separate byte read of address 0x67** supply a byte store at literal-base offset 9.

After that read and store, the function:

- Writes a retained byte at doubled-link-base offset 9.
- Writes the retained absolute-address-0xa9 word at literal-base offset 0x44.
- Reads the word at literal-base offset 0x54.

A **third, separate byte read of address 0x67** then supplies another byte store at literal-base offset 9.

Neither later store is specified to reuse the byte originally written to 0x67 or the byte previously read for address 0x21. Intervening memory operations may matter through aliasing or access semantics.

### Remaining updates and final transfer

The word loaded from literal-base offset 0x54 becomes another address base. A retained offset-0x74 byte portion is written through that base at offset 0x0d. The saved doubled-link-base offset-0x14 word is restored to doubled-link-base offset 0x54.

A fresh word read at literal-base offset 0x14 supplies a later byte store. Further updates write the retained offset-0x74 byte portion at doubled-link-base offset 0x11 and write the newer base’s byte and word portions at literal-base offsets 9 and 0x44.

The function then reads a word at doubled-link-base offset 4. Through that newly loaded base, it performs a halfword store of 0x65 at twice the base and reads words at offsets 0x64 and 0x74. Between those reads, it writes the retained literal-base offset-0x14 byte portion at the newer base’s offset 0x11 and writes decimal 100 as a word at its offset 0x54.

Finally, it reads doubled-link-base offset 0x64, performs a byte store at doubled-link-base offset 9, writes through the latest loaded base at offset 0x54, and writes a word at twice the earlier offset-0x64 result.

The final unresolved transfer receives:

| Argument | Represented value |
|---|---|
| First | 0x72 |
| Second | The word loaded at offset 0x64 through the base obtained from doubled-link-base offset 4 |
| Third | The saved doubled-link-base offset-0x14 word |
| Fourth | The word loaded at offset 0x74 through that same newly obtained base |

A return follows in the representation. Whether the transfer returns normally, dispatches elsewhere, or terminates execution is unresolved.

## Side effects

The observable surface includes indirect invocations, dependency effects, direct memory reads and writes, absolute-address accesses, and the final transfer.

No rollback or transaction boundary is represented. A fault or nonreturning dependency may leave earlier effects in place.

No synchronization, locking, barriers, or atomicity guarantees are established. Concurrent access rules and device-memory behavior remain **documented_only**.

A retained value and a fresh read are different inputs to later behavior. In particular:

- The publication base is sampled before the absolute-address-0x60 store.
- The two late offset-0x14 and offset-4 words are sampled after the absolute-address-0x3a write.
- Address 0x67 supplies three distinct late byte reads.
- The final third argument is a saved word, not a fresh read of its original location.

There is no visible error-reporting mechanism. Invalid targets, inaccessible storage, invalid shifts, or incompatible calling conventions do not have specified graceful recovery behavior.

## Dependencies

Only the subject function has a resolved identity in the supplied dependency map. The following IDs are contract-local opaque references:

| ID | Meaning |
|---|---|
| F-0001 | First indirect invocation, selected from incoming control context |
| F-0002 | Ordered family of remaining invocation sites before direct memory mutation |
| F-0003 | Final unresolved four-argument transfer |

F-0002 does not assert a shared target, signature, result convention, or behavior across its sites.

For all dependencies, global preservation, allocation, release, ownership, exceptions, faults, reentrancy, and normal return are **documented_only**. A dependency cannot be treated as pure because it has no visible arguments or its result is ignored.

## Acceptance examples

These are conditional static criteria, not executed tests.

1. **32-bit truncation.** Under the bit-vector interpretation, shifting 0x12345678 left by 8 produces 0x34567800. Preserving the discarded byte in a wider argument changes the represented relationship.

2. **Mask before shift.** Selecting the low 24 bits of 0xffffffff and shifting left by 6 produces 0x3fffffc0. Shifting the unmasked word is not equivalent.

3. **Unsigned widening.** If an explicit unsigned byte read returns 0xfe, its corresponding widened word copy stores 254, not a sign-extended negative value.

4. **Overlapping clear and read.** In stable ordinary memory, an offset-0x14 clear followed by a read of the identical word yields zero if no intervening access changes overlapping storage. Apparent base similarity alone does not establish that condition.

5. **Publication-base sampling.** An access-observation trace must place the stored publication-base read after the final literal-base offset-0x20 store and before the word store to absolute address 0x60.

6. **Late opening order.** The trace must show the absolute-address-0xa9 word read, retained-base offset-0x74 word read, absolute-address-0x3a byte write, doubled-link-base offset-0x14 word read, and literal-base offset-4 word read in that order.

7. **Independent 0x67 observations.** If valid access-observable storage supplies 0x11, 0x22, and 0x33 on the three late reads of address 0x67, the corresponding writes receive those values at address 0x21 and the two literal-base offset-9 destinations respectively. The offset-0x44 word read precedes the first read; the later reads retain their intervening memory operations.

8. **Intermediate writes remain effects.** Access-observable storage may observe the offset-0x20 writes of 0x20 and 0x2f even though the sequence ends with 0x20.

9. **No invented null rejection.** A zero base does not require an early return or status result. Progression instead depends on whether the resulting low-address accesses are valid.

10. **Final retained argument.** If F-0003 is reached, its first argument is 0x72 and its third is the saved offset-0x14 word, even if the original storage has since changed.

## Uncertainties

The following remain **documented_only**:

- Application purpose, object types, and intended invariants.
- Actual calling convention and hidden inputs.
- Dependency identities, signatures, and effects.
- Signed overflow, signed shifts, oversized shift counts, and address wrapping.
- Byte order, alignment, memory mapping, and atomicity.
- Validity of low absolute addresses.
- Aliasing, object extent, lifetime, and concurrent-access rules.
- Allocation, ownership, and release behavior.
- Exact repeated-group count in this contract.
- Normal return from the final transfer.
- Fidelity of the supplied representation to executable machine behavior.

These gaps do not authorize invented validation, text semantics, ownership rules, or dependency-preservation guarantees.

## Evidence

The supplied static body establishes the represented interface, indirect invocation structure, result transformations, access widths, calculated addresses, repeated stores, absolute-address operations, and final argument relationships.

The three ordering-sensitive regions are directly supported: publication-base sampling precedes the absolute-address-0x60 write; the absolute-address-0x3a write precedes the two following word reads; and the late address-0x67 sequence contains three separate reads, with the offset-0x44 read before the first and intervening memory effects before the later two.

Comments, earlier prose, and reviewer feedback are not independent behavioral evidence. No implementation inspection beyond the embedded material, dependency inspection, runtime execution, or runtime tests have been performed.
