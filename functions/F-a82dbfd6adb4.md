<!-- Trusted provenance: F-a82dbfd6adb4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a82dbfd6adb4

## Purpose

F-a82dbfd6adb4 invokes an ordered sequence of indirect dependencies, derives numeric values and addresses from their outputs and incoming machine state, performs mixed-width memory mutations, and finishes with another indirect invocation.

No domain-level purpose is established. Repeated offsets, self-address stores, and character-like constants do not establish allocation, initialization, formatting, decoding, or collection management.

This contract distinguishes **established static evidence**, **[INFERENCE]** for interpretations without behavioral guarantees, and **documented_only** for unresolved details. No runtime tests have been performed.

## Interface

### Inputs and outputs

The represented interface has no explicit parameters and no declared result value. Observable outputs include dependency interactions, memory accesses, global assignments, and the terminal control transfer.

Implicit inputs include:

- An incoming return-link value and another incoming register value.
- Dependency results and secondary machine-state outputs.
- Existing memory and undeclared global objects.
- An unresolved terminal target.

The absence of explicit parameters does not establish independence from caller state. Secondary machine-state outputs are not implicitly zero and are not established to follow a conventional return-value interface.

### Widths, conversions, and layout

Explicitly typed memory accesses establish 8-bit byte, 16-bit halfword, and 32-bit word operations. Paired dependency results are represented as 64-bit values, with 32-bit low and high components. Explicit unsigned byte reads are zero-extended when converted to words. Explicit byte and halfword destinations retain the low eight and sixteen bits, respectively.

**These rules do not establish the access widths of undeclared globals.** A conversion after a global reference establishes the converted value, not the width or signedness of the underlying read.

| Global reference or assignment | Established value-level behavior | Unresolved storage behavior |
|---|---|---|
| Read associated with fixed 0x05 | Conversion to unsigned 32 bits | Source width, signedness, and extension |
| Read associated with fixed 0xa9 | Retention in a 32-bit quantity | Source width, signedness, and conversion details |
| Three later references associated with fixed 0x67 | Separate evaluations; the latter two feed explicit 8-bit destinations | Source read width and signedness |
| Publication-base global | Retention in a signed 32-bit quantity | Source access width, declared type, and conversion details |
| Assignments associated with fixed 0x01, 0x15, and 0x67 | Right-hand values explicitly narrowed to eight bits | Destination storage width |
| Assignment associated with fixed 0x21 | Receives a fresh evaluation of the 0x67 global | Both globals’ declared types and assignment conversion |
| Assignments associated with fixed 0x54 and 0x60 | Receive a doubled retained value and numeric 0x20, respectively | Destination storage widths |
| Assignment associated with fixed 0x3a | Receives an expression formed from an eight-bit character conversion, multiplication by eight, and addition of 0x30 | Character signedness and destination conversion/storage width |

These storage details remain **documented_only**. In particular, neither a global’s address-like designation nor a nearby explicit byte operation proves an unsigned byte read.

Addresses derive from represented 32-bit values. Offsets describe access locations, not a verified record layout. Numeric extraction of paired-result components does not establish memory byte order.

Signed 32-bit shifts, products, and address calculations occur throughout. Unbounded arithmetic is not an adequate execution model. Signed overflow, signed right shifts, oversized shift counts, and address wrapping remain **documented_only** unless an example explicitly assumes 32-bit bitvector arithmetic.

### Preconditions and failures

Completion requires every reached dependency to be callable and return, and every reached memory operation to be supported.

No null, alignment, bounds, mapping, lifetime, provenance, or writability checks are represented. Low fixed locations are not established to be invalid, ordinary RAM, or device registers.

There is no explicit error result, rollback, cleanup path, or validation-failure branch. Faults and nonreturning dependencies can leave earlier effects intact.

## Behavior

### Dependency-driven prefix

The prefix is an ordered sequence of indirect invocations with no represented conditional branch or loop. The first target derives from the incoming return-link value plus 0xffe. Subsequent targets derive from embedded code-location values with that displacement.

