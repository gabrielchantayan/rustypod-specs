<!-- Trusted provenance: F-915e1bdbe86d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-915e1bdbe86d

## Purpose

F-915e1bdbe86d performs an ordered stream of indirect dependency interactions and direct memory effects. Its observable outputs are byte stores, two halfword reads whose values become dependency arguments, a delayed word store, an assignment to a distinguished global location, and a terminal indirect transfer.

No application-level purpose is established. Address scaling does not establish an array, record format, device protocol, or allocation scheme. The global assignment does not establish a reset operation.

The supplied representation contains no normal return path. Completion of its represented sequence remains conditional on preceding dependencies returning and preceding operations permitting continuation.

This contract separates:

- **Established static evidence:** relationships visible in the supplied representation.
- **Inferred intent:** interpretations that impose no acceptance requirement.
- **documented_only:** behavior requiring information not supplied.

No runtime tests have been performed.

## Interface

### Invocation and inputs

The displayed interface has no declared parameters and no returned value. It nevertheless consumes inherited execution state and dependency-produced data.

| Input | Required distinction |
|---|---|
| Incoming return-link value | Selects the opening indirect target. |
| Inherited eight-bit value | Supplies the initial byte-store family; no initialization inside this operation is established. |
| Additional inherited 32-bit word | Contributes one argument to a four-argument interaction. |
| Primary dependency results | Supply addresses, argument data, retained values, and four apparent 64-bit results. |
| Secondary dependency-result state | Supplies an address, a retained word, an early scaled argument, and seven later unchanged single arguments. |
| Memory and execution environment | Determines target validity, access permissions, arithmetic interpretation, and whether execution continues. |

An interaction with no displayed arguments is not evidence that its complete execution-level interface has no inputs.

Primary results and secondary-result state are not interchangeable. A newer primary result does not replace a separately retained secondary value merely because it is more recent.

### Widths and layouts

Direct memory accesses have these established widths:

- Byte stores: eight bits.
- Two halfword reads: 16 bits each.
- Delayed word store: 32 bits.

Address-bearing numerical values and ordinary words are represented as 32-bit quantities. Four results are represented as 64-bit quantities and divided numerically into lower and upper 32-bit portions.

The halfword reads use byte offsets 56 and 32 from different scaled bases. No containing object, object extent, field identity, or record stride is established. Numerical separation of a 64-bit value does not establish an in-memory byte order.

The global location, identified here as G-01, receives a byte-valued expression. Its declaration and physical store width are **documented_only**.

### Arithmetic domain

Scaling relationships below describe mathematical values only where the represented operations support them.

Explicitly unsigned 32-bit scaling retains the low 32 bits. Unsigned byte narrowing retains the low eight bits. Signed 32-bit calculations must not silently be replaced with wider arithmetic or unsigned wrapping. Negative signed shifts, signed overflow, and numerical-address conversions remain **documented_only**.

Historical scaling has intermediate 32-bit boundaries. Combining several factors into one mathematical product does not establish that the product was computed in a wider temporary.

Two byte stores use character conversion. Without character signedness and conversion rules, out-of-range inputs do not establish a portable low-byte result. This affects both the address-derived store and the later extracted-data store.

One address calculation includes a right shift by 32 of an apparently 32-bit signed quantity. Its result is unresolved; no zero result, masked shift count, or wider interpretation is assumed.

### Preconditions and outputs

Every reached indirect target must support the required transfer. Every reached memory access requires suitable mapping, permissions, and alignment behavior. Retained addresses must remain usable until their final reached access.

These are environmental requirements for continuation, not validated conditions. No visible null exclusion or bounds check protects the accesses.

There is no established success result, error result, or normal return value.

## Behavior

### Observable ordering

The represented path is linear: no data-dependent branch or loop is displayed. That does not make it an unconditional transaction. A dependency may fail or not return; an access may fault; unresolved arithmetic may prevent a meaningful continuation.

The direct-effect progression is:

1. Stores of inherited byte data.
2. Six explicit zero-byte stores.
3. One address-derived byte store.
4. Stores of retained byte data, including the first halfword read and G-01 assignment.
5. One extracted-data byte store.
6. Stores of a three-bit value, including the second halfword read and delayed word store.
7. Remaining dependency activity and terminal transfer.

This progression does not imply adjacent accesses. Dependencies separate most effects, including individual members of repeated store families. Their effects are unknown, so they cannot be discarded, reordered, or merged merely because a displayed result is unused.

