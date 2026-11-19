<!-- Trusted provenance: F-3f142cfaad4d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3f142cfaad4d

## Purpose

F-3f142cfaad4d performs an ordered sequence of external invocations and direct memory effects. It writes individual bytes to computed addresses, reads two 16-bit quantities, assigns a retained byte to a global destination, writes a 32-bit value through a retained pointer, and reaches an indirect terminal continuation.

**Established static evidence:** The supplied representation shows no formal parameters, ordinary return value, top-level conditional branch, or top-level loop. Incoming machine state, dependency outputs, and memory contents influence later effects. No displayed status result selects a recovery path.

**[INFERENCE] Inferred intent:** The operation coordinates externally supplied state and memory effects. An application-level purpose—such as initialization, reset, allocation, or device configuration—is not established.

This contract describes the represented path, not a guarantee that every invocation or access completes. Each later requirement is conditional on execution reaching that point.

## Interface

### Inputs and observable outputs

Although no explicit parameters are represented, the operation consumes:

- Incoming control-link state used to select the initial dependency destination.
- An incoming eight-bit quantity reused in early byte writes.
- An incoming 32-bit quantity used in a four-argument invocation.
- Primary dependency results represented predominantly as 32-bit words.
- Four dependency results represented as 64-bit quantities.
- Secondary dependency outputs, including a pointer, a separately retained scalar, and words consumed by later invocations.
- Memory contents at two computed 16-bit read locations.

Observable outputs comprise dependency invocations and their displayed arguments, direct memory accesses, a global assignment, and terminal control transfer. No ordinary success result or error result is represented.

An invocation with no displayed arguments does not establish an interface free of implicit inputs.

### Widths and layout

| Item | Established representation |
|---|---|
| Computed-address byte writes | Eight bits per write |
| Explicit memory reads | Two accesses of sixteen bits each |
| Publication through the retained pointer | One thirty-two-bit write |
| Ordinary scalar operands | Predominantly thirty-two bits, with signed and unsigned interpretations distinguished |
| Wide dependency results | Sixty-four bits, separated numerically into low and high thirty-two-bit halves |
| Retained and extracted byte contents | Eight-bit patterns |
| Restricted byte contents | Unsigned values from zero through seven |

Scaling an address does not establish an array element size, object layout, or access width. A destination scaled by eight still receives only one byte where a byte write is represented.

The global assignment has an eight-bit source. The destination declaration is absent; its physical width and layout are **documented_only**.

Memory byte order is **documented_only**. Numeric separation of a wide result into halves does not establish memory byte order.

### Preconditions

Continued execution requires executable dependency destinations and compatible interfaces. Every reached access requires a valid mapping and any alignment required by the actual platform. The retained pointer must remain usable until its later 32-bit write.

No local validation establishes these conditions. A zero or negative producer value does not trigger a represented rejection. A computed zero address is not a sentinel permitting an access to be omitted.

Arithmetic relationships below apply where the represented operations have a defined interpretation. They do not silently impose unsigned wraparound on signed expressions.

## Behavior

### Sequence and dependency boundaries

The direct effects progress through these groups:

1. Incoming-byte writes.
2. Six literal-zero byte writes.
3. A byte write whose contents derive from its destination address.
4. Retained-byte writes, including the first 16-bit read.
5. The global assignment.
6. Further retained-byte writes.
7. An extracted-byte write.
8. Restricted-range byte writes, including pointer-derived accesses and publication.
9. Further byte writes and dependency activity.
10. Terminal continuation.

These groups contain distinct ordered invocations, including invocations whose primary results are unused. Grouping them does not authorize collapsing them into interchangeable calls or moving effects across their boundaries.

Important ordering constraints are:

- Each explicit 16-bit read follows its adjacent byte write.
- Each read completes before the invocation consuming its value begins.
- The global assignment precedes the invocation displaying two zero arguments.
- The publication byte write precedes the pointer-based 32-bit write.
- Both publication writes precede the invocation receiving the publication value three times.
- Publication precedes additional calls and byte writes; it is not completion.

No dependency is assumed to preserve memory, globals, or implicit machine state merely because its primary result is unused.

### Dispatch and ordinary argument relationships

