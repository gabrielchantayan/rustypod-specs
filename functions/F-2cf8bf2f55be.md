<!-- Trusted provenance: F-2cf8bf2f55be; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2cf8bf2f55be

## Purpose

This routine exposes an ordered series of dependency interactions and memory effects, ending in a terminal control transfer rather than an ordinary return.

Its application-level purpose is not established. The supplied evidence does not justify classifying it as initialization, reset, allocation, serialization, or resource release. Repetition and descriptive labels do not establish those meanings.

The contract concerns:

- Which externally supplied values determine payloads, addresses, and dependency arguments.
- The widths and ordering of direct memory accesses.
- Values retained across intervening dependency interactions.
- The conditions under which later effects are reachable.
- Boundaries beyond which behavior remains unresolved.

**Established static evidence** means behavior represented in the supplied body, not independently verified executable behavior. **Inferred intent** imposes no acceptance requirement. **documented_only** identifies unresolved behavior that must not be replaced with an assumed convention.

## Interface

### Entry state and completion

The represented interface has no explicit parameters and no returned value. It consumes implicit entry state:

| Entry resource | Required role |
|---|---|
| Control/link value | Determines the initial dependency target |
| Eight-bit value | Supplies the initial byte payload |
| 32-bit scalar | Contributes to one four-argument dependency interaction |
| Execution environment | Supports computed control transfers and memory accesses |

Further inputs arrive through dependency results. These include ordinary 32-bit values, an unsigned 32-bit value, three represented 64-bit results, and secondary-channel values interpreted as scalars or a pointer.

The actual calling convention, input locations, initialization requirements, and secondary-channel ABI are **documented_only**. A call with no displayed arguments does not prove that its target consumes no implicit state.

There is no represented success result, error status, returned object, or normal completion notification. Completion of the visible path leads to the terminal facility described below.

### Access widths and layout

The direct-access contract distinguishes:

| Access | Represented extent |
|---|---|
| Computed-address payload write | One byte |
| First explicit data read | Two bytes beginning 56 bytes beyond a computed base |
| Second explicit data read | Two bytes beginning 32 bytes beyond another computed base |
| Retained-pointer write | Four bytes beginning at the pointer itself |
| Global assignment | Assignment of an eight-bit-derived value; destination declaration unavailable |

A byte destination does not establish an eight-byte record, array stride, object header, or allocation size. Likewise, the read offsets establish access locations, not structure definitions.

The two-byte reads supply 16-bit quantities to dependencies. Their argument-register representation and any extension beyond 16 bits are not established by a dependency signature.

Byte order, address-space interpretation, pointer conversion, and machine alignment requirements are **documented_only**. Logical low and high halves of 64-bit values must be distinguished from memory byte order.

### Preconditions and arithmetic domain

Each event requires all preceding dependency interactions to return control and all preceding accesses to complete. The environment must permit execution at computed call targets and the required accesses at computed data addresses.

No explicit checks reject null, out-of-range, misaligned, inaccessible, or expired addresses. Zero and high-bit-set results are not visibly screened before use. This absence of checks is not a guarantee that such inputs are valid.

Unless stated otherwise, the represented scalar and address transformations are 32-bit operations. Numerical scales in this contract describe value relationships at that width, not unrestricted mathematical products.

For the explicitly unsigned 32-bit result, left scaling discards bits beyond the 32-bit result width. For signed-looking transformations, overflow and invalid shift behavior cannot be resolved from this representation alone. Pointer-to-integer conversions also remain unresolved.

One right shift requests a distance equal to the represented operand width. That event has no established numerical interpretation.

## Behavior

### Observable ordering

The supplied body represents a single ordered path, without an explicit conditional branch or loop. This does not guarantee that the path finishes: a dependency can fail to return, and a memory access can prevent further execution.

Dependency boundaries are observable even when their displayed results are unused. They must not be removed, combined, cached, or reordered on the assumption that repetition means redundancy.

The initial target depends on the incoming control/link value with a displacement of 4,094 bytes. Ordinary targets use fixed control-location anchors with that displacement. One exceptional target construction uses downward alignment to a four-byte boundary and a displacement of 1,536 bytes. These are computed transfers; no function-pointer-table lookup is established.

The following recurring interactions matter independently of the direct memory effects:

- A primary result forwarded after scale-four transformation.
- An unchanged 32-bit result forwarded to another dependency.
- The result of a scale-four interaction forwarded unchanged.
- Equal arguments supplied twice or three times.
- A fresh transformed value accompanied by an older retained value.
- Secondary-channel values forwarded unchanged or, at one earlier boundary, scaled by eight.
- A distinct single-argument scale-two interaction.

These descriptions identify argument relationships, not interchangeable dependency identities.

### Payload identity and retention

Direct byte writes use six distinguishable payload classes:

| Payload class | Required value relationship |
|---|---|
| Entry payload | The incoming eight-bit value |
| Constant payload | Zero |
| Destination-derived payload | The least significant eight bits of that write’s destination |
| Retained payload | The least significant eight bits of eight times a particular unsigned 32-bit dependency result |
| Separate extracted payload | Original bits 19 through 26 of that same unsigned result |
| Three-bit payload | The upper three bits of a separately retained scalar’s most significant byte |

These classes must not be collapsed merely because particular inputs make their values equal.

The retained payload remains associated with its original producing result, not with later destination-producing results. The three-bit payload likewise retains its own provenance.

Retention is a represented data dependency. It does not independently establish which registers survive a call, whether storage is protected from modification, or what preservation obligations unknown dependencies satisfy.

### Entry-payload effects

Most entry-payload writes use a destination eight times the relevant dependency result. The associated following dependency receives that result scaled by 16,384 in both displayed argument positions. The byte write occurs before that interaction.

Three exceptions constrain implementations:

- The specially aligned-target dependency receives a scale-four argument. Its result determines a byte destination at scale two; the following paired arguments use scale 4,096.
- One ordinary scale-eight destination is followed by paired scale-four arguments, not paired scale-16,384 arguments.
- One write has a four-argument consumer. Positions one, two, and four contain the same scale-16,384 value derived from the destination-producing result. Position three contains the incoming 32-bit scalar scaled by 1,024.

The four-argument interaction is followed by a dependency consuming a secondary-channel value scaled by eight.

Some paired consumers produce results that subsequently participate in scale-four forwarding. Their results must not be confused with the earlier values used to calculate the byte destinations.

A scalar retained during a late entry-payload event is already the destination-producing result scaled by 16,384. Later large-scale effects depend on this retained scalar, not an unrelated fresh result.

### Constant and destination-derived effects

After the entry-payload region, six direct byte writes use zero. Their destinations use scale eight, and their associated paired arguments use scale 16,384.

These writes are unconditional in the represented path. There is no preceding direct read that tests whether clearing is necessary. The zero payload does not depend on the entry byte or existing destination contents.

A later distinct write stores the least significant byte of its own scale-eight destination. Its following dependency receives two equal scale-16,384 arguments and the full scale-eight destination value as the third displayed argument.

That destination-derived value is not the retained payload introduced afterward.

### Retained-payload effects and first read

A particular unsigned 32-bit dependency result establishes the retained payload. Its first destination is sixteen times that result, while the payload comes from scaling by eight and retaining one byte. The following paired arguments use scale 32,768.

The retained payload then participates in these nonordinary address relationships, in the indicated relative order:

| Address basis | Destination scale | Associated consumer relationship |
|---|---:|---|
| Earlier retained entry-region scalar | 65,536 | Equal arguments at scale 134,217,728 |
| Three fresh dependency results at separate boundaries | 8,192 | Equal arguments at scale 16,777,216 |
| A second retained scalar derived from the earlier retained scalar | 4,096 | Equal arguments at scale 8,388,608 |
| A later fresh result | 8,192 | Computed base and a loaded 16-bit quantity |

The second retained scalar is the earlier retained scalar multiplied by 32,768. These are successive 32-bit transformations; combining their scales using unbounded arithmetic is not generally justified.

At the first explicit read boundary:

- The retained-payload byte write completes first.
- The read begins 56 bytes beyond a base equal to 4,096 times the relevant fresh result.
- The following dependency receives that base first and the loaded 16-bit quantity second.

No dependency invocation separates that byte write from the read.

Later, the retained payload is assigned to a global before a dependency receives two zero arguments. The assignment does not establish reset semantics. Subsequent dependencies may alter the global.

Additional retained-payload writes use scale-eight destinations. One is followed by paired scale-four arguments. A nearby, separate interaction receives one scale-two argument.

### Separate extraction and three-bit effects