Calls without represented arguments, calls whose primary results are unused, and calls receiving explicit zeros remain behaviorally significant because dependency effects are unknown.

Argument relationships include:

- Primary words shifted left eight bits.
- Paired-result low words shifted left eight bits, with high words forwarded separately.
- Shifted high words and secondary machine-state outputs.
- Retained values repeated in several argument positions.
- Explicit zeros and shifted retained zeros.
- Masks followed by shifts, including unsigned logical right shifts.
- Upper portions extracted from 64-bit results before further 32-bit operations.

Some retained samples survive many intervening calls. This does not establish that those calls preserve memory or globals. Complete prefix callsite and argument enumeration is **documented_only** in this contract.

### Values entering direct mutation

The following descriptive names identify numeric relationships, not owned objects:

| Name | Relationship |
|---|---|
| Primary base | A retained primary result multiplied by 2²⁵ |
| Shift base | A retained paired-result low word shifted left twenty-five bits |
| Prefix word | Another retained primary result shifted left eight bits |
| Compact index | A later primary result shifted left eight bits, masked with 0x7fff, then logically shifted right six bits |
| Doubled-index base | Twice the compact index |
| Final pair | Last paired result before direct mutation |

The first explicit word store places the shift base at final-pair high word plus 0x14. Shift base plus nine receives that high word’s low byte. The word at 256 times final-pair low word plus 0x54 receives the primary base. Shift base plus nine is then overwritten with the shift base’s low byte.

Primary base plus 0x14 is cleared as a word, then an explicit unsigned byte is sampled at primary base plus five. Doubled-index base plus 0x14 is cleared, and the byte sample is stored, zero-extended, at primary base plus 0x34.

Only afterward is primary base plus 0x14 sampled as the **read-back word**. Another clearing word store follows at doubled-index base plus 0x14. Aliasing and external-memory behavior prevent treating the read-back word as unconditionally zero.

### Early derived mutations

Two consecutive explicit byte stores at doubled-index base plus 0x15 receive the read-back word’s low byte.

The global associated with fixed 0x05 is evaluated and converted to unsigned 32 bits. This converted value is the **global-selected base**, not an established zero-extended byte. The global associated with fixed 0x01 receives an explicitly eight-bit-converted doubled-index base. Global-selected base plus 0x15 receives an explicit byte store of the compact index.

The prefix word supplies an explicit byte store at primary base plus 0x11 and a word store at read-back word plus 0x54. Primary base plus 0x11 is then cleared as a byte. The computed word destination is not inherently fixed 0x54.

The word at global-selected base plus 0x54 is retained as the **retained field word**, then replaced with the read-back word. Doubled-index base plus 0x34 receives the compact index as a word.

A word sampled at primary base plus 0x74 selects a destination whose word at offset 0x54 receives the primary-base seed shifted left twenty-six bits. A separate indirect update evaluates two word reads at doubled-index base plus 0x74: one selects the destination at offset 0x54, and the other supplies the stored value. Their relative evaluation order within that update is **documented_only**.

The retained field word is stored at its own address plus 0x24 twice. Between those stores, the destination selected through primary base plus 0x74 receives the retained field word’s low byte at nine and the prefix word at 0x54. Its byte at nine is written again afterward.

Doubled-index base plus 0x14 next receives three word stores: the primary-base seed shifted left twenty-six bits, then the retained field word twice. Between the first and second stores, the global associated with fixed 0x54 is assigned twice the retained field word. This global assignment is distinct from the earlier computed word store; its destination width is unresolved.

### Quadrupled-base mutations and sampling boundary

Four times the retained field word is the **quadrupled base**; eight times it is the **eightfold value**.

Two explicit byte stores at doubled-index base plus 0x15 receive the quadrupled base’s low byte. Next, the global associated with fixed 0x15 receives an explicitly eight-bit-converted doubled-index base. Only afterward does quadrupled base plus 0x54 receive the prefix word as a word.

Doubled-index base plus 0x34 receives the quadrupled base. A fresh word at doubled-index base plus 0x74 is doubled into the **doubled field value**. Quadrupled base plus 0x54 then receives the quadrupled base itself.