The opening interaction precedes all direct stores. Its target depends on the incoming return-link value plus 4,094 bytes. Ordinary interactions use occurrence-specific fixed execution references with that displacement. Two exceptional interactions have different target-selection relationships, described under Dependencies.

### Inherited-data stores

The initial family stores the inherited eight-bit value without a displayed read of the destination.

For the ordinary relationship, the byte destination is eight times a producing result. The following dependency receives two equal arguments, each 16,384 times that producer. The store occurs before that interaction.

The family also contains these distinct consumer contracts:

- The first store precedes F-U03, which receives four times the producer and supplies the first apparent 64-bit result.
- F-U04 receives four times an earlier result. Its result determines a byte destination scaled by two, followed by equal arguments scaled by 4,096.
- A four-argument interaction receives three copies of the producer scaled by 16,384, in argument positions one, two, and four. Position three contains the additional inherited word scaled by 1,024. The subsequent interaction consumes secondary-result state scaled by eight.
- Several equal-argument interactions have their own returned result scaled by four for a later interaction.
- The final inherited-data store uses the usual destination factor of eight but is followed by equal arguments scaled by four.

Other dependency activity includes unchanged primary-result forwarding. Factor-four forwarding is therefore a recurring relationship, not a universal rule.

### Zero stores and address-derived data

Six subsequent stores explicitly write zero. Each uses its own producer, with destination factor eight and following equal-argument factor 16,384.

The six destinations are not established as contiguous or distinct. Dependencies occur between stores. This family is not a demonstrated six-byte clearing operation.

The following byte store derives its data from the destination’s numerical value rather than from inherited data. Its destination is eight times its producer. Where character conversion is unambiguous, the stored character equals that destination value; outside that conversion range, the byte result is **documented_only**.

The following dependency receives, in order, two copies of the producer scaled by 16,384 and the destination value itself.

### Retained byte and historical words

A later unsigned 32-bit result determines a retained byte: the low eight bits of that result scaled by eight. Its first store uses a destination scaled by sixteen. The following dependency receives equal arguments scaled by 32,768.

That byte remains the data source for later stores even when different producers determine their destinations. The relevant destination and argument relationships are:

| Destination basis | Destination factor | Following equal-argument factor |
|---|---:|---:|
| Earlier retained scaled word | 65,536 | 134,217,728 |
| Current producer, at three separated stores | 8,192 | 16,777,216 |
| Further-scaled historical word | 4,096 | 8,388,608 |
| Current producer, before first halfword read | 8,192 | Replaced by base-and-read-value arguments |
| Current producer, at five later stores | 8 | 16,384 |
| Current producer, at the family’s final store | 8 | 4 |

These relationships are not a claim that the dependencies share an implementation.

The historical operand in the first row originates in an earlier inherited-data store group. It already contains that earlier producer scaled by 16,384. It is neither the unscaled producer nor the latest primary result.

A further scaling of that retained word by 32,768 supplies another historical word. This second word later determines both the factor-4,096 destination and the high-byte source used by the three-bit family.

This arithmetic is materially constrained by width. For example, under nonnegative, representable signed arithmetic, the first historical word is a multiple of 16,384. Scaling a nonzero such word by 134,217,728 exceeds the signed 32-bit range. A contract cannot make that case defined by silently widening the calculation.

Retained dataflow is established at the representation level. Physical preservation across unresolved dependencies remains **documented_only**.

### First halfword read and global assignment

The last factor-8,192 retained-byte store is followed by a 16-bit read before any intervening dependency invocation.

Both accesses use the same producer but different address relationships:

- Byte destination: producer scaled by 8,192.
- Halfword address: producer scaled by 4,096, plus 56 bytes.

The receiving dependency gets the factor-4,096 base as its first argument and the read value as its second. The read is not from the byte destination merely because the accesses share a producer.

Later, G-01 is assigned the retained byte immediately before an interaction displaying two zero arguments. This establishes an assignment event, not a terminal invariant. The immediately following dependency, or any later dependency, may modify G-01.

### Extracted data and three-bit data

After the retained-byte family, one store derives character data from a retained unsigned result. The pre-conversion quantity consists of original bits 19 through 28, positioned at the low end, and can range from zero through 1,023.