The separate extracted payload comes from original bits 19 through 26 of the unsigned result that established the retained payload. Although the represented extraction first restricts the source to 29 bits, the final one-byte truncation excludes original bits 27 and 28.

This payload is written at a scale-eight destination produced by another dependency. Its following paired arguments use scale four.

The three-bit payload has a different origin: the most significant byte of the second retained scalar is reduced to its upper three bits. Its numerical range is zero through seven, not merely false or true.

If the earlier retained-scalar transformations are interpreted as 32-bit modular arithmetic, this payload simplifies to the low three bits of the original result underlying the earlier retained scalar. That is a **conditional deduction**, not a resolution of signed overflow semantics.

Most three-bit writes use scale-eight destinations followed by equal scale-16,384 arguments. Exceptions include:

| Exceptional basis | Byte destination relationship | Following interaction |
|---|---|---|
| Intermediate affected by the unresolved full-width shift | Twice that intermediate | Equal arguments scaled by 4,096 |
| Retained secondary-channel pointer, interpreted numerically | Pointer value scaled by 256 | Equal arguments scaled by 524,288 |
| Retained secondary scalar | Scalar scaled by 2,048 | Equal arguments scaled by 4,194,304 |
| Several ordinary scale-eight destinations | Unchanged address relationship | Calls with no displayed arguments instead of the usual pair |

The unresolved intermediate involves a scale-four transformation followed by a right shift of 32 on a quantity represented as 32 bits. This contract requires preserving the unresolved boundary. It does not select zero, the unchanged operand, sign fill, wider arithmetic, or any processor-specific shift convention.

Near the pointer-derived write, another secondary scalar becomes retained state. That scalar later accompanies fresh scale-four results at four separate two-argument interactions.

Other later interactions forward distinct secondary-channel values unchanged. Those values must not be replaced by ordinary primary results or assumed equal to one another.

### Multiword interactions

Three dependency results are represented as 64-bit quantities.

The first two each have a consumer receiving three displayed 32-bit arguments: low half, high half, high half. These events occur during the entry-payload portion.

The third occurs after the unresolved-shift boundary and has a two-argument consumer receiving low half, then high half.

These are logical argument relationships. They do not authorize merging the halves into one argument, swapping them to match memory byte order, or eliminating the repeated high half.

### Pointer-directed read, word write, and ending

The second explicit 16-bit read occurs after a three-bit byte write. Its address is 128 times the numerical retained pointer value plus 32 bytes. The loaded quantity is the following dependency’s sole displayed argument.

This calculation does not first read the word stored at the retained pointer. The pointer’s numerical value, rather than its contents, supplies the address basis.

At a later boundary, two direct writes occur without a dependency invocation between them. A three-bit byte is written at eight times a fresh result. Then four times that same result is written as a 32-bit quantity at the retained pointer itself. Both writes precede a dependency receiving three copies of the scale-four value.

The word destination is not the scale-128 read address or the scale-256 byte destination. The scale-four word value also becomes retained state and later accompanies another fresh scale-four result.

Following the word boundary, an unchanged dependency result is supplied twice to another dependency. Further calls and three-bit writes remain observable.

The final byte write is not the final event. Additional dependency interactions precede F-U003. That facility receives a zero selector and a fixed terminal-site token. If it supplies a callable target, F-U004 is invoked. The ending is represented as non-returning; its concrete exception, handler, or termination mechanism is **documented_only**.

## Side effects

The established direct effects are computed-address byte writes, two 16-bit reads, one global assignment, one 32-bit pointer-directed write, dependency transfers, and the terminal sequence.

No rollback, cleanup, retry, transactional boundary, or error-recovery path is represented. If execution is interrupted, later effects are not required to occur; earlier effects are not visibly undone.

Addresses are not proven disjoint. Byte destinations, read spans, the global, the word destination, and dependency-visible storage may overlap. Consequently:

- Write-before-read ordering matters even when a write overlaps a read span.
- The byte-before-word order matters when their spans overlap.
- A later direct write or dependency effect may replace an earlier value.

The routine establishes no allocation, release, ownership transfer, reference-counting, or exclusive-access protocol. The retained pointer is not proven newly allocated, independently owned, or valid throughout its later uses.

Unknown dependencies may modify globals, pointed-to contents, other memory, resources, and control flow. Neither the global assignment nor a completed write establishes the eventual value after later calls.

Atomicity, thread safety, reentrancy, idempotence, and volatility properties are not established.

