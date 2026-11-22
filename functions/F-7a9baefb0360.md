<!-- Trusted provenance: F-7a9baefb0360; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7a9baefb0360

## Purpose

F-7a9baefb0360 is a stateful operation with three explicit 32-bit inputs. Its visible behavior consists of interleaved dependency invocations and byte, halfword, and word memory accesses. Inputs, implicit execution state, dependency results, and global reads contribute to subsequent addresses and stored values.

A domain-level purpose is not established. Repeated scaling and bit extraction are consistent with an encoded state representation, but that is **inferred intent**, not a required interpretation. The evidence does not establish an allocator, constructor, serializer, collection operation, or device interface.

This contract distinguishes:

- **Established static evidence:** access widths, operand relationships, and ordering represented in the supplied material.
- **Inferred intent:** possible explanations without acceptance force.
- **documented_only:** behavior that remains unresolved.

The representation reports invalid instruction data and incomplete control flow. Accordingly, each described event is conditional on execution reaching it, preceding calls returning, and preceding accesses completing. Neither successful end-to-end execution nor normal return is established.

## Interface

### Explicit inputs

| Input | Width | Visible use |
|---|---:|---|
| First argument | 32 bits | No explicit use in the visible body. |
| Second argument | 32 bits | Supplied to the first indirect invocation. |
| Third argument | 32-bit unsigned | Logically shifted right by 10 to obtain a memory address. |

The absence of an explicit first-argument use does not establish that unresolved dependencies cannot observe its incoming machine state. The complete calling convention is unknown.

The third argument is not visibly validated. Its lowest 10 bits do not affect the derived address. Its remaining 22 bits form an address directly; there is no subsequent multiplication restoring the original scale.

### Implicit inputs

The operation also depends on:

- Incoming execution context contributing to the first indirect call target.
- Two incoming machine-state values used by an early selector-based byte store.
- Primary and secondary dependency-result channels.
- Global memory at absolute addresses 0x01, 0x28, and 0x7C.
- A further global word whose address is not established here.
- Memory reached through captured words, scaled values, and dependency-produced addresses.

The three explicit arguments therefore do not determine the operation’s effects.

### Output and completion

No return value is declared. Internal dependency results include 32-bit values and combined 64-bit results consumed as low and high 32-bit components.

No ordinary return is visible. The representation ends at an invalid-instruction boundary. Whether actual execution traps, continues through unavailable instructions, or requires a different instruction interpretation is **documented_only**.

No success indicator, error code, completion callback, or designated output buffer is established.

### Preconditions and numeric conventions

For a described execution prefix to complete, executed call targets must be valid, reads must be readable, and writes must be writable at their represented widths. Alignment and address-space rules must be satisfied, but those rules are unavailable. No visible checks enforce these conditions.

The represented widths are:

- Byte: 8 bits.
- Halfword: 16 bits.
- Word and address-bearing scalar: 32 bits.
- Combined result: 64 bits.

Byte stores retain the lowest 8 bits; halfword stores retain the lowest 16 bits. Selector bytes are interpreted as unsigned values. Unsigned right shifts are logical.

Unsigned 32-bit calculations wrap modulo 2³². Some address expressions instead use signed 32-bit intermediates. Overflow and left shifts involving those intermediates cannot safely be assigned ordinary mathematical or portable signed-language semantics. Machine-width wrapping for those expressions is a conditional interpretation, not a verified runtime rule.

Endianness is unknown. Accesses are specified by width and order, not by an assumed byte layout.

## Behavior

### Entry and input-derived accesses

The first visible event is an indirect invocation receiving the second argument. Several further indirect invocations follow. Some consume preceding results; others have no explicit arguments in the representation. An argumentless invocation does not establish independence from machine state or memory.

Before the third argument is used as an address, an early byte mutation:

- Reads an unsigned byte one byte beyond an incoming address-bearing value.
- Adds 0x11 to the selector’s numeric value to obtain the destination.
- Stores the low byte of another incoming 32-bit value logically shifted right by 22.

For this store, truncation retains source bits 22 through 29, not the entire shifted 10-bit result.

