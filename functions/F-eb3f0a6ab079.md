<!-- Trusted provenance: F-eb3f0a6ab079; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-eb3f0a6ab079

## Purpose

F-eb3f0a6ab079 describes an externally stateful operation consisting of ordered dependency invocations, computed-address reads, and mixed-width memory writes. Its application-level purpose is not established.

This is a **partial behavioral contract supported by static evidence**, not a claim that the supplied representation is a complete executable function. The representation reports invalid instruction data, removal of an unreachable region, overlapping global symbols, and truncated terminal control flow.

The observable behavior includes selector-directed byte writes, computed word stores, absolute low-memory accesses, and repeated accesses to the same location. These effects do not establish allocation, serialization, initialization, destruction, or successful completion. An encoding or state-manipulation purpose is **[INFERENCE]**, not an acceptance requirement.

## Interface

### Explicit inputs

The represented interface has four 32-bit arguments.

| Input | Established use |
|---|---|
| First argument | Opaque 32-bit value; no explicit use appears in the supplied body. |
| Second argument | Opaque 32-bit value supplied twice to the first indirect invocation. |
| Third argument | Unsigned 32-bit value supplied to the first invocation and later shifted right by 10 to obtain an absolute memory address. |
| Fourth argument | Signed 32-bit value whose represented left shift by three supplies the fourth argument of the first invocation. |

The first invocation receives, in argument-position order, the second argument, the second argument again, the third argument, and the shifted fourth argument. No explicit validation precedes it.

A portable arithmetic interpretation of the fourth argument’s shift is not established for negative values or values whose shift exceeds the signed range. The underlying instruction semantics are **documented_only**.

### Implicit inputs

Execution also depends on:

- Incoming control state that determines the first indirect target.
- Two incoming machine-state values used by the first represented byte update.
- Contents of memory reached through argument-derived and dependency-derived addresses.
- The byte at absolute address 0x01.
- The word at absolute address 0x28, with three distinct represented read occurrences in one later sequence.
- An unnamed global word, with two distinct represented read occurrences.
- Dependency results, including secondary machine-result components whose calling convention is unavailable.

The word at address **0x7c is not an established implicit input**. Its explicit named-global access is a 32-bit store. A computed access could alias that address, but such an alias is conditional on actual address values, not independently established.

### Outputs and completion

No ordinary return value is declared. The supplied representation has no established normal return; its endpoint is an invalid-instruction termination marker.

Supported output channels are memory updates and dependency effects. Normal completion, earlier faults, other transfers of control, and continuation beyond the represented endpoint remain **documented_only**.

### Preconditions and ownership

For a reached operation to execute as represented:

- Its dependency target must be executable under the actual calling convention.
- Its read addresses must support the indicated access widths.
- Its write addresses must support the indicated access widths.
- The environment must permit all reached absolute low-memory accesses.
- Alignment and address-formation requirements must be satisfied.

No buffer capacity, object boundary, nullability rule, ownership transfer, allocation, retention, or release obligation is established. An address passed to a dependency is not thereby established as owned, retained, or consumed by that dependency.

## Behavior

### Widths and arithmetic

Bytes, halfwords, and words are respectively 8, 16, and 32 bits. Some dependency results are represented as 64-bit quantities whose low and high 32-bit components are consumed separately. This establishes the represented data relationships, not a verified high-level return convention.

Unsigned right shifts are logical. Byte observations used as selectors are zero-extended to values from 0 through 255. Adding 0x11 to such a selector produces a destination from 0x11 through 0x110 inclusive. This operation does not read a full-width pointer from the selector location.

Byte stores retain the low eight bits; the halfword store retains the low sixteen bits. Explicit unsigned 32-bit operations wrap at 32 bits. Signed multiplication, subtraction, and left shifts do not acquire portable wraparound semantics merely because the representation uses them to form addresses. Their actual machine behavior remains **documented_only** where relevant.

Masks and operand widths must be preserved. In particular:

- A 30-bit mask followed by a right shift of 20 leaves at most ten bits before byte truncation.
- Under 32-bit bit-vector semantics, shifting right by 9 and then left by 25 retains original bits 9 through 15.
- A represented 64-bit right shift by 54 consumes bits from the full result, not just its low word.