## Dependencies

The dependency IDs below are opaque document-local boundaries.

| ID | Role |
|---|---|
| F-U001 | Initial entry-dependent computed call; its result participates in scale-four forwarding |
| F-U002 | Collection of ordinary computed-call boundaries, including the exceptional aligned-target construction |
| F-U003 | Terminal facility receiving a zero selector and fixed site token |
| F-U004 | Callable target supplied by F-U003, if that facility provides one |

F-U002 denotes multiple unresolved targets, not one shared implementation. Similar target construction, argument shape, or repetition does not prove common identity.

No dependency is assumed to preserve globals, maintain pointer validity, own or release memory, return normally, or follow a conventional success/failure protocol.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Each assumes that its boundary is reached, required memory is accessible, and the stated numerical interpretation is applicable.

### Ordinary byte visibility

With entry byte 0xA5 and an ordinary destination-producing result of 3, address 24 receives 0xA5 before the associated dependency receives 49,152 twice.

Writing zero instead, using a wider direct store, or moving the write after the call violates this criterion.

### Exceptional argument scale

For the scale-eight entry-payload exception and result 3, address 24 receives the entry byte, but the following arguments are 12 and 12.

The corresponding retained-payload and separate-extraction exceptions have the same address/argument scales but different payload provenance.

### Retained payload and truncation

An unsigned producing result of 35 yields retained payload 24: eight times 35 is 280, whose least significant byte is 24. The first destination is 560 because it uses scale sixteen.

Later destinations derived from other results still receive 24.

### Extraction distinction

For unsigned source 0x07F80000, the separate extracted payload is 0xFF, while the retained scale-eight payload is zero.

Substituting one payload for the other fails even though both originate from the same producing result.

### Three-bit boundaries

A retained most significant byte of 0x1F yields payload zero; 0x20 yields one; 0xFF yields seven.

Boolean conversion fails the last criterion.

### First read ordering

For fresh result 2 at the first read boundary, the retained-payload byte destination is 16,384. The two-byte read begins at 8,248. The consumer receives base 8,192 and the loaded quantity as separate arguments.

The byte write must precede the read.

### Multiword order

For a represented 64-bit result with low half 0x9ABCDEF0 and high half 0x12345678, either three-argument multiword consumer receives low, high, high. The later two-argument consumer receives low, high.

No memory byte-order assumption changes this ordering.

### Pointer address versus contents

With numerical retained pointer value 256, the second two-byte read begins at 32,800. The separate pointer-derived byte destination is 65,536. The later four-byte write begins at 256 itself.

Changing the word stored at address 256 does not, by itself, change those numerically derived addresses.

### Adjacent writes and interruption

For word-event result 3, address 24 receives the three-bit byte before address 256 receives the four-byte value 12, assuming retained pointer value 256. The following consumer receives 12 in all three positions.

If the byte write prevents further execution, neither the word write nor that consumer is required. No rollback of prior effects is implied.

No numerical acceptance value is assigned to the full-width-shift intermediate without additional evidence.

## Uncertainties

The following remain **documented_only**:

- Actual dependency identities, signatures, implicit inputs, and preservation rules.
- Entry calling convention and valid initial execution state.
- Signed overflow, overflowing shifts, and the right shift by the operand width.
- Pointer/integer conversion and secondary-channel ABI details.
- Byte order, alignment, permissions, lifetimes, and accessible address ranges.
- Global storage width, semantic role, and eventual value.
- Ownership, aliasing, synchronization, and dependency resource effects.
- Concrete fault handling and terminal control behavior.
- Fidelity of the supplied representation to executable instructions.

This contract deliberately does not publish a per-invocation catalogue of private control-location anchors. Its dependency-family descriptions are not permission to substitute arbitrary calls with matching argument shapes.

**Inferred intent:** the repetition may reflect mechanically expanded behavior. It does not establish redundancy, safe deduplication, or an application-level operation.

## Evidence

The supplied body supports the access widths, payload relationships, retained-value dependencies, argument ordering, and direct-effect ordering described here. Numerical acceptance examples are deductions from those relationships.

Prior prose, source labels, and comments are not independent behavioral evidence. In particular, commentary about omitted unreachable material establishes no additional reachable behavior.

Only static examination of the supplied material has been performed. No dependency-body inspection, machine-instruction inspection, implementation access, runtime tests, smoke runs, or empirical verification has been performed.