F-U001 is selected from incoming control-link state. Most remaining destinations are derived from distinct fixed control-related anchors and belong to F-U002. F-U003 and F-U004 have exceptional destination derivations; F-U004 includes four-byte alignment of its anchor before displacement.

These derivations do not resolve concrete dependency identities.

A frequent argument relationship is four times a preceding 32-bit primary result. Some receiving invocations produce a word that a later invocation consumes unchanged. Other invocations duplicate a quantity in two positions or repeat it three times.

These relationships establish numerical provenance, not semantic roles such as size, pointer, handle, or status.

### Incoming-byte writes

Early byte writes use the incoming eight-bit quantity. The ordinary destination is eight times the associated producer word. A common subsequent invocation receives two copies of that producer word scaled by 16,384.

Exceptions remain distinct:

- After the first such write, F-U003 receives four times its producer word and returns the first 64-bit result. Its consumer receives the low half, high half, and low half, in that order.
- F-U004 receives four times an earlier result. Its own result determines a byte destination scaled by two; the following paired arguments are each scaled by 4,096.
- One invocation receives four arguments. Positions one, two, and four repeat the store-producing word scaled by 16,384. Position three uses the incoming 32-bit quantity scaled by 1,024. A subsequent invocation consumes a secondary output of this invocation scaled by eight.
- Several paired-argument invocations return a new word subsequently consumed at four times its value.
- The second and third 64-bit results each have a consumer receiving the low half followed by two copies of the high half.
- The final incoming-byte write is followed by two arguments each equal to four times its producer word, not the common factor of 16,384.

One paired quantity from a later incoming-byte stage is retained across substantial intervening activity. This **retained incoming-stage quantity** is already scaled by 16,384. Later uses must preserve that provenance rather than substitute a newer primary result or treat the retained quantity as unscaled.

### Literal-zero and address-derived writes

Six subsequent byte writes store literal zero. Each destination is eight times its own producer word, and each following paired invocation uses that word scaled by 16,384. These writes do not depend on the incoming byte being zero.

A separate later write stores the low eight bits of its destination’s numeric value. That destination is eight times its producer word. The following invocation receives two copies of the producer scaled by 16,384, then the destination’s numeric value as its third argument.

This destination-derived content rule is distinct from both the incoming byte and the retained byte described next.

### Retained-byte provenance and writes

An explicitly unsigned 32-bit producer establishes the **retained byte**: the low eight bits of eight times that producer. Its low three bits are therefore zero.

The first destination is sixteen times the unsigned producer; the following paired arguments use that producer scaled by 32,768. These unsigned 32-bit scaling operations retain only their width-limited results.

Before the first explicit read, subsequent destinations using this retained byte occur in this order:

| Destination basis | Destination scale | Following paired-argument scale |
|---|---:|---:|
| Retained incoming-stage quantity | 65,536 | \(2^{27}\) |
| First newer producer | 8,192 | \(2^{24}\) |
| Second newer producer | 8,192 | \(2^{24}\) |
| Third newer producer | 8,192 | \(2^{24}\) |
| Retained derived basis | 4,096 | \(2^{23}\) |
| Further producer immediately before the first read | 8,192 | Read-consuming interface described below |

The **retained derived basis** originates from multiplying the retained incoming-stage quantity by 32,768. The represented operands are signed 32-bit quantities; overflow semantics remain unresolved. It is not a newer dependency result.

After the first read and additional dependency activity, the global assignment takes the retained byte. The immediately following invocation receives two literal-zero arguments. No intervening dependency preservation guarantee is implied.

Six later byte writes reuse the retained byte at eight times distinct producer words. The first five have paired arguments scaled by 16,384; the sixth has paired arguments equal to four times its producer. A subsequent single-argument invocation uses twice a producer result rather than the frequent factor of four.

### Explicit 16-bit reads

The first read follows a retained-byte write at 8,192 times a producer word. It reads sixteen bits at 4,096 times that same word plus 56 bytes. The consuming invocation receives the 4,096-scaled base first and the read value second.

The second read occurs during the restricted-byte phase. Following its adjacent byte write, it reads sixteen bits at 128 times the retained pointer’s numeric representation plus 32 bytes. Its consumer receives only the read value as a displayed argument.

Neither access is an offset from the unscaled original producer. The second is not a read directly through the publication pointer.

