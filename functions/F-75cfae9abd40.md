<!-- Trusted provenance: F-75cfae9abd40; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-75cfae9abd40

## Purpose

F-75cfae9abd40 has an established effect-level contract: it performs ordered memory writes, obtains two 16-bit memory observations, and exchanges values with opaque dependencies. Some dependency outputs are retained across many subsequent interactions and later determine addresses, stored data, or outgoing arguments.

No application-level purpose is established. In particular, the evidence does not justify describing this operation as initialization, decoding, reset, allocation, or cleanup. Repeated address scaling does not establish a buffer, record layout, or common containing object.

This specification distinguishes three evidence levels:

- **Established static evidence:** represented access widths, value provenance, argument positions, retention relationships, and ordering.
- **Inferred intent:** possible interpretations that create no behavioral obligation.
- **documented_only:** unresolved behavior requiring additional evidence.

Requirements apply conditionally when execution reaches the relevant operation. They do not establish that the entire represented path can execute successfully. No runtime tests have been performed.

## Interface

### Explicit inputs

Four explicit argument positions are represented.

| Position | Represented type information | Established contribution |
|---|---|---|
| First | Signed 32-bit integer | Determines the first byte-write address and both displayed arguments of F-D01. |
| Second | Uninterpreted 32-bit value | No explicit contribution to the represented calculations. |
| Third | Uninterpreted 32-bit value | No explicit contribution to the represented calculations. |
| Fourth | Signed 32-bit integer | Determines the third argument at an early four-position dependency boundary. |

The second and third positions have no established meaning as lengths, flags, capacities, or handles. Their lack of explicit use does not establish the absence of hidden calling-convention inputs.

Additional incoming state supplies an eight-bit pattern and control-related information used to select the initial dependency target. Neither is established as being derived from the explicit arguments. The incoming byte is not restricted to Boolean values.

### Dependency results and retained state

The function uses several distinct result channels:

- Signed 32-bit primary results.
- An unsigned 32-bit primary result from which two byte patterns are derived.
- Three 64-bit results.
- An early auxiliary signed 32-bit result.
- An auxiliary pointer to signed 32-bit storage.
- A different auxiliary signed 32-bit result retained for later address formation and argument forwarding.
- Seven further auxiliary 32-bit results, each forwarded separately.

Primary and auxiliary results are not interchangeable. Equality of width or numerical value does not erase their different provenance.

The pointer is retained as an access capability. Its production does not establish ownership, allocation, lifetime, or permission to release its target.

### Outputs and preconditions

No ordinary result value, success status, or recoverable-error result is established. Observable outputs consist of reached memory effects and dependency interactions.

Each reached byte write requires an accessible one-byte destination. Each observation requires an accessible two-byte extent. The late pointer-directed write requires accessible 32-bit storage at that time. Dependency targets must support the represented invocations.

The function contains no explicit validation of these requirements. Numerical address zero is not rejected. Pointer validity at an earlier boundary does not establish validity after intervening dependencies.

## Behavior

### Ordering and arithmetic interpretation

The supplied representation presents a single ordered path without an explicit loop or conditional branch. Repetition is not evidence of retries, optional stages, or early-success exits.

Memory access widths and their ordering relative to dependency interactions are contractual. Equal addresses do not permit effects to be merged across dependency boundaries. Dependencies whose primary results are unused remain observable interactions.

Unless specifically identified as unsigned, the address-producing integer calculations are represented at signed 32-bit width. References below to left shifts describe those width-sensitive operations, not unrestricted multiplication. Negative signed operands, nonrepresentable signed shifts, and signed multiplication or addition overflow have no established portable numerical outcome.

Unsigned 32-bit left shifts retain the low 32 result bits. Conversion into the represented uninterpreted one-byte storage retains eight bits. The two writes expressed through character conversions require the additional qualification described under Uncertainties.

Numerical address relationships do not establish effective address width, pointer representation, alignment rules, or memory mapping.

### Direct memory-effect inventory

If all represented direct accesses are reached, the byte writes occur in these consecutive value families:

| Value family | Number of byte writes |
|---|---:|
| Incoming eight-bit pattern | 20 |
| Zero | 6 |
| Pattern derived from that write’s numerical destination | 1 |
| Retained byte derived from the unsigned result | 13 |
| Alternate selection from the same unsigned result | 1 |
| Recurring value in the range zero through seven | 21 |
| **Total** | **62** |

There are also two direct 16-bit reads, one pointer-directed 32-bit write, and one assignment to separately identified storage whose declaration is unavailable. The physical width of that assignment is **documented_only**; it is not included in the byte-write count.