The next important operations are ordered as follows:

- Quadrupled base plus 0x15 receives the eightfold value’s low byte.
- Retained field word plus 0x44 receives the doubled field value as a word.
- An explicit unsigned byte at quadrupled base plus five selects a word at offset 0x74. That word is the **first selected base**.
- First selected base plus 0x44 receives the doubled field value.

Quadrupled base plus 0x15 next receives the quadrupled base’s low byte. Its byte at thirteen receives the first selected base’s low byte; its words at 0x14 and 0x54 receive the quadrupled base.

A word at quadrupled base plus 0x64 selects another mutation base. That base receives the first selected base’s low byte at thirteen and its own address as a word at 0x14.

The following boundary must be preserved:

| Order | Explicit access |
|---|---|
| First | Read a word at quadrupled base plus four, retaining the **byte-source base** |
| Second | Store the first selected base’s low byte at doubled field value plus nine |
| Third | Store the eightfold value as a word at doubled field value plus 0x44 |
| Fourth | Read an unsigned byte at retained byte-source base plus five |

The byte-source base is retained **before** the eightfold-value store. An aliasing write can affect subsequent memory reads without changing that retained address.

After the first byte sample, quadrupled base plus nine receives the doubled field value’s low byte. First selected base plus 0x44 receives the zero-extended sample. Quadrupled base plus 0x14 receives the eightfold value. Doubled field value plus 0x15 receives the byte-source base’s low byte, and doubled field value plus 0x44 receives the quadrupled base.

The unsigned byte at retained byte-source base plus five is then read again. These samples need not agree. Quadrupled base plus 0x15 receives the eightfold value’s low byte before the second sample selects a word at offset 0x74, the **selected word**. Quadrupled base plus 0x15 then receives the byte-source base’s low byte, and its word at 0x54 receives the quadrupled base.

### Constants and retained publication base

Twice the selected word is the **suffix base**; four times it is the **publication value**.

Quadrupled base plus nine receives the suffix base’s low byte, and its word at 0x14 receives the eightfold value again. Numeric 0x3a is stored explicitly as:

- A byte at publication value.
- A halfword at eight times the selected word.
- A word at publication value plus 0x20.

Quadrupled base minus 0x96 is the **adjusted value**; four times it is the **adjusted base**.

Adjusted base plus 0x20 receives four word writes: 0x20, 0x2f, a zero-extended unsigned byte sampled explicitly at adjusted base plus four, and 0x20 again. The sample follows the 0x2f write.

After these writes, the publication-base global is evaluated once and retained in a signed 32-bit quantity as the **publication base**. The source-global read width and conversion remain unresolved. The global associated with fixed 0x60 is then assigned 0x20.

A repeated region contains the following ordered explicit stores:

| Width | Destination | Value |
|---|---|---|
| Word | Eightfold-derived base plus 0x54 | Publication value |
| Byte | Publication base plus 0x19 | Adjusted base’s low byte |
| Byte | Adjusted base plus 0x11 | Publication base’s low byte |
| Word | Publication base plus 0x34 | Eightfold-derived base |

The first group uses the eightfold value as its eightfold-derived base. Later groups use eight times the adjusted value.

Repetition is explicit, not a represented data-dependent loop. Its exact count is **documented_only** here. Repeated accesses cannot be replaced by final-state equivalence. Every publication destination uses the retained global sample; intervening aliases do not cause resampling.

### Suffix observations

The global associated with fixed 0xa9 is evaluated first and retained in a 32-bit quantity as the **fixed suffix sample**. This does not prove a 32-bit source read or unsigned source. An explicit word at publication value plus 0x74 is then retained as the **early suffix sample**.

Next, the global associated with fixed 0x3a receives the character-conversion-derived expression described in the interface. Under an eight-bit destination and modulo arithmetic, its stored byte would be the low byte of eight times the adjusted value plus 0x30; the missing declaration prevents making that storage guarantee unconditional.

