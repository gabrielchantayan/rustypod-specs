<!-- Trusted provenance: F-b6ebdf35c9e3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b6ebdf35c9e3

## Purpose

F-b6ebdf35c9e3 performs an ordered sequence of unresolved indirect invocations, transforms their results, and reads and writes memory through fixed and computed addresses. Its last represented external interaction is an unresolved invocation with four arguments.

No application-level purpose is established. The representation does not justify identifying the operation as initialization, parsing, rendering, serialization, or object construction. Constants and address offsets are behavioral evidence, not proof of a record format or application meaning.

This contract distinguishes:

- **Established static evidence:** interactions, widths, transformations, and ordering represented in the supplied body.
- **Inferred intent:** descriptive interpretations that impose no additional acceptance requirements.
- **documented_only:** unresolved behavior for which no semantics are invented.

Descriptions of later execution are conditional on preceding dependencies returning and preceding accesses completing. An interaction that has occurred is not a guarantee that the affected state subsequently remains unchanged. No runtime tests have been performed.

## Interface

### Inputs and outputs

The represented interface has no explicit parameters and no explicit return value.

Implicit inputs include:

- Incoming control-related machine state used to form the first indirect target.
- Results of indirect invocations, including combined 64-bit results.
- Auxiliary machine-result state whose producers and calling convention are not fully established.
- Bytes and 32-bit words read from fixed and computed addresses.
- A separately stored 32-bit base value.
- The unresolved terminal invocation target.

Observable outputs are dependency interactions, memory accesses, and the terminal invocation. No success indicator, error code, owned result, output length, or caller-visible return value is established.

### Widths and arithmetic

| Quantity | Represented width | Contract interpretation |
|---|---:|---|
| Byte access | 8 bits | Stores retain the low eight bits |
| Halfword store | 16 bits | Stores retain the low sixteen bits |
| Word access | 32 bits | Transfers the represented 32-bit pattern |
| Combined dependency result | 64 bits | Numerical low and high 32-bit portions are consumed separately |
| Address arithmetic | Apparently 32 bits | Target overflow and address-conversion behavior are documented_only |

Explicit widening of an unsigned byte yields a value from 0 through 255. It does not sign-extend values with bit 7 set.

Numerically splitting a 64-bit result does not establish memory byte order. Explicit unsigned right shifts are zero-filling. Signed shifts and arithmetic require separate qualification.

Where this contract uses **conditional 32-bit machine semantics**, a multiplication or left shift retains the low 32 bits before a subsequent transformation. This is a conditional interpretation of represented bit patterns, not a determination of all signed-expression semantics. Signed overflow, invalid address conversion, and out-of-range shift behavior remain documented_only.

### Preconditions

Reaching normal completion requires executable indirect targets, compatible calling conventions, returning dependencies, and accessible storage at every reached address and width. The environment must support the required alignment or unaligned accesses.

These requirements are not visible validations. In particular, the body does not exclude low addresses or establish that dependencies leave later-used storage accessible.

## Behavior

### Dependency interactions and retained values

A long straight-line sequence of indirect invocations precedes the explicit memory phase. The first target incorporates incoming control-related state and a displacement of 4094 bytes. Later target expressions use fixed continuation-like values or code-associated references with the same displacement.

Similar expressions do not establish identical callees. An occurrence whose result is unused remains an interaction; an occurrence without represented arguments may still consume implicit state and affect memory or execution context.

Represented invocations have zero through four arguments. Arguments include previous results shifted left by eight bits, low and high portions of combined results, auxiliary machine-result state, masked or additionally shifted values, and explicit zeroes. Some retained values survive many intervening invocations in the represented dataflow.

Width-sensitive transformations include:

- Retaining the low 24 bits and narrower fields.
- Extracting bits 10–19 before further scaling.
- Unsigned 64-bit right shifts by 34, 36, or 44 places before conversion to 32 bits.
- Successive shifts of already width-limited intermediates.

Under conditional 32-bit machine semantics, two successive 28-bit left shifts of a 32-bit intermediate produce zero. They do not preserve a 56-bit intermediate. Quantities explicitly set to zero and subsequently shifted without reassignment remain zero in the represented dataflow. Neither observation establishes that dependencies preserve globals or memory.

No conditional rejection, retry, or explicit recovery is represented.

### Bases entering the memory phase

Near the end of the dependency sequence, retained results form:

- A base obtained by multiplying a result by \(2^{25}\).
- A separate base obtained by shifting a result left by 25 bits.
- A retained value formed by an eight-bit left shift.
- A small quantity obtained by shifting a result left by eight bits, retaining the low fifteen bits, and shifting right by six.
- The low and high 32-bit portions of the final combined result.

Under conditional 32-bit machine semantics, the small quantity is a multiple of four from 0 through 508. Twice that quantity is a multiple of eight from 0 through 1016. These restrictions establish neither allocation nor address validity.

### Initial memory publications

The memory phase first publishes the retained 25-bit-shifted value as a word at offset 0x14 from the final result’s high-word-derived base. It next writes the high word’s low byte at offset 9 from the 25-bit-shifted base, publishes the \(2^{25}\)-derived base as a word at offset 0x54 from 256 times the final result’s low word, and overwrites the offset-9 byte with the retained value’s low byte.

The \(2^{25}\)-derived base receives a zero word at offset 0x14, followed by a byte read at offset 5. A zero word is then written at offset 0x14 from twice the small quantity. The observed byte is zero-extended and published as a word at offset 0x34 from the \(2^{25}\)-derived base, after which its offset-0x14 word is read.

The read-back is not unconditionally zero. Intervening computed stores can overlap it, and access-dependent memory behavior is not excluded.

The doubled-small base receives another clear and two separate byte stores at offset 0x15. A byte read at absolute address 0x05 precedes a byte write at absolute address 0x01. The read byte, widened unsigned, supplies another computed base.

Further publications include:

- A byte at offset 0x15 from that byte-derived base.
- A retained byte at offset 0x11 from the \(2^{25}\)-derived base, later overwritten with zero.
- A retained word at offset 0x54 from the earlier read-back value.
- A read of the byte-derived base’s offset-0x54 word before that location is overwritten.
- A word at offset 0x34 from the doubled-small base.

Additional reads at offset 0x74 supply later destinations. One publication uses two represented reads of the same offset-0x74 location: one supplies a destination base and the other supplies a stored word. Equality of their observations is not established; their relative evaluation order within that expression is not resolved here.

Repeated offset-0x24 word publications, offset-9 byte publications, and offset-0x14 word publications remain separate events. This phase also writes a scaled word to absolute address 0x54 and a truncated byte to absolute address 0x15.

### Linked and scaled memory effects

A previously loaded word is scaled by four and eight. Another offset-0x74 word read is scaled by two. These values participate both as stored data and as addresses.

A nested access reads an unsigned byte at offset 5 from a four-times-scaled base, then reads a word at offset 0x74 from the byte’s zero-extended value. Later accesses again read an offset-5 byte before another byte-derived offset-0x74 word read. These observations are not interchangeable.

This phase publishes full words and truncated cross-reference bytes through related bases. It also includes a byte widened to a word at offset 0x44. A byte publication of an address-derived value does not publish a complete 32-bit pointer.

Computed reads at offsets 0x04, 0x14, 0x44, 0x54, 0x64, and 0x74 contribute to later addressing. No independence between these memory regions is established. Each read observes memory at its represented position, not a snapshot taken at entry.

### Constants and repeated publications

A later group publishes 0x3A in three widths: a byte at a four-times-scaled address, a halfword at an eight-times-scaled address, and a word at offset 0x20 from the four-times-scaled address.

A different scalar is adjusted by subtracting 0x96. Four times that adjusted scalar supplies a publication base. At its offset 0x20, the body makes four word publications: 0x20, 0x2F, a zero-extended byte observed at its offset 4, and 0x20 again. The byte read follows the first two writes and precedes the third. It is not necessarily an observation of entry memory.

The body then reads the separately stored base and writes the word 0x20 at absolute address 0x60.

A long fixed repeated-store region follows. Each recurring group contains, in order:

- A scaled-target word publication at offset 0x54 from an eight-times-scaled base.
- A byte publication of the publication base’s low byte at stored-base offset 0x19.
- A byte publication of the stored base’s low byte at publication-base offset 0x11.
- A word publication of the eight-times-scaled base at stored-base offset 0x34.

The initial group uses the original, unadjusted scalar for its eight-times-scaled base; later groups use the adjusted scalar. The region has no represented conditional termination. Its exact multiplicity remains **documented_only** here. It is neither a variable-length iteration nor one idempotent update.

### Tail reads, overwrites, and terminal invocation

The following descriptive roles identify retained values without assigning an object schema:

| Role | Meaning |
|---|---|
| Publication base | Four times the adjusted scalar |
| Stored base | The separately loaded 32-bit base |
| Linked base | Twice the later byte-derived offset-0x74 word |
| Scaled target base | The four-times-scaled value published by the recurring offset-0x54 stores |
| Adjusted scalar | The scalar after subtraction of 0x96 |