Counts describe direct represented accesses, not dependency-internal accesses. They imply neither distinct destinations nor adjacency.

### Incoming-pattern propagation

The first effect is a byte write at twice the first explicit argument. It precedes F-D01, which receives two equal arguments formed by a 12-bit left shift of that input.

The remaining nineteen writes in this family obtain their address bases from dependency results. Ordinarily, the byte destination uses a three-bit left shift of its producer result, and the following consumer receives two equal arguments formed by a 14-bit left shift of the same result.

The following distinctions are required:

- The fourth write uses twice its producer result as its destination. Its following consumer receives the producer shifted left by 12 bits twice. The producer is selected through a different target-selection relationship from its neighboring interactions.
- The fifth write is followed by a four-position consumer. Positions one, two, and four contain the producer shifted left by 14 bits. Position three contains the fourth explicit argument shifted left by ten bits.
- Immediately after that four-position interaction, a single-position consumer receives the early auxiliary integer shifted left by three bits. Its input is not the primary return channel.
- The twentieth write retains the usual three-bit-shifted destination, but its following consumer receives the signed product of its producer by four twice.

The already-scaled argument supplied at the sixteenth write is retained for much later use. Its provenance is the 14-bit-shifted value, not the original producer and not a subsequent dependency result.

Two 64-bit forwarding boundaries occur between writes seventeen and eighteen and between writes eighteen and nineteen. Their layouts are specified under Dependencies.

### Zero and destination-derived writes

After the incoming-pattern family, six writes each store one byte of zero. Each uses a distinct dependency-produced basis, a destination formed by a three-bit left shift, and a following consumer receiving the same basis shifted left by 14 bits twice.

The next byte value is derived from its own numerical destination rather than from a memory read. Under low-eight-bit character-storage interpretation, the stored pattern equals the destination’s low eight bits.

Its following consumer has three displayed positions: the basis shifted left by 14 bits twice, then the destination value formed by the three-bit shift. This does not establish validation, address registration, or readback semantics.

### Retained-byte propagation and separate-storage assignment

A distinct unsigned 32-bit dependency result determines a retained byte: the low eight bits after a three-bit unsigned left shift. That byte supplies thirteen direct byte writes and the separate-storage assignment. Later destination-producing results do not replace its provenance.

The first retained-byte destination uses the unsigned result shifted left by four bits. The following consumer receives that result shifted left by 15 bits twice.

The next six writes have wider address relationships:

- The first uses the scaled value retained from incoming-pattern write sixteen. Its destination applies a further 16-bit left shift; the following consumer applies a further 27-bit left shift.
- The next three use distinct newer signed results. Each destination applies a 13-bit left shift, and its following consumer receives the corresponding 24-bit-shifted value twice.
- The fifth uses a retained signed 32-bit product of the earlier scaled value by 32,768. Its destination applies a further 12-bit left shift, and its following consumer receives a further 23-bit left shift twice.
- The sixth uses another newer signed result, a 13-bit-shifted destination, and the first direct memory-observation boundary.

The product by 32,768 is formed before the first of these six writes. It is not a later reconstruction from a newer address result. Its width and production history matter both here and when deriving the recurring small value.

After the first observation and intervening dependency interactions, the retained byte is assigned to separate storage. The next consumer receives two explicit zeros. The storage’s label and those zero arguments do not establish reset semantics or preservation across the consumer.

The final six retained-byte writes occur after this assignment. Their destinations use newer signed results shifted left by three bits. Their following consumers ordinarily receive equal 14-bit-shifted arguments; the last instead receives its producer’s signed product by four twice.

### Two memory observations

Both observations read actual 16-bit memory contents for outgoing arguments. Neither is established as a bounds check or as a readback of an earlier write.

**First observation.** The relevant signed producer determines both the preceding retained-byte write and the read address. The byte destination uses a 13-bit left shift. The observation begins 56 bytes beyond the base obtained by multiplying that producer by 4,096. The byte write precedes the read, which precedes the consumer invocation. That consumer receives the unoffset base first and the observed contents second.

**Second observation.** The retained auxiliary pointer is converted to a signed 32-bit integer representation for address calculation. The observation begins 32 bytes beyond 128 times that representation. It follows small-value write fifteen, whose destination comes from a newer primary result rather than from the pointer. The following consumer has one displayed position containing the observed contents.

The separate retained auxiliary integer does not determine the second observation’s address. Memory byte order, alignment requirements, and outgoing extension of the 16-bit contents are **documented_only**.

### Alternate byte and recurring small-value family