Only afterward are explicit words sampled at suffix base plus 0x14 and adjusted base plus four, becoming the **saved suffix field** and **saved adjusted field**. The global assignment can affect these later observations through overlap, but cannot change already retained samples.

Suffix base plus thirteen receives the adjusted base’s low byte. Adjusted base plus eight receives byte 0x6c; its bytes at thirteen and nine receive the publication base’s low byte. Suffix base plus 0x54 receives the suffix base as a word.

The global associated with fixed 0x67 is assigned the fixed suffix sample explicitly narrowed to eight bits. A word at suffix base plus 0x44 is then sampled. A fresh evaluation of the 0x67 global supplies the assignment to the 0x21 global. Neither access width nor the assignment’s final conversion is established by the absent declarations.

The sampled suffix word supplies a byte at adjusted base plus 0x10 and a word at publication base plus 0x54. Its signed-right-shift-derived low byte is stored at suffix base plus one. The shift count is adjusted base’s low eight bits; unsupported-count behavior remains unresolved. Publication base plus 0x54 then receives the adjusted base.

A word at adjusted base plus 0x40 is sampled. Publication base plus 0x74 receives the saved adjusted field; adjusted base plus 0x10 receives the new word; publication base plus 0x44 receives the suffix base.

A second evaluation of the 0x67 global supplies an explicit byte store at adjusted base plus nine. Suffix base plus nine receives the adjusted base’s low byte. Adjusted base plus 0x44 receives the full retained 32-bit fixed suffix sample.

Adjusted base plus 0x54 is read as the **later derived base**. A third evaluation of the 0x67 global supplies another explicit byte store at adjusted base plus nine. These are three separate source-global evaluations, not three established byte reads.

### Terminal updates

Later derived base plus thirteen receives the early suffix sample’s low byte. Suffix base plus 0x54 receives the saved suffix field.

A word at adjusted base plus 0x14 is retained. Suffix base plus seventeen receives the early suffix sample’s low byte. Adjusted base plus nine receives the later derived base’s low byte; its word at 0x44 receives that full base.

A word at suffix base plus four supplies the **terminal source base**. A halfword at twice that base receives 0x65. Only afterward is the word at terminal source base plus 0x64 sampled as the **terminal second argument**.

The retained adjusted-base-plus-0x14 word supplies a byte at later derived base plus seventeen. Later derived base plus 0x54 receives word value 100.

Terminal source base plus 0x74 supplies the **terminal fourth argument**. A subsequent word at suffix base plus 0x64 supplies the **terminal destination base**. Suffix base plus nine receives the terminal source base’s low byte. Terminal destination base plus 0x54 receives the terminal second argument. The final word store places the terminal destination base at twice the terminal second argument.

The terminal invocation receives, in order: numeric 0x72, terminal second argument, saved suffix field, and terminal fourth argument. A return follows in the representation; the precise machine-level transfer mechanism is **documented_only**.

## Side effects

Explicit memory operations mix potentially overlapping 8-, 16-, and 32-bit accesses. Global references add operations whose actual source or destination widths are not fully established.

Computed locations may alias fixed globals, other computed locations, or the publication global’s storage. Consequently, unresolved global widths affect possible overlap, fault boundaries, and later observations—not merely numeric interpretation.

Repeated reads and duplicate stores remain distinct represented events. Byte order and memory semantics determine mixed-width overlap effects. No atomicity, synchronization, concurrency-safety, or reentrancy guarantee is established.

The direct-memory region establishes no allocation, release, reference counting, or ownership transfer. Dependencies may perform such operations, but their contracts are unknown. Numeric bases are not established to identify owned objects or remain valid indefinitely.

## Dependencies

| Opaque function ID | Scope |
|---|---|
| F-unresolved-entry | Initial return-link-derived invocation |
| F-unresolved-prefix | Collective designation for remaining prefix callsites |
| F-unresolved-terminal | Terminal unresolved invocation |

These are provisional references, not verified identities. The collective prefix designation does not imply interchangeable callsites or a single target.

