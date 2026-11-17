<!-- Trusted provenance: F-bef21e07df14; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bef21e07df14

## Purpose

F-bef21e07df14 coordinates indirect dependency invocations with an ordered set of memory accesses. The supplied representation contains 62 eight-bit stores, two 16-bit loads, one global assignment of unspecified width, and one 32-bit store through a retained pointer-like dependency output.

No application-level purpose is established. These effects do not, by themselves, identify initialization, reset handling, serialization, buffer clearing, or resource management.

The represented endpoint invokes a control-transfer value obtained through F-dependency-07. An ordinary return, success indication, or structured error result is not established.

This contract uses three evidence categories:

- **Established:** directly supported widths, ordering, argument relationships, and retained-value relationships.
- **[INFERENCE]:** a deduction requiring an explicitly stated execution or arithmetic model.
- **documented_only:** behavior that the supplied evidence does not resolve.

All event requirements are conditional on execution reaching the event. They do not guarantee that earlier dependencies return or that earlier accesses succeed.

## Interface

### Explicit inputs and incoming state

| Input | Width and interpretation | Established contribution |
|---|---|---|
| First | Signed 32-bit quantity | Determines the initial store address and both displayed arguments to F-dependency-01 |
| Second | Unspecified 32-bit quantity | No explicit use in the represented body |
| Third | Unspecified 32-bit quantity | No explicit use in the represented body |
| Fourth | Signed 32-bit quantity | Contributes the third displayed argument to F-dependency-03 |

The body also depends on an incoming eight-bit payload and incoming control-transfer-related state. Neither is established as a member of the four explicit inputs, and neither has an established default.

Absence of explicit use of the second and third inputs does not prove that dependencies cannot observe corresponding machine state. The complete calling convention is **documented_only**.

### Outputs

There is no established ordinary return value or caller-owned output structure. Observable outputs comprise:

- Direct memory reads and writes.
- Arguments supplied to dependencies.
- Effects introduced by dependencies.
- The final represented control transfer.

The retained pointer-like output is not an explicit input. Its later use does not establish allocation, ownership transfer, or lifetime extension.

### Preconditions for progress

A reached access requires suitable address conversion, permissions, alignment where applicable, and sufficient accessible storage for its width. A reached invocation requires a usable target. Later events additionally require preceding dependencies to return and retained addresses to remain valid.

No local validation, null handling, bounds enforcement, lifetime enforcement, or recovery mechanism is established.

All explicitly sized data accesses use eight, 16, or 32 bits. Three dependency results are represented as 64-bit quantities. Pointer width, address-space width, byte order, and the width of the global assignment remain **documented_only**.

The 16-bit loads use byte offsets of 56 and 32 from numerically derived bases. These offsets do not establish structure layouts or object sizes.

## Behavior

### Ordering and arithmetic interpretation

The represented body has no explicit conditional branches, loops, retries, or rollback. Dependencies nevertheless may fault, transfer control elsewhere, or never return.

The groups below describe direct-effect order, not interchangeable batches. Dependencies intervene between most stores. Their invocations cannot be omitted or treated as pure merely because no arguments are displayed.

Unless otherwise qualified, a scaling relationship describes the numeric relationship represented by the body. Many operands are signed 32-bit quantities. Negative values, overflowing multiplication, and overflowing left shifts cannot automatically be assigned mathematical-integer or wrapping semantics.

Unsigned 32-bit operations support low-32-bit arithmetic and the stated byte extractions. Composed signed-address calculations require additional assumptions.

### Direct-write sequence

| Ordered group | Eight-bit stores | Stored value |
|---|---:|---|
| Incoming-payload group | 20 | Incoming eight-bit payload |
| Zero group | 6 | Zero |
| Address-derived group | 1 | Low eight bits of that store’s numeric destination |
| Retained-byte group | 13 | Low eight bits of eight times a retained unsigned 32-bit result |
| Independent-extraction group | 1 | Bits 19–26 of that same unsigned result |
| Three-bit group | 21 | Bits 29–31 of a separately retained 32-bit address word |

Bit positions are numbered from zero. The retained byte ranges from 0 through 248 in multiples of eight. The final repeated payload ranges from 0 through 7. The two payloads are not interchangeable.

The global assignment occurs between the seventh and eighth stores of the retained-byte group. The first 16-bit load follows that group’s seventh store. The second load follows the three-bit group’s fifteenth store. The 32-bit store follows its seventeenth store.