No alignment adjustment, bounds check, or alternative object-relative address is established.

### Entry and argument-derived observations

Several dependency invocations precede the first represented memory write.

That first write reads a selector byte from an incoming machine-state-derived address plus one. It writes one byte at the selector plus 0x11. The stored value is the low byte of another incoming unsigned machine-state value shifted right by 22.

Later, the third argument shifted right by 10 supplies an absolute base. The representation then:

- Reads a 32-bit word at that base.
- Reads another 32-bit word at base plus eight.
- Passes base plus twenty to an indirect dependency, together with the low and high components of a preceding result and a repeated low component.

The first word subsequently determines a selector location by multiplication by 0x400 and addition of one. An early selector-directed store uses the low byte of the word observed at base plus eight; a later store uses that captured word shifted right by 22.

The selector location is read again for the later store. Intervening dependencies prevent assuming that its contents remain unchanged.

No pointer, page-number, handle, or length interpretation of the third argument is established.

### Recurring access families

**Selector-directed writes.** A byte read supplies a selector; one byte is written at selector plus 0x11. Stored values include zero, truncated shifted results, and truncated previously observed words.

Selector-read locations use distinct calculations: an unscaled value plus one; values scaled by four, eight, sixteen, 0x40, 0x400, or 0x1000 plus one; and shifted or masked values plus one. Additional large scaling factors occur. These patterns do not establish one common record layout.

**Computed word slots.** Word destinations include four times a value plus 0x14, sixteen times a value plus 0x14, other computed bases plus 0x14, and computed bases plus 0x34. Values stored include zero, dependency-produced values, scaled addresses, and shifted captured words.

A write at sixteen times a value plus 0x14 does not establish the contents of four times that value plus 0x14. The representation sometimes writes through the former calculation and then reads through the latter. Equality or overlap requires an actual address calculation.

**Other byte stores.** Not every byte store belongs to the selector family. One uses the high component of a represented result as a base plus 0x1c and stores the result’s low byte. Another later writes a result byte directly at a derived address without adding 0x11.

Unknown dependency effects remain interleaved with these accesses. Neither memory preservation nor independence of computed locations is established.

### Principal ordering landmarks

The following landmarks constrain the represented sequence; they do not enumerate every intervening invocation:

1. Entry invocations, the implicit-state byte update, and the argument-derived word reads.
2. Selector updates using those captured words, followed by computed word-slot activity.
3. F-23325cd5e50c and further selector updates.
4. A word store to absolute address 0x7c.
5. Further selector activity, the address-0x28 observation sequence, and a computed store of its saved word.
6. Dependency-derived word reads at offsets including 4, 8, and 0x0c.
7. F-bbb54b5f036d, an unidentified direct-target dependency, and F-f80b3ac065fc, with indirect calls and writes between them.
8. The result-based byte store at high-component base plus 0x1c.
9. An indirect invocation with two references to the byte at 0x01.
10. A 32-bit store to address zero and, later, a 16-bit store to address zero.
11. Further byte activity, F-de15789688a8, another unidentified direct-target dependency, and F-ea83f9a17404.
12. Offset-0x34 write pairs and subsequent scaled-slot activity.
13. Two reads of the unnamed global, followed by further calls and selector updates.
14. F-674e75a953e8.
15. Further selector and word-slot activity before truncated control flow.

The more specific access relationships below are part of this ordering contract.

### Address 0x7c: established output

An indirect dependency produces a 32-bit value that is stored at absolute address 0x7c. The subsequent indirect invocation receives that saved result twice.

No explicit named-global read at 0x7c appears. Reading its entry contents is therefore not a prerequisite established by this access. Computed-address aliases and dependency-internal accesses remain unknown.

### Address 0x28: capture and separate argument references

The representation contains three word-read occurrences:

- A first read captures a word for a later computed store.
- The next indirect invocation contains two further references to the word at 0x28, one for each of its two arguments.

The argument references are not established uses of the earlier capture. Their relative evaluation order is unspecified by the supplied evidence. The contract does not require all three observations to agree.

