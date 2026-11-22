<!-- Trusted provenance: F-ad2bbb83d9b1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ad2bbb83d9b1

## Purpose

F-ad2bbb83d9b1 performs calculated memory accesses interleaved with a fixed, ordered sequence of opaque dependency interactions.

Its directly established data-memory effects are nineteen eight-bit payload writes, one sixteen-bit read, and one replacement of referenced signed-integer storage. The dependency schedule is also part of the behavior: interactions with unused results or no represented explicit arguments cannot be omitted, combined, or reordered.

No application-level protocol, resource lifecycle, transaction, or success condition is established. All requirements below are conditional on execution reaching the relevant event under an admissible execution model.

## Interface

### Incoming values

The declared interface has one explicit signed-integer argument and no declared return value. Three additional incoming context values participate in the represented behavior.

| Value | Established role |
|---|---|
| Explicit signed integer | Determines the first payload destination and F-D01’s two equal arguments |
| Eight-bit payload pattern | Supplies every direct payload write |
| Reference to signed-integer storage | Identifies replacement storage; its numeric conversion determines another payload destination, two equal dependency arguments, and the sixteen-bit read address |
| Continuation-related quantity | Determines F-D01’s invocation target after an address displacement |

The reference, its converted numeric value, and the integer stored through it are distinct. The old stored integer is not directly read to calculate its replacement.

Additional-context delivery and the complete calling convention are **documented_only**.

### Outputs and widths

There is no declared returned value, status result, or recoverable-error result. Observable outputs are memory accesses, dependency interactions, and resulting control effects.

Established widths are:

- Eight bits for each payload write.
- Sixteen bits for the direct read.
- Thirty-two bits for two unchanged primary-result transfers and seven unchanged secondary-result transfers.
- Ordinary signed-integer storage for the replacement and signed-integer calculations.

The ordinary signed-integer width is not independently established as thirty-two bits. Pointer width, address width, integer representation, byte order, alignment, and addressable-unit size remain **documented_only**.

“Primary” and “secondary” identify result-flow roles, not complete calling conventions or physical result locations.

### Preconditions

Each reached memory access requires accessible storage with the relevant width and permission. Each reached interaction requires a usable invocation target and execution context. Required conversions, arithmetic, and calculated-address interpretations must be admissible.

These are temporal requirements. Opaque dependencies may change memory, permissions, lifetime, or execution state; validity at entry does not establish validity later.

No checked alternative is represented for invalid conversion, arithmetic, access, or invocation.

## Behavior

### Direct memory accesses

Number the payload writes W1 through W19 by occurrence. Each stores the same incoming eight-bit pattern.

| Write | Destination |
|---|---|
| W1 | Twice the explicit signed argument |
| W2 | 256 times the numeric conversion of the incoming reference |
| W9 | 2,048 times a retained early signed secondary result |
| All others | Eight times that occurrence’s signed primary selector |

For each of the sixteen primary-selected writes, the selector comes from the immediately preceding dependency interaction. No dependency invocation intervenes between that producer and its write.

The direct-access order is W1–W13, the sixteen-bit read, W14–W15, the referenced-integer replacement, then W16–W19. Dependency interactions occupy the intervals specified below.

There is no represented local conditional branch, repetition, retry, zero-specific bypass, or input-specific early return. This does not guarantee that every event is reached.

### Exact interaction schedule

The following schedule specifies interaction occurrences independently of their unknown implementations.

An interval numbered *i* begins immediately after Wi and ends immediately before the next payload write. Interval 19 instead ends before the terminal construct. Memory events explicitly attached to an interval remain part of its order.

Every ordinary interaction without a separately assigned ID has the opaque occurrence ID **F-D06-i-j**, where *i* is the interval number and *j* is its one-based interaction position within that interval. Positions count named interactions as well. Thus these IDs identify individual occurrences, not an interchangeable dependency family.

The table uses the following behavioral blocks:

| Block | Ordered interaction requirements |
|---|---|
| **Bare(n)** | Exactly *n* successive interactions without represented explicit arguments |
| **Scaled(n)** | Exactly *n* bare interactions, followed by one consumer receiving four times the signed primary result of the last bare interaction |
| **Secondary(4)** | Four bare interactions, followed by one consumer receiving the unchanged thirty-two-bit secondary-result pattern exposed after the fourth |
| **Primary(2, one)** | Two bare interactions, followed by one consumer receiving the second interaction’s unchanged thirty-two-bit primary-result pattern |
| **Primary(2, two)** | As above, but the consumer receives that pattern in two equal visible argument positions |
| **Retained(4)** | Four bare interactions, followed by a consumer whose first argument is four times the fourth interaction’s signed primary result and whose second argument is the retained early signed secondary result |
| **Replacement(4)** | Four bare interactions, followed by a consumer whose first argument is four times the fourth interaction’s signed primary result and whose second argument is the retained replacement integer |
| **Equal** | One interaction receiving two equal arguments associated with the preceding write, as specified below |