The **input region base** is the third argument logically shifted right by 10. The operation reads a word at that base, then a word at base +0x08. A subsequent indirect invocation receives, in argument-position order, the low and high components of an earlier combined result, the address base +0x14, and the low component again.

The first captured word later selects a byte-read address through multiplication by 0x400 and addition of one. That selector chooses a byte destination at selector +0x11. The first associated store uses the low byte of the second captured word.

After intervening dependency activity, the same selector-address expression is evaluated again. Its newly read byte selects the destination for a store derived from the second captured word shifted right by 22. The selector is not represented as cached across the calls.

### Recurring selector-based writes

A recurring access pattern reads one unsigned byte from a computed location and writes one byte at the selector’s value plus 0x11. Such destinations range from 0x11 through 0x110 inclusive.

The selector is not a full-width pointer. No common allocation base is added to these destinations.

Represented selector locations include:

- Absolute address 0x01.
- An address-bearing value plus one.
- Values scaled by 2, 4, 8, 16, 64, 0x400, or 0x1000, then increased by one.
- Values shifted right by 9, 11, 12, or 21, then increased by one.
- Masked and shifted values, then increased by one.
- Values multiplied by large powers of two, then increased by one.

Stored bytes include zero, low bytes of captured words and dependency results, and truncated extracted fields. One later extraction shifts a combined 64-bit result right by 54 before byte truncation, retaining bits 54 through 61.

Each occurrence is a separate represented access. Intervening dependencies may alter selector memory or other reachable state. Neither repeated addresses nor repeated values justify collapsing access events.

### Early word-state mutations

An early word-update group establishes these relationships, with dependency calls separating several events:

- A four-scaled destination plus 0x14 receives zero.
- Another four-scaled destination plus 0x14 receives the value that selected the first destination.
- A sixteen-scaled destination plus 0x14 receives four times a later dependency result.
- A word is then read from the corresponding four-scaled location plus 0x14.
- A later four-scaled destination plus 0x14 receives that captured word logically shifted right by 11.
- Another word is read at an address formed from the captured word shifted right by 22, plus 0x14.

These are address and value relationships, not evidence of conventional object fields or array elements. In particular, the four-scaled and sixteen-scaled addresses must not be treated as interchangeable.

### Global publication at 0x7C

At a later milestone, a dependency-produced word is written to absolute address 0x7C. This write precedes a dependency invocation receiving that captured value in both explicit argument positions.

Subsequent calls consume transformations of the captured value. Their operands are not shown as fresh reads of 0x7C. Dependencies may nevertheless modify 0x7C or any other memory.

### Three represented reads at 0x28

The 0x28 region contains three distinct represented word reads:

1. A word read captures a saved operand.
2. A subsequent dependency invocation independently reads a word at 0x28 for its first explicit argument.
3. The same invocation independently reads a word at 0x28 for its second explicit argument.

The capture precedes evaluation of both argument expressions. The relative evaluation order of the two argument expressions is not established.

The two dependency arguments are not represented as uses of the saved operand. Their values are not guaranteed to equal that operand or each other when access semantics permit changes or read side effects.

After that invocation and another dependency call, a four-scaled destination plus 0x14 receives the saved operand from the first read. This store is not supplied by either argument read or by a new read after the calls.

### Paired results and additional memory layouts

Later dependency results support the following accesses:

| Location or relationship | Width | Represented behavior |
|---|---:|---|
| Input region +0x00, then +0x08 | 32 bits each | Captured in that order before the region-address invocation. |
| Input region +0x14 | Address | Passed onward; the dependency’s access extent is unknown. |
| High-component-derived base +0x08, then +0x0C | 32 bits each | Read in that order from a combined-result-derived base. |
| Captured address-bearing value +0x04 | 32 bits | Read at later milestones. |
| High component of a combined result +0x1C | 8 bits | Receives the low byte of that result. |
| Computed base +0x34 | 32 bits | Receives consecutive stores in two later groups. |

The reads at +0x08 and +0x0C produce separate operands. Later selector-based stores use their captured values; intervening calls do not turn those uses into fresh reads.

### Separate byte-global argument reads