The tail begins with a **word read at absolute address 0xA9**, followed by a **word read at scaled-target-base offset 0x74**. It then writes a byte at **absolute address 0x3A**, containing the low eight bits of eight times the adjusted scalar plus 0x30.

Only after that byte publication does it read the word at linked-base offset 0x14 and then the word at publication-base offset 0x04. The linked-base word is saved for a later publication and the terminal invocation. These reads must not be moved before the 0x3A write in the behavioral trace.

Subsequent publications place:

- The publication base’s low byte at linked-base offset 0x0D.
- 0x6C at publication-base offset 8.
- The stored base’s low byte at publication-base offsets 0x0D and 9.
- The linked base as a word at its own offset 0x54.

The retained 0xA9 word’s low byte is then published at **absolute address 0x67**. A word read at linked-base offset 0x44 follows. Only afterward is a **fresh byte read from 0x67** used for the byte publication at **absolute address 0x21**. The 0x21 publication does not directly reuse the retained 0xA9 value.

The offset-0x44 word supplies a byte publication at publication-base offset 0x10 and a word publication at stored-base offset 0x54. It also supplies the signed operand of a right shift whose count is the publication base’s low eight bits. The shifted result’s low byte is written at linked-base offset 1. Exact target semantics for that signed shift remain documented_only. Stored-base offset 0x54 is then overwritten with the publication base.

A word read at publication-base offset 0x40 supplies a later word publication at its offset 0x10. Between them, the saved publication-base offset-0x04 word is published at stored-base offset 0x74. The linked base is subsequently published at stored-base offset 0x44.

Two additional, separate byte reads of absolute address 0x67 each supply a publication at publication-base offset 9. Between those reads, the body publishes the publication base’s low byte at linked-base offset 9, publishes the retained 0xA9 word at publication-base offset 0x44, and reads a word at publication-base offset 0x54. Those intervening accesses can matter through aliasing or observable memory behavior. All three tail reads of 0x67 remain distinct.

The word observed at publication-base offset 0x54 becomes another destination base. Its offset 0x0D receives the low byte of the saved scaled-target offset-0x74 word. The saved linked-base offset-0x14 word is then published at linked-base offset 0x54. This is a publication of the saved pattern, not a guarantee that every intervening change to surrounding state is reversed.

A fresh word read at publication-base offset 0x14 supplies a later byte publication. Before that publication, the body writes the saved scaled-target byte at linked-base offset 0x11, overwrites publication-base offset 9 with the new destination base’s low byte, and publishes that destination base as a word at publication-base offset 0x44.

The remaining chain obtains another base from linked-base offset 0x04. It publishes halfword 0x65 at twice that base and then reads its offset-0x64 word. The other destination receives the saved publication-base offset-0x14 byte at offset 0x11 and decimal 100 as a word at offset 0x54.

Further word reads observe the newly obtained base’s offset 0x74 and the linked base’s offset 0x64. Final publications include:

- The newly obtained base’s low byte at linked-base offset 9.
- The newly obtained base’s offset-0x64 word at offset 0x54 from the destination obtained through linked-base offset 0x64.
- That latter destination as a word at twice the newly obtained base’s offset-0x64 value.

The terminal unresolved invocation receives, in order:

1. 0x72.
2. The word read at offset 0x64 from the base obtained through linked-base offset 0x04.
3. The linked-base offset-0x14 word saved immediately after the 0x3A publication.
4. The later word read at offset 0x74 from the newly obtained base.

A return is represented afterward. Reaching that return requires the terminal dependency to return.

## Side effects

### Layout and aliasing

Access widths and offsets are established; a coherent record layout is not.

| Width | Observed computed-base offsets or placement |
|---|---|
| 8 bits | 0x01, 0x05, 0x08, 0x09, 0x0D, 0x10, 0x11, 0x15, 0x19 |
| 16 bits | Directly scaled addresses |
| 32 bits | 0x04, 0x10, 0x14, 0x20, 0x24, 0x34, 0x40, 0x44, 0x54, 0x64, 0x74 |

A byte at offset 0x11 can overlap a word beginning at offset 0x10. Numerical consequences depend on byte order.

Fixed-address effects include byte reads at 0x05 and 0x67, a word read at 0xA9, byte writes at 0x01, 0x15, 0x21, 0x3A, and 0x67, and word writes at 0x54 and 0x60. Computed accesses may overlap these addresses. Literal low-address accesses are not null checks.