A block’s multiplier states the number of consecutive, distinct block occurrences. It does not describe a runtime loop. Blocks and their constituent interactions occur in the displayed order, with no additional dependency interaction between them.

| Interval | Ordered blocks and attached events | Ordinary interactions |
|---|---|---:|
| 1 | Equal, identified as F-D01; 10 × Scaled(4); Bare(4) | 55 |
| 2 | Equal; Bare(2), then capture the early signed secondary result; 2 × Scaled(4); Secondary(4); Scaled(2); 5 × Scaled(4); Bare(4) | 50 |
| 3 | Equal; Scaled(2); 2 × Scaled(4); Secondary(4); Scaled(2); 5 × Scaled(4); Bare(4) | 51 |
| 4 | Equal; 3 × Scaled(4); Secondary(4); Scaled(2); 5 × Scaled(4); Bare(4) | 53 |
| 5 | Equal; 3 × Scaled(4); Secondary(4); Scaled(2); 5 × Scaled(4); Bare(4) | 53 |
| 6 | Equal; Primary(2, one); 9 × Scaled(4); Bare(4) | 53 |
| 7 | Bare(4); 9 × Scaled(4); Bare(4) | 53 |
| 8 | Equal; 3 × Scaled(4); Secondary(4); Scaled(2); 5 × Scaled(4); Bare(4) | 53 |
| 9 | Equal; 4 × Scaled(4); Scaled(3); 2 × Scaled(4); Retained(4); Scaled(2); Scaled(4); Bare(4) | 52 |
| 10 | Equal; 4 × Scaled(4); Scaled(3); 2 × Scaled(4); Retained(4); Scaled(2); Scaled(4); Bare(4) | 52 |
| 11 | Equal; 4 × Scaled(4); Scaled(3); 2 × Scaled(4); Retained(4); Scaled(2); Scaled(4); Bare(4) | 52 |
| 12 | Equal; 4 × Scaled(4); Scaled(3); 2 × Scaled(4); Retained(4); Scaled(2); Scaled(4); Bare(4) | 52 |
| 13 | Sixteen-bit read; F-D02 consuming that observation; 10 × Scaled(4); Bare(4) | 55 |
| 14 | Bare(1); 10 × Scaled(4); Bare(4) | 55 |
| 15 | Referenced-integer replacement; F-D03 receiving the replacement three times; Primary(2, two); 7 × Scaled(4); Scaled(2); Scaled(4); Bare(4) | 51 |
| 16 | Equal; 3 × Scaled(4); Secondary(4); Scaled(2); 2 × Scaled(4); Replacement(4); Scaled(2); Scaled(4); Bare(4) | 51 |
| 17 | Equal; 3 × Scaled(4); Secondary(4); Scaled(2); 5 × Scaled(4); Bare(4) | 53 |
| 18 | Bare(1); Scaled(2); 9 × Scaled(4); Bare(4) | 53 |
| 19 | Equal; 10 × Scaled(4); Bare(4) | 55 |

This establishes 1,002 ordinary interaction occurrences before the terminal construct, conditional on reaching them all.

The final bare interaction in each interval supplies the next write’s primary selector except in intervals 1, 8, and 19:

- W2 uses the converted reference, not interval 1’s final result.
- W9 uses the retained early secondary result, not interval 8’s final result.
- Interval 19 has no following payload write.

A result with no represented local consumer does not make its producer optional. Likewise, a bare interaction can consume hidden context, memory, or globals despite having no represented explicit arguments.

### Equal-argument consumers

The Equal interaction immediately follows the associated payload write.

| Associated writes | Value in both visible argument positions |
|---|---|
| W1 | 4,096 times the explicit argument |
| W2 | 524,288 times the converted reference |
| W9 | 4,194,304 times the retained early signed secondary result |
| W3–W6, W8, W10–W12, W16, W17, W19 | 16,384 times that write’s signed primary selector |

These arguments are calculated from the destination-selecting value, not read back from the written storage.

W7, W14, and W18 instead immediately precede bare interactions. W13 immediately precedes the sixteen-bit read. W15 immediately precedes replacement storage and then F-D03.

### Retained values and unchanged transfers

The early signed secondary result is captured after interval 2’s third interaction: the Equal consumer followed by two bare interactions. Capture precedes the next interaction.