Dependency argument meanings, secondary outputs, memory effects, global preservation, ownership, failure modes, and return behavior are **documented_only**. No dependency is assumed to preserve globals or own or release memory.

## Acceptance examples

These are conditional criteria, not executed tests.

### Explicit widths and compact index

At a paired forwarding site using an eight-bit left shift, low and high words 0x00001234 and 0x55667788 produce arguments 0x00123400 and 0x55667788. An **explicit unsigned byte** sample of 0xfe becomes word value 254. An explicit byte store from 0x12345678 records 0x78.

Under 32-bit bitvector arithmetic, the compact index ranges from zero through 508 in multiples of four; the doubled-index base ranges through 1016 in multiples of eight.

### Undeclared 0x05 source

If the 0x05 global is declared unsigned eight-bit and yields 0xfe, the global-selected base is 0xfe and its computed byte destination is 0x113.

If instead it is a signed eight-bit global yielding −2, conversion to unsigned 32 bits gives 0xfffffffe. With wrapping 32-bit address arithmetic, the destination at offset 0x15 is 0x13.

If a wider unsigned global yields 0x1fe, the destination is 0x213. The supplied representation does not select among these declaration-dependent cases.

### Retention before aliasing

Suppose quadrupled base is 0x1000, retained field word is 0x400, doubled field value is 0xfc0, and eightfold value is 0x2000.

The earlier word store at retained field word plus 0x44 targets 0x444 and stores 0xfc0. If the later sample at quadrupled base plus four yields 0x3000, the retained byte-source base is 0x3000.

The following eightfold-value word store targets 0x1004 and stores 0x2000. Even though it overwrites the sampled base’s storage, the first subsequent byte-source read targets 0x3005, not 0x2005.

### Global observations and destination narrowing

If the three 0x67 global evaluations yield numeric 0x111, 0x222, and 0x333, the latter two explicit byte stores record 0x22 and 0x33. The 0x21 global assignment receives the first evaluation subject to its own unresolved destination conversion; byte value 0x11 is guaranteed only with a suitable eight-bit destination declaration.

If the 0xa9 global is signed eight-bit and yields −1, its retained 32-bit value can be 0xffffffff rather than 255. The later explicit word store uses that retained 32-bit quantity.

If the retained publication base is 0x2000, publication destinations include 0x2019 and 0x2034. Later changes to the source global do not redirect them.

If a prefix dependency does not return, subsequent direct mutations and the terminal invocation are not reached. No rollback is established.

## Uncertainties

The following remain **documented_only**:

- Domain purpose and valid caller context.
- Architecture, byte order, alignment, memory mapping, and address arithmetic.
- Undeclared globals’ widths, signedness, conversion rules, access widths, and volatility.
- Calling convention and secondary-output definitions.
- Signed overflow, signed right shifts, and oversized-shift behavior.
- Dependency identities and contracts.
- Complete prefix invocation enumeration and exact publication repetition count in this contract.
- Object boundaries, lifetimes, ownership, and alias relationships.
- External-memory and concurrency semantics.
- Precise terminal control-transfer behavior.

“Publication” is descriptive, not evidence of synchronization or an object model.

## Evidence

The supplied static representation supports the explicit access widths, retained numeric quantities, dependency relationships, and ordering boundaries described above. Embedded comments, earlier prose, and reviewer feedback are not independent behavioral authority.

Explicit pointer-directed operations establish widths independently of undeclared globals. Conversely, conversion of the 0x05 global to unsigned 32 bits establishes a converted value, not an unsigned eight-bit source read. Retaining the 0xa9 global in a 32-bit quantity does not establish its source access width. The three 0x67 references establish separate evaluations; only their explicitly byte-directed consumers establish eight-bit destination stores.

Established ordering boundaries include the retained base sample before the eightfold-value store, the fixed-0x15 global assignment before the next prefix-word store, publication-global sampling before the 0x60 assignment, and the 0x3a assignment between earlier and later suffix samples.

This is a static behavioral contract, not a hardware execution trace. No runtime tests, dependency execution, or independent machine-level validation have been performed.