After that invocation and another indirect invocation, a computed word store writes the **first captured value** to four times a secondary dependency-produced value plus 0x14.

The established distinction is between saved data and fresh represented references. Exact machine-load scheduling, external mutation, and any access-sensitive memory behavior remain unverified.

### Address 0x01: repeated references and later overlap

The byte at 0x01 is repeatedly observed as a selector, including through calculations whose current scalar inputs reduce the selector location to that address.

After the high-component-base-plus-0x1c byte store, and before the absolute-zero word store, an indirect invocation contains **two references to the byte at 0x01**. These are separate represented argument-read occurrences, not an established single captured observation supplied twice.

No relative evaluation order between those two references is established. They must not silently be replaced in this contract with a cached byte. Equal observations are compatible with unchanged ordinary memory; differing observations require some actual mechanism for change or access-sensitive behavior, which is not itself established.

### Absolute-zero stores

A scalar explicitly set to zero remains zero in the represented dataflow when it determines a word-store destination by multiplication by eight. The destination is address zero.

Later, a masked right shift of that still-zero scalar determines a halfword-store destination. This destination is also address zero.

The word store covers byte addresses 0x00 through 0x03. The halfword store covers byte addresses 0x00 and 0x01. Thus, the halfword store overlaps the first two addressed bytes of the earlier word store **regardless of byte order**. Endianness determines which numerical bits occupy those bytes, not the physical extent of overlap.

Both stores include address 0x01. Selector reads after either store depend on the then-current byte there, including any intervening dependency effects. The halfword store does not independently replace the entire earlier word.

### Repeated offset-0x34 stores

After F-ea83f9a17404 and intervening selector activity, two successive word stores target the same computed address at offset 0x34. The first stores a captured selector; the second stores twice that selector.

A later pair writes the same shifted dependency result twice to another computed address at offset 0x34.

Both events in each pair belong to the represented sequence. Final ordinary-memory equivalence does not establish that one event can be omitted from an access-observing contract. Device-register behavior is possible but not established.

### Unnamed global: two ordered reads

After those offset-0x34 pairs and subsequent scaled-slot and selector activity, but before F-674e75a953e8, the unnamed global word is read twice:

- The first represented read supplies a saved value.
- The second supplies a value multiplied by four, producing a separately saved scaled base.

These are distinct observations. The scaled base is not established as four times the first captured observation.

Subsequent dependency activity uses the scaled base plus one as a selector-read location. That location is read on two represented occasions separated by calls. A later invocation also receives a masked-and-shifted form of the first saved global observation.

Further along this interval, an indirect invocation contains two word-read references at a derived base plus 0x1c. Their relative argument-evaluation order is not established. This is another repeated-reference site, not proof of one cached observation.

F-674e75a953e8 occurs only after this activity.

### Errors and terminal behavior

No explicit validation branch, application-level error result, rollback, or recovery protocol appears.

An invalid address or target has no specified failure return. Earlier writes and dependency effects may already have occurred. Atomicity is not established.

The terminal invalid-instruction marker supports a limitation in recovered control flow; it does not establish an intentional successful trap.

## Side effects

Established effect categories are:

- Ordered dependency invocations.
- Absolute low-memory reads and writes.
- Selector-directed and other computed byte stores.
- Computed 32-bit stores.
- A 16-bit store to address zero.
- Argument-derived and dependency-derived word reads.
- Three represented word-read occurrences at 0x28.
- Two represented reads of the unnamed global.
- Repeated references to the byte at 0x01, including two arguments of one invocation.

The explicit access at 0x7c is an output store, not an established read.

Allocation, release, reference-count modification, resource cleanup, and ownership transfer are not independently established. Dependency effects may change globals or addressed storage; preservation cannot be assumed.

Reentrancy and thread safety are **documented_only**. Shared low-memory locations and overlapping accesses prevent assuming isolated state.

## Dependencies

The supplied identity mapping establishes these direct dependencies in the following relative order:

| Opaque function ID | Represented arguments |
|---|---|
| F-23325cd5e50c | One dependency-produced 32-bit value. |
| F-bbb54b5f036d | One saved 32-bit value supplied twice. |
| F-f80b3ac065fc | Low component, high component, low component, low component of a preceding represented result. |
| F-de15789688a8 | One saved 32-bit value supplied twice. |
| F-ea83f9a17404 | One saved 32-bit value supplied twice. |
| F-674e75a953e8 | One saved 32-bit value supplied twice. |

Numerous indirect dependencies and two additional direct-target dependencies lack supplied opaque identities. Their identities and contracts remain **documented_only**; no identifiers are invented for them.

No dependency has an established memory-preservation, ownership, allocation, failure, or termination contract. Use of secondary machine results depends on an unresolved calling convention.

## Acceptance examples

These examples are conditional static acceptance criteria, not executed tests.

### Argument-derived locations

With third argument 0x00100000, its unsigned right shift by 10 yields 0x400. If execution reaches the relevant operations, the word reads are at 0x400 and 0x408, and the interior address passed onward is 0x414.

This does not establish that the argument is a pointer.

### Selector width and truncation

A selector observation of 0xa5 yields byte destination 0xb6. A computed stored value of 0x1234 contributes byte 0x34.

A full-width pointer read or word store would not match that operation.

### Address-0x28 observations

Suppose the first word observation at 0x28 is 0x11223344. The later computed store uses 0x11223344 even if the invocation’s separate argument references observe other values.

If an admissible environment supplies different values to those argument references, acceptance must associate them with their argument positions without imposing an unsupported evaluation order. The example does not assert that such changes occur in ordinary stable memory.

### Two unnamed-global observations

Suppose the unnamed global’s first observation is 4 and its second is 5. The saved unscaled value is 4; the scaled base is 0x14, so its selector location is 0x15, provided the represented arithmetic is valid.

Using 0x11 as that selector location would incorrectly substitute the first observation for the second.

### Two byte references in one invocation

If the byte at 0x01 is stable at 0xa5 during both argument references, both arguments observe 0xa5. If the environment supports distinguishable observations, the contract does not force equality or specify which argument is evaluated first.

### Repeated stores

For a selector value of 3, the first offset-0x34 pair writes word 3 and then word 6 to the same computed address.

The later duplicate pair contains two equal word-store events, not one event justified solely by final memory.

### Absolute-zero overlap

Reaching both zero-address stores produces a word write covering addresses 0 through 3 and a later halfword write covering addresses 0 and 1. This overlap holds for either byte order.

Final numerical word contents require byte order, stored values, and intervening effects; they are not specified here.

### Invalid access

If a reached computed address is unmapped, this contract supplies no success result or rollback guarantee. A failure after prior effects is compatible with the evidence; its precise mechanism is unknown.

## Uncertainties

The following remain **documented_only**:

- Application purpose and semantic parameter names.
- Complete calling convention and incoming machine-state requirements.
- Reliability of secondary result-component interpretation.
- Executability around invalid instruction data and omitted control flow.
- Normal return or continuation beyond the endpoint.
- Byte order, alignment rules, and relevant signed-operation semantics.
- Exact machine-load realization of repeated represented references.
- Relative evaluation order of memory references within argument lists.
- Unmapped dependency identities, effects, ownership, and failure behavior.
- Address-space mapping and access-sensitive memory behavior.
- Computed aliases beyond explicitly established overlap.
- Object layouts, capacities, concurrency, and resource lifecycle.

Repeated read occurrences do not prove differing values; proximity does not prove equality. Repetitive structure does not justify replacing the represented effects with an inferred encoding algorithm.

## Evidence

This contract is based on the supplied static representation and opaque dependency mapping. Prior prose and reviewer feedback are not independent evidence.

Established evidence includes declared widths, explicit memory-reference occurrences, offsets, masks, shifts, represented dataflow, dependency ordering, repeated stores, and reported control-flow limitations. Ordering within argument evaluation is not established merely by listing argument positions.

Runtime claims are conditional on the represented path being executable, the relevant accesses being supported, and preceding dependencies completing. No runtime tests, dependency execution, address-space validation, or calling-convention verification have been performed.