If the environment’s character conversion retains low bits, the resulting byte selects original bits 19 through 26. That low-bit conversion rule is not independently established. Inputs within the character type’s representable range avoid this uncertainty.

The destination is eight times a separate producer. The following dependency receives that producer scaled by four twice.

The remaining byte-store family uses a different historical source. The high byte of the separately retained scaled word is interpreted as an unsigned eight-bit value. Discarding its lowest five bits yields a value from zero through seven, corresponding to original word bits 29 through 31.

Early stores obtain this reduction as needed. Later stores use retained byte-valued data. An eventual state update replaces the original high byte with its three-bit reduction. Subsequent stores use that replacement directly: they do not reduce it again.

Most destinations are eight times a current producer and precede equal arguments scaled by 16,384. Exceptions include:

- A destination dependent on unresolved full-width shift semantics.
- A destination scaled by 256 from secondary-result address state.
- A destination scaled by 2,048 from a retained secondary-result word.
- Stores followed by interactions with no displayed arguments.
- A store immediately preceding the second halfword read.
- A store immediately preceding the delayed word store.

### Secondary-result address and retained secondary word

One retained secondary-result address participates in three separated direct effects:

| Effect | Address relationship |
|---|---|
| Three-bit byte store | Address value scaled by 256 |
| Second halfword read | Address value scaled by 128, plus 32 bytes |
| Delayed 32-bit store | Retained address itself |

The factor-256 byte store precedes an interaction receiving two copies of the address value scaled by 524,288.

A separate secondary-result word is retained after that store group. It supplies the factor-2,048 byte destination and equal dependency arguments scaled by 4,194,304. Four later two-argument interactions retain that word as their second argument while their first argument comes from a current producer scaled by four.

Later replacement of this retained state must not be moved earlier.

Seven separate interactions forward secondary-result words unchanged as their sole displayed argument. These are distinct from the earlier secondary-result forwarding scaled by eight, the retained address, and the retained secondary word.

The second halfword read immediately follows a current-producer byte store, without an intervening dependency. Its address uses the older retained address, not that current producer. The receiving dependency gets the halfword value as its sole displayed argument.

### Multiword arguments

The four apparent 64-bit dependency results have distinct downstream argument placements:

| Result occurrence | Ordered 32-bit arguments |
|---|---|
| First | Lower portion, upper portion, lower portion |
| Second | Lower portion, upper portion, upper portion |
| Third | Lower portion, upper portion, upper portion |
| Fourth | Lower portion, upper portion |

Repeated portions are required arguments, not redundant documentation. These arrangements establish no pointer pair, ownership transfer, or serialized layout.

### Unresolved address and delayed word store

One three-bit store depends on an apparently 32-bit quantity being shifted right by 32 after factor-four scaling. Its numerical destination is **documented_only**. Conditional on an externally established intermediate value, the downstream destination uses twice that value, and the following equal arguments use 4,096 times that value.

Near the end, a new producer determines a retained word equal to four times its result. A three-bit byte store at eight times that producer precedes a 32-bit store of the retained word through the older retained secondary-result address. No dependency separates those stores.

The next dependency receives three copies of the new retained word. Subsequent activity includes forwarding another primary result twice and supplying the retained word as the second argument of a two-argument interaction. Further byte stores and dependency activity occur before termination.

### Termination

The final represented action invokes F-U05 with immediate zero and a fixed execution-location operand, then transfers control to the code target it produces. The ending is marked non-returning.

No process exit, reset, signal, exception class, diagnostic, or exit status is established.

## Side effects

Direct accesses may alias one another, G-01, or dependency-visible memory. Distinct numerical formulas do not prove disjoint storage. Later overlapping stores may replace earlier data; byte-level effects of an overlapping word store depend on endianness.

No allocation, release, borrowing, transfer, or retention-rights contract is visible. An address-bearing argument does not establish ownership. Unknown dependencies may retain addresses, modify globals, or invalidate storage.

There is no visible rollback, retry, restoration, error return, or recovery branch. Interrupted execution may leave a prefix of the effects already applied.

No locking, atomicity, barriers, reentrancy, or thread-safety guarantee is established. Represented order alone does not specify inter-thread visibility or device-memory semantics.

## Dependencies

The IDs below are opaque document-local dependency labels. F-U02 denotes a category of occurrences, not one proven common implementation.