### Incoming-payload writes

The first direct store precedes every represented dependency invocation. Its destination is the numeric address equal to twice the first input. F-dependency-01 then receives two equal displayed arguments, each representing the first input scaled by 4,096.

For the remaining stores, the usual destination is eight times a primary dependency output. The immediately following consumer usually receives two equal arguments, each representing that output scaled by 16,384.

The following exceptions are observable requirements:

- The fourth store uses twice the output of F-dependency-02. Its following consumer receives two equal arguments scaled by 4,096.
- The fifth store is followed by F-dependency-03. Its first, second, and fourth arguments use the destination-producing output scaled by 16,384. Its third argument uses the fourth explicit input scaled by 1,024. The next invocation consumes a secondary output associated with F-dependency-03, scaled by eight.
- The twentieth store retains the usual eightfold destination, but its following consumer receives two equal arguments scaled by four.

The 16,384-scaled value associated with the sixteenth store is retained across later dependency activity. Subsequent primary outputs do not replace it for the later accesses that depend on it.

Other invocations occur throughout this group, including consumers of fourfold-scaled primary outputs and consumers of unchanged outputs. The repeated store pattern does not establish that these intervening boundaries are identical, redundant, or free of side effects.

### Multiword output handling

Three dependency results are represented as 64-bit quantities.

Two occur during the incoming-payload group: one between its seventeenth and eighteenth stores, and one between its eighteenth and nineteenth stores. Each associated consumer receives three ordered 32-bit portions: low word, high word, high word.

The third occurs between the second and third stores of the three-bit group. Its consumer receives low word followed by high word.

These requirements concern argument selection and order, not memory byte order. Whether each result is physically returned as one 64-bit value or as multiple machine outputs is **documented_only**.

### Zero and address-derived writes

Each zero store has a destination determined by a separate primary output scaled by eight. Its following consumer receives two equal arguments scaled by 16,384.

The six stores do not establish a contiguous cleared region. Destinations may repeat or overlap; distinct producer values need not designate distinct valid objects.

The next store also uses an eightfold-scaled primary output as its destination, but its payload is the destination’s low eight bits. Its following consumer receives, in order:

- Two copies of the producer output scaled by 16,384.
- The producer output scaled by eight.

This payload must not be replaced with zero or with the incoming payload.

### Retained byte and retained address state

A later unsigned 32-bit result supplies the retained byte. Only the result’s low five bits affect that byte. The full result remains available for the later independent extraction.

The first seven retained-byte destinations have these ordered relationships:

| Store within group | Destination relationship |
|---|---|
| First | Retained unsigned result scaled by 16 |
| Second | Older retained address value scaled by 65,536 |
| Third through fifth | Three separate fresh primary outputs, each scaled by 8,192 |
| Sixth | Second retained address word scaled by 4,096 |
| Seventh | Another fresh primary output scaled by 8,192 |

The older retained address value is the value retained from the sixteenth incoming-payload store’s argument calculation. The second retained address word is formed from that older value by scaling by 32,768.

The consumers following stores one through six receive equal argument pairs using, respectively, scaling factors of 32,768 for the unsigned result, \(2^{27}\) for the older retained value, \(2^{24}\) for each of the three fresh outputs, and \(2^{23}\) for the second retained word.

**[INFERENCE]** If every relevant signed calculation is interpreted by retaining only its low 32 bits:

- The second retained word equals the original address producer’s value scaled by \(2^{29}\), modulo \(2^{32}\).
- The second store’s destination equals that original value scaled by \(2^{30}\), modulo \(2^{32}\).
- The sixth store’s destination is zero.
- The later three-bit payload equals the original address producer’s low three bits.

These deductions neither establish signed wrapping as the actual execution rule nor make address zero accessible.

### First load and global assignment

The seventh retained-byte store and the first load share a producer output, but use different numeric addresses. The byte destination uses scaling by 8,192; the load uses scaling by 4,096 followed by a byte offset of 56.

The byte store precedes the 16-bit load. F-dependency-04 receives the 4,096-scaled base as its first displayed argument and the loaded quantity as its second. The load occurs before that consumer begins. Signedness and extension into the argument carrier are **documented_only**.

After further dependency activity, the retained byte is assigned to global storage. The immediately following invocation receives two displayed zero arguments.