Both reads observe memory at their respective sequence positions. Earlier writes and dependency side effects can affect their values. Signed interpretation and extension into argument positions are **documented_only**.

### Extracted and restricted byte contents

A later byte write uses bits 19 through 26 of the earlier unsigned producer that established the retained byte, with bit zero denoting the least significant bit. Its destination instead depends on another producer word scaled by eight. The following paired arguments are four times that destination-producing word.

The remaining restricted writes derive their contents from the highest byte of the unsigned interpretation of the retained derived basis. Dividing that byte by 32 and discarding the remainder yields the **restricted byte**, from zero through seven.

This range is established by the unsigned byte operation. It does not resolve the earlier signed multiplication that produced the basis.

Most restricted-byte destinations are eight times their associated producer words. Many following invocations receive paired arguments scaled by 16,384, but some display no arguments, one consumes the second read, and publication uses a triple argument.

Three destination rules are exceptional:

- One stage applies a right shift of 32 to a represented 32-bit intermediate previously scaled by four. Its unresolved result determines a byte destination scaled by two and paired arguments scaled by 4,096.
- Another destination is 256 times the retained pointer’s numeric representation. Its following paired arguments use that representation scaled by \(2^{19}\).
- A later destination is the retained secondary scalar scaled by 2,048. Its following paired arguments use that scalar scaled by \(2^{22}\).

After the unresolved-shift stage, the fourth 64-bit result is consumed as two arguments: low half, then high half. Unlike the earlier three consumers, it does not repeat a half.

### Secondary outputs and publication

The retained pointer and retained secondary scalar are distinct values. Neither is replaced by intervening primary results in the represented relationships.

**Seven later invocations consume separate secondary-output words unchanged as single displayed arguments.** Five occur before publication and two after publication. The earlier incoming-byte-stage secondary output scaled by eight is a separate transformed use and is not an eighth unchanged use.

Four other invocations receive the retained scalar in argument position two, alongside four times a newer primary result in position one.

Near the end, one producer determines both a byte destination at eight times its value and a 32-bit publication value equal to four times its value. The restricted byte is written first. The publication value is then written directly through the retained pointer. The next invocation receives three copies of the publication value.

A later invocation receives two copies of a new, unscaled primary result. Still later, another invocation receives four times a newer primary result and the retained numeric publication value.

Publication does not establish synchronization, ownership transfer, or a stable final memory value.

### Terminal behavior

After the remaining dispatches, F-U005 receives zero and a fixed location-related operand. The represented continuation invokes the supplied code destination, F-U006.

No ordinary return is represented. The terminal annotation does not establish whether execution traps, halts, resets, enters a handler, or performs another platform-specific action.

## Side effects

Direct memory effects are individual byte writes, two 16-bit reads, one pointer-based 32-bit write, and the global assignment. No surrounding region is established as cleared, copied, or initialized.

Computed locations are not checked for disjointness. Writes may overlap one another, either read location, the global destination, or the publication location. Ordering remains significant under overlap. In particular, the publication byte can be overwritten by the immediately following 32-bit write before the next dependency begins.

No allocation, release, reference-count operation, ownership transfer, or lifetime extension is established. The retained pointer is an external capability whose continued validity is not locally proved.

Unknown dependencies may modify the global, publication location, or other memory. Direct effects therefore describe sequence-position actions, not a guaranteed final snapshot.

No local retry, rollback, cleanup path, or error return is represented. A failed access or non-returning dependency may leave preceding effects in place and prevent later effects. The resulting exception or platform fault behavior is **documented_only**.

## Dependencies

The following are document-local opaque references, not resolved implementation identities.

| Opaque ID | Observable role |
|---|---|
| F-U001 | Initial dispatch selected using incoming control-link state |
| F-U002 | Family of distinct ordered dispatches derived from fixed control-related anchors |
| F-U003 | Exceptional early dispatch producing the first 64-bit result |
| F-U004 | Exceptional aligned-anchor dispatch whose result controls an early scaled-by-two byte destination |
| F-U005 | Terminal mechanism supplying an indirect continuation destination |
| F-U006 | Continuation invoked through the supplied destination |

F-U002 denotes a family, not one interchangeable dependency. Members differ in displayed argument counts and output uses.