The alternate-selection write uses the earlier unsigned producer after retaining only its low 29 bits and shifting right by 19 bits, followed by a character conversion. Before conversion, this selection can contain ten bits. Under low-eight-bit character-storage interpretation, the stored byte corresponds to producer bits 19 through 26; bits 27 and 28 are discarded by that final reduction.

Its destination uses a newer signed result shifted left by three bits. The following consumer receives that result’s signed product by four twice.

The final twenty-one byte writes share a small value derived from the retained product by 32,768. The product is viewed as unsigned, its high byte is selected, and that byte is shifted right by five bits. The resulting value is between zero and seven and corresponds to product bits 29 through 31.

This is a conditional relationship to the retained product. It is not an unconditional selection from an earlier unscaled producer. Reusing an already reduced byte does not reduce it again.

Most writes in this family use a newer signed result shifted left by three bits for the destination and a following consumer with two equal 14-bit-shifted arguments. Exceptions are:

| Family position | Observable distinction |
|---|---|
| 2 | Address and following arguments depend on unresolved width-sized shift arithmetic. |
| 4 | Destination derives from the auxiliary pointer’s signed integer representation using an eight-bit left shift; the consumer receives its 19-bit shift twice. |
| 9, 16, 20 | The immediately following invocation has no displayed arguments. |
| 11 | Destination derives from the separate retained auxiliary integer using an eleven-bit left shift; the consumer receives its 22-bit shift twice. |
| 15 | Followed by the second 16-bit observation and its single-position consumer. |
| 17 | Followed by the pointer-directed 32-bit write and a three-position consumer. |

At position two, the represented calculation includes a signed 32-bit left shift by two followed by a right shift by 32. That right-shift count equals the operand width. Consequently, neither the doubled destination nor the following pair of 12-bit-shifted arguments has an established portable value. Zero, sign propagation, masked counts, and widened evaluation are not interchangeable resolutions.

### Late word write and terminal boundary

At small-value position seventeen, a primary signed result determines a three-bit-shifted byte destination and a signed product by four. The byte write precedes the 32-bit write of that product through the retained auxiliary pointer. The following consumer receives the product in three equal positions.

If the byte and word destinations overlap, the word write is later. Its byte-level effect depends on memory byte order. No atomic publication or synchronization property is established.

Later interactions forward one primary result unchanged in two positions and, separately, pair a newer primary result shifted left by two bits with the retained word value. That second argument is retained data, not an explicit reread through the pointer.

Four small-value byte writes remain after the word write. Further dependency interactions remain after the final byte write.

F-D05 receives selector zero and fixed numerical marker 142,643,732. Its represented result supplies a target subsequently invoked without displayed arguments. Successful completion, an inevitable trap, target validity, and continuation are **documented_only**.

## Side effects

Writes may alias one another, the separately identified storage, executable memory, or dependency-accessed memory. No common allocation, contiguous layout, or non-aliasing guarantee is established.

Each write establishes its immediate reached effect, not persistence through subsequent calls. Unknown dependencies may modify globals or pointees, invalidate storage, or introduce additional accesses. Retaining a pointer value does not preserve its target’s lifetime.

No direct allocation, release, ownership transfer, or eight-bit memory read is established.

There is no represented rollback or recoverable-error branch. A faulting access or nonreturning dependency may leave a completed prefix of effects. Exception propagation, fault delivery, atomicity, thread safety, reentrancy, idempotence, and safe reinvocation are **documented_only**.

## Dependencies

These opaque IDs identify documentation roles, not established unique target identities. Different occurrences within a role may invoke different targets.

| ID | Established role |
|---|---|
| F-D01 | Initial interaction selected using incoming control-related state. |
| F-D02 | Primary-result production, consumption, and forwarding. |
| F-D03 | Auxiliary-result production and consumption. |
| F-D04 | 64-bit-result production and ordered word forwarding. |
| F-D05 | Terminal interaction supplying the represented final invocation target. |

F-D02 includes interactions with unused primary results, unchanged 32-bit forwarding, two-bit-left-shifted forwarding, and a distinct one-bit-left-shifted forwarding between the final retained-byte write and the alternate-selection write. Invocations without displayed arguments are not established as input-free or side-effect-free.

F-D03 has independent provenance requirements:

- The early auxiliary integer supplies the shifted single argument after the four-position boundary.
- The auxiliary pointer supplies small-value destination four, the second observation’s address basis, and the late word-write destination.
- The separate auxiliary integer is retained after small-value write four. It supplies destination eleven and the unchanged second position of four later paired consumers.
- Those four paired consumers occur between small-value writes eleven and twelve, twelve and thirteen, thirteen and fourteen, and fourteen and fifteen. Their first positions use newer primary results shifted left by two bits.
- Seven further auxiliary outputs are forwarded unchanged to individual single-position consumers. These occur between small-value writes four and five, five and six, six and seven, seven and eight, ten and eleven, eighteen and nineteen, and nineteen and twenty.