This specifies an assignment event, not a final global value. Later dependencies may change the global, and unresolved aliasing may connect it to other accesses.

The remaining six retained-byte stores each use a fresh primary output scaled by eight. Their following consumers receive equal arguments scaled by 16,384 for the first five stores and by four for the last. Subsequent one-argument traffic includes a twofold-scaled output; fourfold scaling is not universal.

### Independent extraction and three-bit writes

The independent-extraction store writes bits 19–26 of the retained unsigned result. Its destination comes from a different primary output scaled by eight. Its following consumer receives two copies of the destination-producing output scaled by four.

Although the intermediate extraction retains more than eight bits, narrowing to the stored byte excludes bits 27 and 28. Neither affects this write.

The following 21 stores use the top three bits of the second retained address word. Ordinary destinations use fresh primary outputs scaled by eight, followed by consumers receiving equal arguments scaled by 16,384.

Exceptions within that group are:

| Store | Established distinction |
|---|---|
| Second | Destination depends on an unresolved full-width shift; following equal arguments use scaling by 4,096 |
| Fourth | Destination uses the retained pointer’s numeric representation scaled by 256; following equal arguments use scaling by \(2^{19}\) |
| Ninth, sixteenth, twentieth | Immediately following invocations have no displayed arguments |
| Eleventh | Destination uses a separately retained secondary 32-bit output scaled by 2,048; following equal arguments use scaling by \(2^{22}\) |
| Fifteenth | Followed by the second 16-bit load and F-dependency-05 |
| Seventeenth | Followed by the direct 32-bit store and F-dependency-06 |

For the second store, a fourfold-scaled signed 32-bit primary output is subjected to a right shift by 32 before contributing to the destination. The shift’s meaning is **documented_only**. Neither a zero result nor shift-count masking is established.

The secondary output used by the eleventh store is also passed unchanged as the second argument of four later invocations, each paired with a fresh primary output scaled by four. Other secondary outputs are consumed individually elsewhere. They must not be substituted for primary outputs or conflated with one another.

### Retained-pointer accesses and paired writes

The same retained pointer-like secondary output determines three direct accesses, in this order:

1. An eight-bit store at its numeric representation scaled by 256.
2. A 16-bit load at its numeric representation scaled by 128, plus 32 bytes.
3. A 32-bit store directly through the pointer.

The scaled addresses are not ordinary indexing through the pointer. No encompassing allocation or common object layout is established.

The fifteenth three-bit store precedes the second load. F-dependency-05 receives the loaded 16-bit quantity as its sole displayed argument. Extension of that quantity is unresolved.

At the seventeenth three-bit store, one fresh primary output determines both an eightfold-scaled byte destination and a fourfold-scaled 32-bit value. The byte write precedes the word write through the retained pointer, with no dependency invocation between them.

F-dependency-06 then receives three copies of the stored 32-bit value. That value is retained for another later invocation as its second argument, paired with a fresh primary output scaled by four. Nearby traffic also passes two unchanged copies of another fresh output.

If the two stores overlap, their order remains significant. Final byte contents additionally depend on byte order.

### Endpoint and errors

Ordinary dependency invocations continue after the final byte store. After the last such invocation, F-dependency-07 receives zero and an embedded site-related operand. Its resulting control-transfer value is invoked.

Exception delivery, handler dispatch, termination, and possible continuation are **documented_only**. No ordinary return path is established.

No structured failure result or local cleanup path is represented. An access fault or non-returning dependency may leave only a completed prefix of effects observable.

## Side effects

Direct effects consist of the specified accesses, global assignment, and control transfers. Dependencies may add arbitrary effects not resolved here.

No direct allocation, release, reference counting, ownership acquisition, or cleanup protocol is established. Retaining an output does not prove ownership or keep its pointee alive. Unknown dependencies must not be assumed to preserve globals, preserve pointees, or release memory.

There is no established transaction or rollback. Previously completed writes need not be undone when progress stops.

Access width does not establish atomicity. Synchronization, thread safety, reentrancy, memory-mapped-I/O behavior, and cross-thread visibility guarantees are **documented_only**.

## Dependencies

The following opaque IDs identify distinguished invocation boundaries, not verified external identities:

| ID | Boundary |
|---|---|
| F-dependency-01 | First invocation, following the entry store |
| F-dependency-02 | Exceptional producer selected using alignment-related target calculation |
| F-dependency-03 | Four-argument consumer involving the fourth explicit input |
| F-dependency-04 | Consumer of the first 16-bit load |
| F-dependency-05 | Consumer of the second 16-bit load |
| F-dependency-06 | Consumer following the paired byte and word stores |
| F-dependency-07 | Terminal operation supplying the invoked control-transfer value |

F-dependency-01 is selected from incoming control-transfer state using a displacement of 4,094. Most later ordinary target selections use embedded control-transfer values with that displacement. F-dependency-02 instead uses four-byte alignment followed by a displacement of 1,536, and receives a primary output scaled by four.

These relationships do not identify target implementations. Similar selection patterns do not establish identical callees; different boundary IDs do not prove different underlying functions.

Remaining indirect dependencies are unresolved individually. Complete prototypes, hidden arguments, return conventions, memory effects, ownership rules, and return guarantees are **documented_only**. A boundary with no displayed arguments is not necessarily a complete zero-argument interface.

## Acceptance examples

These are conditional static deductions, not executed tests. Each assumes progress to the relevant event, suitable address interpretation, accessible storage, and returning preceding dependencies.

### Entry effect precedes invocation

With first input 3 and incoming payload 0xA5, the initial write places 0xA5 at numeric address 6. F-dependency-01 subsequently receives two arguments equal to 0x3000. Moving the invocation before the write would violate the established order.

### Retained-byte narrowing

For retained unsigned result 0x3F, the retained byte is 0xF8. Its first destination is 0x3F0, and its following consumer receives two arguments equal to 0x001F8000. The write affects eight bits, not 32.

### Conditional composed addresses

**[INFERENCE]** Under low-32-bit arithmetic, an original older-address producer value of 5 gives an older retained value of 0x00014000 and a second retained word of 0xA0000000. Their corresponding retained-byte destinations are 0x40000000 and zero. The three-bit payload is 5.

Without that arithmetic model, those composed results are not unconditional requirements.

### Independent payloads

A retained unsigned result of 0x07F80000 produces an independently extracted byte of 0xFF. Changing only bit 27 or 28 leaves that byte unchanged. The three-bit group’s payload remains determined by separate retained address state.

### Load timing and global mutation

If a preceding dependency changes a load location, its consumer receives the quantity loaded at the actual access, not an entry-time snapshot.

Likewise, a later dependency may change the global after its assignment. Requiring the global’s final value to equal the retained byte would exceed the evidence.

### Overlapping paired stores

For a paired-store producer value of 5, the byte destination is address 40 and the word value is 20. If the retained pointer also designates address 40, the byte write still precedes the overlapping 32-bit write. F-dependency-06 receives three copies of 20. Final bytes require byte-order information.

### Unresolved shift and interrupted progress

A known producer output does not, alone, resolve the full-width-shift destination. Requiring that destination always to be zero is unsupported.

If the initial destination is inaccessible, later invocations are not guaranteed. No local recovery or structured error return is established.

## Uncertainties

The following remain **documented_only**:

- Application intent and complete valid-entry conditions.
- Hidden incoming state and completeness of the displayed interface.
- Dependency identities, side effects, ownership behavior, and return guarantees.
- Pointer validity across intervening invocations.
- Signed overflow, exceptional shifts, address conversion, and address-space width.
- Alignment, permissions, byte order, aliasing, and object boundaries.
- Extension of the loaded 16-bit quantities.
- Global assignment width, interpretation, and eventual value.
- Exact terminal behavior and possible continuation.
- Atomicity, synchronization, and concurrency guarantees.

No particular application intent is inferred. Repeated values and arithmetic regularity alone do not justify naming the operation or assuming a higher-level data structure.

## Evidence

Static inspection of the supplied representation supports the direct-access counts and widths, phase ordering, distinguished argument relationships, multiword argument selection, and retention of values across dependency boundaries.

The unsigned payload relationships follow from the represented unsigned 32-bit arithmetic and eight-bit narrowing. The composed signed-address examples are explicitly conditional inferences, not established execution semantics.

Embedded names, comments, prior prose, and reviewer labels are not independent evidence of purpose, ownership, validity, or runtime behavior.

No tools, independent implementation access, runtime tests, or observed executions were used. The acceptance examples are static deductions only.