| ID | Established role |
|---|---|
| F-U01 | Opening indirect interaction selected from inherited return-link state. |
| F-U02 | Ordinary indirect interactions selected from occurrence-specific fixed execution references. |
| F-U03 | Early exceptional interaction using a 1,026-byte displacement; accepts a factor-four argument and exposes the first apparent 64-bit result. |
| F-U04 | Exceptional interaction selected using four-byte alignment followed by a 1,536-byte displacement; its result supplies an inherited-data destination. |
| F-U05 | Terminal mechanism receiving zero and a fixed execution-location operand and producing a code target. |

Complete interfaces, hidden inputs, preservation rules, ownership, side effects, error behavior, and actual return behavior are **documented_only**.

## Acceptance examples

These are conditional static examples, not observed executions. Each assumes its relevant point is reached, its stated data is available, and required arithmetic and accesses are supported. Local examples do not prove a complete preceding execution producing that state.

### Store-before-interaction

For an ordinary inherited-data group with producer 3 and inherited byte 165, address 24 receives byte 165 before the dependency receives 49,152 twice. Matching eventual memory alone does not satisfy the ordering requirement.

### Zero destination

A zero-store producer of zero attempts to store zero at numerical address zero. No visible check suppresses the access. Whether execution continues depends on the environment.

### Unambiguous address-derived character

For producer 9, the address-derived destination and character value are both 72. The following arguments are 147,456, 147,456, and 72.

This example avoids out-of-range character conversion. A destination of 264 would not do so merely because its low byte is 8.

### Unsigned narrowing

For unsigned result 33, the retained byte is 8. Its first destination is 528, followed by equal arguments of 1,081,344. Different later producers do not by themselves change that retained byte.

### Historical operand selection

If the earlier retained scaled word is zero while a newer primary result is one, the historical factor-65,536 destination remains zero, with equal arguments zero. Substituting the newer result would change the behavior.

A nonzero historical example additionally requires resolution of the signed-width issues; mathematical multiplication alone is insufficient.

### First read ordering

For producer 2, the retained-byte store targets 16,384 and precedes a halfword read at 8,248. If that read supplies 4,660, the receiving arguments are 8,192 and 4,660.

### Multiword placement

For lower portion 17 and upper portion 34, the four argument arrangements are respectively:

- 17, 34, 17.
- 17, 34, 34.
- 17, 34, 34.
- 17, 34.

### Three-bit stability

At a local state with retained high byte 96, the three-bit data value is 3. After replacement of the high-byte state by its reduction, subsequent stores still use 3. Reducing the replacement again would incorrectly produce zero.

### Second read and aliasing

For retained address value 4, the second halfword read targets 544. The receiving dependency gets the read value, not 544 or a current primary result.

For late producer 5 and retained address 40, the byte store at 40 precedes a word store of 20 at the same address. The earlier byte need not survive. The following dependency receives 20 three times.

### Unresolved cases

No numerical acceptance result is assigned to the full-width right shift. Likewise, an extracted pre-conversion character value outside the target character range has no unconditional byte expectation.

## Uncertainties

- **documented_only — execution fidelity:** no executable trace validates the supplied representation.
- **documented_only — interface:** inherited-state provenance, complete calling conventions, secondary-result definitions, and physical preservation are unavailable.
- **documented_only — arithmetic:** signed overflow, negative signed shifts, address conversion, out-of-range character conversion, and the full-width shift remain unresolved.
- **documented_only — dependencies:** implementations, hidden effects, memory lifetime changes, ownership, and failures are unknown.
- **documented_only — memory:** mapping, permissions, alignment, endianness, atomicity, and device semantics are unavailable.
- **documented_only — G-01:** declaration, physical write width, and application meaning are unknown.
- **documented_only — termination:** the non-returning annotation does not identify an external terminal event.
- **documented_only — omitted material:** a supplied comment reports removed unreachable material; neither the omitted content nor its reachability is independently verified.
- **Inferred intent only:** repetition may suggest generated interaction behavior. This supplies no additional requirement.

## Evidence

The supplied static representation is the sole basis for the established relationships: access widths, data dependencies, argument ordering, historical reuse, direct-access order, global assignment, and represented terminal transfer.

Embedded comments, earlier prose, and reviewer feedback are not independent behavioral proof. Dependency bodies, a memory map, complete execution semantics, and runtime observations were not supplied.

No tools or runtime tests were used. The acceptance examples establish conditional arithmetic and ordering obligations only.