### Ownership, errors, and concurrency

No explicit allocation, release, ownership transfer, or cleanup protocol is established. Unresolved dependencies may allocate, retain, overwrite, reverse, release, or invalidate previously affected state.

No bounds checks, address validation, rollback handler, synchronization, or error translation are represented. If a later dependency fails, transfers control elsewhere, or never returns, earlier interactions have already occurred and no local rollback by F-b6ebdf35c9e3 is represented. **Their resulting memory state is not guaranteed to persist:** the unresolved dependency may change or invalidate it before failure or during indefinite execution.

Fault behavior and dependency failure behavior are documented_only. Atomicity, reentrancy, concurrent observation, and device-memory semantics are likewise unresolved.

## Dependencies

Only F-b6ebdf35c9e3 has an established mapped identity.

| Opaque reference | Role | Established boundary |
|---|---|---|
| F-unknown-001 | Category for unresolved indirect invocation occurrences | Represented arguments and result consumption |
| F-unknown-002 | Terminal unresolved invocation | Four ordered arguments beginning with 0x72 |

F-unknown-001 is not an assertion that its occurrences reach one function. These references imply no preservation of globals, registers, memory, ownership, storage lifetime, or execution context.

## Acceptance examples

These are conditional static criteria, not runtime results.

1. **Unsigned widening.** If an unsigned byte read observes 0xE7, its word publication contains 0x000000E7 rather than 0xFFFFFFE7.

2. **Byte truncation.** A byte publication of 0x1234ABCD writes 0xCD only.

3. **Small-base boundary.** Under conditional 32-bit machine semantics, the small masked quantity is a multiple of four no greater than 508; 511 is not possible.

4. **Cumulative shifts.** Under those semantics, successive 28-bit left shifts of a 32-bit intermediate yield zero.

5. **Distinct widths.** The byte, halfword, and word publications of 0x3A are distinct access events. Three byte writes do not reproduce that trace.

6. **Ordered overwrite.** With ordinary non-aliased memory and no other changes, the four-write offset-0x20 group leaves 0x20. Its trace nevertheless includes all four writes and the intervening byte read.

7. **Tail interleaving.** If linked-base offset 0x14 overlaps 0x3A, its read follows the 0x3A publication. Moving the read earlier is inconsistent with the represented order.

8. **Fresh observation for 0x21.** If the intervening offset-0x44 read has access-dependent behavior that changes byte 0x67, the subsequent 0x21 publication uses the freshly observed byte.

9. **Separate offset-9 publications.** If an intervening computed store aliases 0x67, the two later reads may observe different bytes. Replacing them with one cached observation is not equivalent.

10. **Failure or non-return.** If a reached dependency never returns, subsequent body accesses and the terminal invocation need not occur. Earlier interactions have occurred, but their resulting state need not remain unchanged. A dependency that overwrites or invalidates that state before continuing indefinitely is not excluded; no local rollback is represented.

11. **Terminal boundary.** Reaching F-unknown-002 requires the four specified arguments. A return from that dependency does not establish preservation of the preceding publications or a caller-visible result.

## Uncertainties

The following remain **documented_only**:

- Application-level purpose and accuracy of the represented function boundary.
- Calling convention and implicit entry requirements.
- Dependency identities, auxiliary-result producers, and dependency side effects.
- Byte order, alignment behavior, address-space model, and signed arithmetic semantics.
- Runtime-count signed-shift behavior, especially counts outside 0–31.
- Relative evaluation order of reads within the repeated-read publication expression.
- Meaning and validity of low-address accesses.
- Object boundaries, alias relationships, storage lifetimes, and access-dependent memory behavior.
- Exact multiplicity of the long repeated-store region.
- Failure mechanisms and state changes made by dependencies before failure or non-return.
- Whether the terminal interaction is a conventional call or another control-transfer form.

“Linked,” “stored,” and “publication” are descriptive roles only. Any data-structure or ownership interpretation is inferred intent, not an established requirement.

## Evidence

This contract is based on static reading of the supplied representation, not on embedded comments, prior prose, or reviewer assertions as behavioral authority.

The representation establishes ordered indirect interactions, explicit access widths, repeated publications, and the tail’s interleaved reads and writes. It contains no explicit local rollback path. Because the indirect dependencies remain unresolved, it does not establish that previously affected state survives their later execution, failure, or non-return.

No tools, independent implementation inspection, runtime execution, or tests were used. Acceptance examples describe conditional static requirements, not observed outcomes.