That retained value determines W9’s destination and Equal arguments and supplies the second argument of each Retained(4) consumer in intervals 9–12. Later secondary results do not replace it.

Seven Secondary(4) blocks occur, in intervals 2, 3, 4, 5, 8, 16, and 17. Their transferred patterns remain unchanged; fourfold scaling is not permitted.

The two Primary blocks occur in intervals 6 and 15. They likewise preserve thirty-two-bit patterns unchanged, with one and two visible receiving positions respectively. Pattern preservation alone does not establish a wider receiver’s extension rule or signed interpretation.

These are specific represented value-flow requirements, not assumptions that dependencies preserve arbitrary globals or machine state. Secondary-result production and retention mechanisms remain **documented_only**.

### Late sixteen-bit observation

The read begins at the numeric address equal to 128 times the converted reference, plus 32.

It immediately follows W13, without an intervening dependency invocation. F-D02 consumes the observation before any other interaction in interval 13 and before W14.

The observation is taken then, not at entry. Earlier writes, aliasing, dependencies, or external activity may have changed the region.

The access width is sixteen bits. Observation signedness, receiving argument width, and sign-extension or zero-extension behavior are **documented_only**. No atomic-snapshot or synchronization property is established.

### Referenced-integer replacement

The primary selector for W15 also determines the replacement integer: four times that selector. This multiplication occurs before W15.

The uninterrupted local effect order is:

1. W15’s eight-bit write.
2. Replacement through the incoming reference.
3. F-D03 receiving the replacement in three equal visible positions.

No dependency invocation separates these events. The old stored integer is not directly read; this is replacement, not an established read-modify-write operation.

The replacement value is retained for interval 16’s Replacement(4) consumer. That second argument is not a later reread of storage. A dependency’s subsequent modification of the stored integer does not alter this represented retained value.

### Arithmetic, addressing, and aliasing

Scale descriptions are mathematical relationships conditional on admissible represented operations. They do not establish wrapping, saturation, truncation, or discarded-high-bit behavior.

The first destination uses signed multiplication by two. The replacement uses signed multiplication by four. The read address uses signed multiplication by 128 followed by addition of 32. Other described scales use signed left shifts; the largest shift is twenty-two positions.

Operand widths and signed-shift rules matter independently. Negative operands are not automatically admissible. A valid payload destination does not validate its larger associated arguments. A valid replacement multiplication does not validate W15’s address shift.

The reference-to-signed-integer conversion is execution-model-dependent. The stored integer cannot substitute for that conversion.

No shared allocation, aggregate layout, contiguous buffer, or disjointness requirement is established. Destinations may coincide or overlap the read region or replacement storage, subject to access validity. Such accesses remain distinct events.

W15 may overlap replacement storage, in which case the later integer replacement may overwrite payload bits according to the actual layout. Later writes and dependency effects may modify storage again. No unconditional final-memory image is established.

### Terminal behavior and errors

After interval 19’s final four bare interactions, F-D04 receives zero and a fixed code-location quantity. Its apparent result supplies the invocation target for F-D05, which has no represented explicit arguments.

This ordering is established; its executable control-transfer meaning is **documented_only**. Whether F-D04 traps, terminates, transfers control, or allows F-D05 to be reached is unresolved. Normal return is not established.

There is no represented local validation result, rollback, recovery, retry, or error translation. Invalid operations must not be assigned invented exception types, error codes, wrapped results, or guaranteed effect preservation.

## Side effects

Conditional on reaching every direct access:

| Direct data-memory effect | Count |
|---|---:|
| Eight-bit payload write | 19 |
| Sixteen-bit read | 1 |
| Signed-integer replacement write | 1 |

These counts exclude dependency-internal accesses and are not bounds on total memory effects.

No direct allocation or release is represented. Reference use establishes neither ownership nor exclusivity, ownership transfer, permission to retain the reference, nor responsibility for releasing storage.

Unknown dependencies must not be assumed to preserve lifetime or to allocate, retain, transfer, or release memory.

No atomicity, no-tearing guarantee, synchronization, cross-thread visibility rule, device-memory interpretation, or reentrancy guarantee is established. The locally uninterrupted replacement sequence is not a proven atomic transaction.

## Dependencies

| Opaque ID | Role |
|---|---|
| F-D01 | Interval 1’s first interaction; consumer of two equal explicit-argument-derived values |
| F-D02 | Interval 13’s first interaction; consumer of the late sixteen-bit observation |
| F-D03 | Interval 15’s first interaction; consumer of three equal replacement values |
| F-D04 | Terminal operation with unresolved execution semantics |
| F-D05 | Apparent invocation through F-D04’s result |
| F-D06-i-j | Individual remaining ordinary interaction at interval *i*, position *j* |