After the combined-result byte store at high-component-derived base +0x1C, and before the later address-zero word store, an indirect invocation uses the byte global at absolute address 0x01 in both explicit argument positions.

These are two independent represented byte reads for arguments. They are not selector reads and are not uses of an earlier cached selector. The relative evaluation order of these argument expressions is unknown.

Both arguments originate from unsigned 8-bit values. The evidence does not establish the complete ABI-level widening convention. Neither equality between the two observed values nor equality with an earlier read of 0x01 is guaranteed.

Dependency activity and additional selector writes intervene between this invocation and the address-zero word store.

### Address-zero and mixed-width stores

A value explicitly set to zero later selects a word destination through multiplication by eight. Consequently, the represented destination is address zero. That destination receives a 32-bit dependency result.

After further calls and selector-based writes, a halfword destination is formed by masking the same zero value and shifting right by 11. This destination is also zero. A 16-bit store writes the low halfword of a subsequent dependency-produced operand.

The halfword event must not be treated as a word replacement. Endianness and intervening effects prevent specifying a final four-byte image at zero.

Later, a captured low result component is masked with 0x3FFFFFFF and shifted right by seven to form a byte-store base. The store address includes four times the still-zero value, so that contribution is zero. The stored byte comes from a subsequent combined result.

No evidence establishes whether low addresses represent ordinary memory, special hardware, invalid mappings, or recovery artifacts.

### Consecutive stores

Two later groups contain back-to-back word stores without an intervening dependency invocation:

- At a destination formed from a captured value shifted right by nine, plus 0x34, the operation stores a zero-extended selector byte and then twice that byte.
- At another dependency-related base +0x34, it stores the same shifted result twice.

The first pair exposes an intermediate value. The second still represents two writes. Their external effects may differ from a single final store.

### Two successive reads of the unnamed global

Farther into the operation, after a scaled word store, the unnamed global word is read in two successive represented statements:

- The first read captures a word used later for masking, shifting, and dependency arguments.
- The second independently reads the same global and multiplies that value by four for subsequent address calculations.

The first read precedes the second. There is no intervening dependency invocation between these statements, but equality is not guaranteed without an access-semantics contract.

Later selector reads use the four-scaled value derived from the second read. A later masked-and-shifted dependency argument uses the first captured value. These operand histories must remain distinct.

### Trailing word and selector activity

The trailing region continues scaled word stores, captured-word transformations, and selector-based byte writes.

Additional repeated memory expressions occur as dependency arguments:

- Two independent word reads at a computed base +0x1C supply two argument positions.
- In another invocation, two independent word reads at a four-scaled address +0x14 supply differently transformed arguments: one shifted right by 12 and another shifted left by two.

No mutual argument-evaluation order is established for either pair. These expressions must not be silently replaced with one shared capture.

Other trailing events explicitly capture a word before calls and later store a shifted version of that capture. Those saved-operand uses are distinct from the repeated argument-read cases.

The last visible region performs another selector-based byte mutation followed by further indirect calls. The representation then ends at invalid instruction data. No normal return, release phase, rollback, or final success publication is established.

## Side effects

Directly represented modifications include:

- Byte destinations between 0x11 and 0x110.
- A word at absolute address 0x7C.
- A word and, later, a halfword at address zero.
- Computed byte and word destinations derived from inputs, globals, implicit state, and dependency results.

Reads can also be externally significant. In particular, the three word reads at 0x28, two byte argument reads at 0x01, two ordered reads of the unnamed global, and later repeated argument reads are distinct represented accesses.

Dependencies may produce additional effects. No supplied dependency contract establishes global preservation, memory preservation, allocation, release, borrowing, retention, or ownership transfer. Passing an address does not establish any of those properties.

The operation is not visibly transactional. A nonreturning call or fault can occur after earlier effects. No rollback is visible.

Thread safety, reentrancy, atomicity, barriers, volatile semantics, and asynchronous observation guarantees are **documented_only**.

## Dependencies

The mapped dependencies occur in this relative order, with substantial unresolved activity between them:

| Opaque function ID | Explicit argument relationship |
|---|---|
| F-23325cd5e50c | One dependency-produced 32-bit value. |
| F-bbb54b5f036d | The same captured 32-bit value twice. |
| F-f80b3ac065fc | Low and high components of a combined result, followed by the low component twice. |
| F-de15789688a8 | The same captured 32-bit value twice. |
| F-ea83f9a17404 | The same captured 32-bit value twice. |
| F-674e75a953e8 | The same captured 32-bit value twice. |

Their semantics, result conventions, error behavior, ownership obligations, and memory effects are unknown.

Numerous indirect invocations and two additional fixed-target invocations lack mapped identities. Their identities remain **documented_only**. No recursive invocation of F-7a9baefb0360 is visible.

Repeated captured arguments in this table must not be confused with independent global-read expressions used by unresolved invocations.

## Acceptance examples

These are conditional static criteria, not executed tests.

### Input-derived address equivalence

Third arguments 0x00100000 and 0x001003FF both yield base 0x400. If reached, the initial region reads occur at 0x400 and 0x408, and the address passed onward is 0x414.

This establishes address equivalence only, not identical overall effects.

### Selector and truncation

A selector value of 0xA5 chooses destination 0xB6. A byte-store operand of 0x1234 writes 0x34.

If an intervening dependency changes a selector source from 0x20 to 0x40, separate reads choose destinations 0x31 and 0x51. Reusing the earlier selector would not preserve the represented accesses.

### Independent 0x28 observations

Suppose the initial capture at 0x28 yields 0x11223344. Suppose the two argument-position reads subsequently yield 0x55667788 and 0x99AABBCC, respectively.

The dependency receives those independently observed argument values. This example does not specify which argument read executes first. If execution reaches the later saved-operand store, its value is 0x11223344, regardless of subsequent changes to 0x28.

### Independent byte-global arguments

If the two argument-position reads at 0x01 yield 0x12 and 0x34, the corresponding invocation receives those byte-derived values. It must not substitute one cached selector for both.

This invocation follows the combined-result byte store at +0x1C and precedes the address-zero word store, without requiring immediate adjacency.

### Distinct unnamed-global reads

If the unnamed global’s first read yields 0x100 and its second yields 0x200, the saved masking-and-shifting operand is 0x100, while the four-scaled address contribution is 0x800.

Using 0x400 as that contribution would incorrectly reuse the first capture.

### Consecutive stores and halfword width

For selector 0x7F, the first consecutive-store destination receives word values 0x0000007F and then 0x000000FE.

At the address-zero halfword event, operand 0x89ABCDEF writes halfword 0xCDEF. Final byte order and surrounding contents remain unspecified.

### Interrupted execution

If a dependency does not return, later direct effects need not occur. Earlier effects need not be undone. No particular error return follows from the evidence.

## Uncertainties

The following remain **documented_only**:

- Domain purpose and valid end-to-end use cases.
- Complete control flow and behavior at the invalid-instruction boundary.
- Complete ABI, implicit inputs, argument widening, and secondary result channels.
- Dependency effects, ownership, and lifetime guarantees.
- Signed-intermediate overflow and shift behavior.
- Endianness, alignment, address mapping, and hardware access semantics.
- Whether repeated expressions correspond exactly to physical bus transactions.
- Equality of independently read values and evaluation order between argument expressions.
- Valid input ranges beyond access-dependent preconditions.
- Failure signaling, cleanup, successful completion, and concurrency guarantees.

The representation establishes capture order at 0x28 and ordered reads of the unnamed global. It does not establish mutual evaluation order within the repeated-argument pairs.

## Evidence

This contract derives solely from the supplied static representation and dependency mapping. The previous draft and reviewer feedback are not independent behavioral authority.

Static evidence supports the stated widths, argument relationships, saved operands, repeated memory expressions, statement ordering, and incomplete ending. It specifically distinguishes three represented reads at 0x28, two successive reads of the unnamed global, and two byte-global argument reads after the +0x1C byte store.

Reported invalid instruction data, removed unreachable material, and overlapping global symbols limit confidence in a complete executable interpretation.

No implementation inspection beyond the supplied material, instruction-level validation, runtime execution, or tests have been performed. The acceptance examples express conditional arithmetic and ordering consequences, not observed runtime results.