For F-D04, each 64-bit result is partitioned numerically into lower and upper 32-bit words. The two early consumers receive lower, upper, upper. The later consumer, between small-value writes two and three, receives lower, upper. These layouts describe argument positions, not memory serialization.

Full signatures, hidden inputs, preservation rules, ownership, allocation, release, failures, and termination are **documented_only**. Grouping interactions by role does not authorize omitting intervening calls.

## Acceptance examples

These are conditional static obligations, not executed tests. They assume reachability, suitable memory and targets, and an arithmetic interpretation supporting the stated values.

1. **Entry ordering:** First argument 16 and incoming pattern 165 require a byte write of 165 at numerical address 32 before F-D01 receives 65,536 twice. A failing write may prevent the call.

2. **No zero-address shortcut:** First argument zero produces numerical destination zero. An access-free early return is not established.

3. **Access-width preservation:** Replacing a zero-byte write with a four-byte clear changes neighboring effects and possible fault behavior; it is not equivalent.

4. **Argument provenance:** At the four-position boundary, producer basis 2 and fourth explicit argument 3 give 32,768, 32,768, 3,072, and 32,768. The next single argument must derive from the auxiliary result, even if a primary result differs.

5. **Retained byte:** Unsigned producer 35 establishes byte 24, first retained-byte destination 560, and equal consumer arguments 1,146,880. Newer address producers do not redefine byte 24.

6. **Multiword layout:** Lower word 7 and upper word 9 require arguments 7, 9, 9 at either early F-D04 consumer, but 7, 9 at the later consumer.

7. **First observation:** Producer 2 gives preceding byte destination 16,384 and observation address 8,248. The consumer receives 8,192 followed by the actual 16-bit contents observed after the byte write.

8. **Character-conversion qualification:** Under low-eight-bit character storage, toggling unsigned-producer bit 19 toggles the alternate byte’s least significant bit. Changing only bits 27 or 28 leaves that stored byte unchanged. This example does not resolve other character-conversion interpretations.

9. **No repeated reduction:** If the execution interpretation supplies a retained product whose unsigned high byte is 160, the recurring value is 5. Later reuse remains 5, not zero. This does not prove that such a product is reachable under portable signed arithmetic.

10. **Pointer-derived observation:** Pointer integer representation 256 gives observation address 32,800. The outgoing argument is the contents there, not the pointer representation or the separate auxiliary integer.

11. **Word-write ordering:** Late producer 9 gives byte destination 72 and word value 36. The byte write precedes the pointer-directed word write; the consumer receives 36 three times. Four later byte writes remain if reached.

12. **Unresolved boundaries:** Acceptance cannot choose the width-sized shift’s destination by assuming familiar processor behavior, or replace the terminal interaction with an assumed trap or successful return.

## Uncertainties

- **documented_only — Arithmetic:** signed overflow; negative or nonrepresentable signed shifts; the width-sized right shift; character conversion outside the destination character range; pointer-to-integer and integer-to-pointer conversion; effective address width.
- **documented_only — Memory:** mapping, alignment, accessible extents, byte order, aliasing, lifetime, and fault delivery.
- **documented_only — Interface:** hidden incoming state, auxiliary-output conventions, argument extension, and calling convention.
- **documented_only — Dependencies:** exact identities, complete interaction signatures, preservation, global effects, ownership, allocation, release, failures, and exceptional control flow.
- **documented_only — Separate storage:** declaration, physical assignment width, address, and application meaning.
- **documented_only — Completion:** whole-path reachability through unresolved arithmetic, terminal result production, final invocation behavior, and continuation.
- **Inferred intent:** scaling and repeated argument layouts suggest structural regularity, but establish no application purpose, data format, or success criterion.

## Evidence

The behavioral claims are grounded in the supplied static representation’s explicit access widths, value origins, retained-data relationships, argument positions, and relative ordering. The direct-access inventory excludes dependency-internal effects and the separately identified assignment of unknown width.

Comments, labels, the prior draft, and reviewer feedback are not independent behavioral authority. No dependency bodies, authoritative calling-convention description, memory map, or machine-level resolution of unsupported arithmetic were supplied.

No runtime tests have been performed. Acceptance examples express conditional requirements; they do not demonstrate reachability, valid storage, dependency preservation, or completion.