The occurrence IDs do not assert distinct implementation identities, nor do repeated block shapes assert a shared implementation.

F-D01’s target depends on the incoming continuation-related quantity with displacement 4,094. Other ordinary indirect interactions use occurrence-specific fixed target anchors with the same displacement. Target formation alone establishes neither executability nor signature compatibility.

Complete signatures, hidden inputs, memory and global effects, preservation obligations, ownership behavior, and return behavior remain **documented_only**.

## Acceptance examples

These are conditional static requirements, not observed executions. Unless examining a boundary, they assume admissible arithmetic, conversions, storage, invocation targets, and continuation.

### First access and complete first interval

For explicit argument 3 and payload pattern 165, W1 stores that pattern at numeric address 6. F-D01 receives 12,288 twice.

F-D01 is followed by ten distinct Scaled(4) blocks and four final bare interactions before W2. In the first block, F-D06-1-5 supplies the signed primary result consumed, fourfold, by F-D06-1-6. Removing any intervening bare interaction violates the schedule even if its result has no local consumer.

For explicit argument zero, the first destination and both F-D01 arguments are zero. There is no zero-specific bypass. Whether address zero supports the write is unresolved.

### Retained early value

If interval 2’s captured early secondary result is 7, W9 targets numeric address 14,336 and its Equal consumer receives 29,360,128 twice.

Each Retained(4) consumer in intervals 9–12 receives 7 in its second position even if later secondary results differ.

### Coincident destinations and different continuations

If W3’s and W7’s selectors both equal 5, both writes target numeric address 40.

W3 immediately precedes a consumer receiving 81,920 twice. W7 immediately precedes a bare interaction. Combining the writes or exchanging their continuations violates the contract.

### Unchanged patterns

If interval 6’s Primary block transfers the thirty-two-bit pattern numerically denoted by 5, its consumer receives that pattern, not 20.

A highest-bit-set pattern must also remain unchanged. Wider numeric interpretation remains unresolved.

### Late observation and replacement

If the reference converts to 256, the sixteen-bit read starts at numeric address 32,800. F-D02 receives the observation taken after W13, not substituted entry-time contents.

For W15 selector 9, its destination is address 72 and its replacement integer is 36. W15 precedes replacement, then F-D03 receives 36 three times. Overlap may cause replacement to overwrite payload bits.

A later storage change does not alter the retained second argument of interval 16’s Replacement(4) consumer: it remains 36.

### Arithmetic and continuation boundaries

Under a hypothetical thirty-two-bit signed model requiring the positive associated argument to fit, explicit argument 524,288 gives mathematical destination 1,048,576 but associated arguments 2,147,483,648. Writable destination storage does not establish valid argument arithmetic.

After W19, the Equal consumer, ten Scaled(4) blocks, and four bare interactions still precede F-D04. W19 is not completion.

If a dependency does not return, subsequent events may remain unreached. No local restoration of earlier effects is represented.

## Uncertainties

The following remain **documented_only**:

- Correspondence between the supplied representation and executable behavior.
- Ordinary integer, pointer, and address widths; representation and conversion rules.
- Arithmetic validity, alignment, byte order, addressable-unit size, and storage extent.
- Additional-context delivery and complete calling conventions.
- Secondary-result production and retention mechanisms.
- Complete dependency identities, hidden inputs, effects, and preservation obligations.
- Sixteen-bit observation interpretation and wider argument extension.
- Permissions, lifetime, ownership, allocation, retention, transfer, and release.
- Concurrency, fault behavior, reentrancy, terminal execution semantics, and normal return.

These uncertainties do not erase the established occurrence schedule or producer-consumer relationships.

**Inferred intent:** the repeated interactions may participate in a larger coordinated operation. No particular protocol, transaction, or resource discipline follows from that inference.

## Evidence

**Established static evidence:** the supplied representation establishes the direct-access inventory, distinguished widths, access scales, ordered dependency occurrences, explicit argument relationships, unchanged-pattern transfers, and retained-value relationships described above.

The schedule accounts for bare interactions as well as interactions whose results are consumed. It distinguishes each fresh primary producer, secondary-pattern transfer, retained early value, and retained replacement from unrelated later results or storage contents.

Static ordering establishes the late read immediately after W13, replacement immediately after W15, and the remaining interactions after W19 before the terminal construct. It does not establish dependency implementations or executable terminal semantics.

Prior prose, feedback, comments, and provenance annotations are not independent behavioral evidence. **No runtime tests have been performed.**