Complete interfaces, implicit channels, memory effects, error behavior, termination properties, ownership rules, and preservation guarantees are **documented_only**.

## Acceptance examples

These are conditional static criteria, not observed executions. Each assumes the relevant stage is reached, mappings are usable, and arithmetic has the stated interpretation.

1. **Incoming byte and paired-factor exception.** With producer word 3 and incoming byte 0xA5, an ordinary stage writes one byte containing 0xA5 at address 24. A common paired invocation receives 49,152 twice. The final incoming-byte stage instead receives 12 twice.

2. **Literal-zero independence.** With the same producer and incoming byte, a literal-zero stage writes zero at address 24 rather than 0xA5.

3. **Distinct provenance despite equal contents.** Unsigned retained-byte origin 0x23 establishes byte 0x18 and first destination 0x230. At the separate destination-derived-content stage, producer 0x23 gives destination 0x118 and contents 0x18. Equal contents do not make the rules equivalent.

4. **Byte extraction boundary.** Unsigned origins 31 and 32 establish retained bytes 248 and zero. This is eight-bit extraction, not saturation.

5. **Wide-result ordering.** For low half 5 and high half 9, the first wide consumer receives 5, 9, 5. The second and third receive 5, 9, 9. The fourth receives only 5, 9.

6. **First read.** Producer word 2 gives adjacent byte destination 16,384 and read address 8,248. The consumer receives base 8,192 and the value read at that sequence position.

7. **Extracted contents.** Unsigned origin 0x00580000 gives extracted byte 0x0B. This does not determine the separate destination producer.

8. **Restricted-byte boundary.** Highest-byte values 31, 32, and 255 yield restricted contents zero, one, and seven. These examples do not resolve the signed derivation of that highest byte.

9. **Pointer-derived access.** Pointer representation 64 gives second-read address 8,224, not 96. Publication nevertheless writes through the retained pointer itself.

10. **Secondary-output distinction.** Conditional on reaching the terminal portion, the represented path contains seven unchanged, single-argument secondary-word uses: five before publication and two afterward. An earlier secondary word of 3 supplies 24 to its scaled-by-eight consumer; that transformed invocation is outside the seven.

11. **Publication ordering and overlap.** Publication producer 5 causes address 40 to receive the restricted byte, then the retained pointer to receive the 32-bit value 20. Only afterward does the next invocation receive 20 three times. Under overlap, the earlier byte is not guaranteed to survive.

12. **Unresolved shift and failure prefix.** No concrete destination is required for the width-sized-shift stage without additional machine evidence. If a reached access fails, there is no established permission to skip it, manufacture an error result, or undo preceding effects.

## Uncertainties

The following remain **documented_only**:

- Fidelity and executable boundaries of the supplied representation.
- Complete calling convention and implicit entry-state provenance.
- Concrete dependency identities and complete secondary-output interfaces.
- Signed multiplication overflow, signed left shifts, and negative-value arithmetic.
- The right shift by 32 of a represented 32-bit intermediate.
- Address conversion, wraparound, alignment requirements, and fault behavior.
- Memory byte order and extension of 16-bit read values into arguments.
- The global destination’s declaration, physical width, and application meaning.
- Pointer ownership, lifetime, and validity across dependency activity.
- Atomicity, concurrency, hardware ordering, and synchronization.
- Actual terminal platform behavior.

Unsigned 32-bit operations and eight-bit extraction have their stated width restrictions. Those restrictions do not establish universal modulo-\(2^{32}\) semantics for signed calculations.

## Evidence

The supplied static representation supports the direct access widths, value provenance, displayed argument relationships, and relative ordering described here. The prior draft and reviewer feedback are not independent behavioral authorities.

The seven unchanged secondary-word uses are independently distinguishable from the earlier scaled-by-eight use, the retained pointer, and the separately retained scalar. Five unchanged uses precede publication; two follow it. The transformed early use does not increase that count.

Other established features include four wide-result consumers with the stated argument patterns, six literal-zero byte writes, two explicit 16-bit reads, the retained-value distinctions, and byte-before-word publication ordering.

The representation does not establish dependency internals, valid mappings, application intent, or authoritative machine behavior for unresolved arithmetic. Comments concerning unreachable material and terminal behavior do not supply additional executable behavior.

No runtime tests have been performed.
