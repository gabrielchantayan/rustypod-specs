<!-- Trusted provenance: F-db42fdbde845; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-db42fdbde845

## Purpose

F-db42fdbde845 performs a fixed visible sequence of dependency invocations, computed-address memory accesses, and global-storage updates. It accepts one explicit 32-bit input and has no explicit return value. Its last represented invocation targets address zero.

No application-level purpose is established. Descriptions such as initialization, allocation, serialization, dispatch, or supervisor service are **documented_only**, not conclusions supported by the visible operations.

**Coverage qualification:** this document is an incomplete behavioral inventory, not an exhaustive call-by-call trace contract. The opening call sequence and selected later sequences are specified precisely; other portions describe their memory effects and important dataflow without enumerating every intervening call, argument, or target calculation. Those omitted operations are visible static facts, not unknown dependency semantics. This document therefore does not authorize an implementation to omit, merge, reorder, or replace them.

Three evidence classes apply:

- **Established:** represented accesses, widths, operands, and sequencing.
- **Inferred execution model:** 32-bit machine arithmetic used to explain address calculations and truncation.
- **documented_only:** unresolved dependency behavior, ABI details, memory-map interpretation, and higher-level intent.

No runtime tests have been performed.

## Interface

### Explicit input and output

The explicit input is an uninterpreted 32-bit value. No accepted range, validation rule, enumeration, or pointer interpretation is established.

The first invocation receives four explicit arguments, in order: the input, the input, zero, and the input.

There is no explicit result. Observable outputs consist of dependency interactions and memory effects. Reaching the textual return requires every preceding operation to complete, including the address-zero invocation.

### Implicit inputs

Execution additionally depends on:

- An incoming control-address value used as the first call-target base.
- An incoming register-carried word published before any direct memory write otherwise described here.
- Dependency results, including represented two-word results.
- Additional register-carried words and a byte pointer observed after calls.
- Existing memory at computed addresses, including absolute address 1.
- Three separately referenced global quantities.
- A stack-relative address exposed to a dependency.

The additional register-carried values are not established as conventional return values of particular dependencies. Their ABI producer-consumer relationships remain **documented_only**.

### Widths and arithmetic

Addresses are byte addresses. Byte, halfword, and word accesses transfer 8, 16, and 32 bits respectively. A represented two-word result is separated into low and high 32-bit components; this does not require a signed 64-bit interpretation.

Explicit conversions from unsigned bytes to words zero-extend. Byte writes retain the low eight bits. Passing an otherwise untyped byte or halfword does not, by itself, establish its ABI extension to an argument register.

Unsigned right shifts insert zeros. Where supported by the operands, calculations below use 32-bit bit-vector arithmetic: multiplication, addition, and left shift retain the low 32 bits. This is an execution-model interpretation, not a portable guarantee for represented signed expressions. Signed overflow, signed left shifts, pointer arithmetic, alignment, and address validity remain qualified.

Offsets such as 0x0d, 0x14, 0x24, 0x34, 0x44, and 0x74 identify accessed locations, not complete structure layouts.

### External preconditions

The body contains no checks establishing accessible memory or executable targets. Successful progress requires compatible calling conventions and valid storage for every reached access. The stack-relative address must designate suitable storage for whatever the receiving dependency actually does.

No behavior is specified for invalid addresses beyond the absence of represented recovery or rollback.

## Behavior

### Sequencing and dependency boundaries

The represented body has no explicit loop or conditional branch. Its visible sequence is fixed, conditional on each preceding operation returning normally.

Dependencies may alter globals, accessible memory, machine state, or control flow. Calls with no explicit arguments are not thereby pure. Repeated reads separated by calls are fresh observations; repeated writes remain separate effects.

Retained scalar values follow the represented dataflow across calls. That retention does not imply preservation of memory addressed by those values.

### Exact opening calls

The following document-local opaque dependency IDs identify call occurrences only. They do not assert verified callee identities or equivalence between targets.

| Occurrence | Target relationship | Explicit arguments and relevant result |
|---|---|---|
| F-D001 | Incoming control-address value plus 0xffe | Input, input, zero, input |
| F-D002 | 0x08836cb3 plus 0xffe | No explicit arguments |
| F-D003 | 0x08836cb5 plus 0xffe | No explicit arguments |
| F-D004 | 0x08836cb7 plus 0xffe | No explicit arguments |
| F-D005 | 0x08836cb9 plus 0xffe | No explicit arguments; returns a word used below |
| F-D006 | 0x08836cbb plus 0xffe | Four times the F-D005 result |
| F-D007 | 0x08836cc7 plus 0xffe | No explicit arguments |

Between F-D005 and F-D006, a word containing the incoming register-carried value is written at four times the F-D005 result plus 0x14. F-D006 receives the scaled component without that offset.

Subsequent targets include code-address anchors plus 0xffe and other explicitly represented offsets. Their complete ordered inventory is outside this document’s coverage. The first target must not be generalized to an unspecified derived address: its known increment is exactly 0xffe.

### Early word publications

The early publications, in order, include:

1. The incoming register-carried word at a four-times-scaled destination plus 0x14.
2. Four times a later result at sixteen times that same result plus 0x14.
3. That retained four-times-scaled value at a separately selected four-times-scaled destination plus 0x14.
4. Another publication of four times a result at sixteen times that result plus 0x14.
5. Zero at another four-times-scaled destination plus 0x14.

Dependency calls intervene. In this portion, some calls receive a scaled result once and others receive it twice. These multiplicities are real invocation operands, not interchangeable descriptions.

The four-times and sixteen-times destination forms must not be conflated.

### Selector-derived low-address writes

A recurring effect reads an unsigned selector byte and then writes a byte at the selector value plus 0x11. Its destination range is therefore **0x11 through 0x110**, inclusive.

Represented selector addresses include:

- A word masked with 0x3fffffff, shifted right by 16, 17, or 7 bits, then incremented.
- An unmasked word shifted right by 9 or 19 bits, then incremented.
- A retained word plus one.
- A scaled word plus one.
- Absolute address 1.

Later occurrences introduce additional masked, shifted, and scaled expressions. This list is not exhaustive.

The written byte may be zero or the low byte of a scaled result, shifted masked value, dependency result, or separately read word. Each selector read precedes its associated write. Unless a selector is explicitly retained, a later occurrence rereads memory even when its address expression is identical.

These operations are distinct from the late byte **data reads** specified below.

### Initial acquisitions and forwarding

A byte is read at a retained dependency result shifted left by three bits. Its unsigned value is passed in all four explicit argument positions of a dependency invocation.

Later represented interactions include:

- An additional register-carried word followed by zero as two arguments.
- A selector-based clear using the current byte at address 1.
- Two copies of a scaled result followed by the earlier unsigned byte shifted left by 31.
- A register-carried unsigned word shifted right by 18.
- Four copies of a dependency-produced word.

Under the 32-bit model, shifting an unsigned byte left by 31 retains only its least significant bit. The result is zero for an even byte and 0x80000000 for an odd byte.

A subsequent two-word result supplies a retained low word and a high word passed onward. A selector-based clear intervenes before that high-word forwarding call.

### Mixed-width acquisition and stack exposure

The middle portion contains distinct accesses:

| Width | Address relationship |
|---|---|
| Word | A retained base plus a high-bit-only offset derived from bit 19 of that base |
| Byte | A register-carried address plus 0x0e |
| Byte | A retained base plus another register-carried offset |
| Halfword | An unsigned word shifted right by 19, plus a previously acquired unsigned byte |

The high-bit-only offset is either zero or 0x80000000 in the 32-bit model. These accesses must not be normalized to one width or interpreted as a common object layout.

One dependency receives the same stack-relative address in all four explicit argument positions. The supplied representation does not establish the storage’s initialization, extent, type, or permissible retention.

Two-word results are split and used independently. One such result leads to a word publication at four times its high component plus 0x14, containing an earlier unsigned word shifted right by 19. A subsequent invocation receives its low component, its high component, and zero, in that order.

### Offset-specific updates and the 256-times-scaled store

A retained address receives two separate byte writes at offset 0x0d:

- First, the low byte of twice that retained address.
- Later, after intervening calls and a word acquisition, the value 4.

The second write does not eliminate the first from the observable sequence.

Let the **retained source result** denote the dependency result used in this stage’s two 0x44-offset reads. The represented accesses include:

- A word read at four times the retained source result plus 0x44.
- A later word read at the retained source result masked with 0x3fffffff, shifted right by 20, plus 0x44.
- A word write at **256 times a newer dependency result plus 0x14**, storing **four times the retained source result**.

Thus the 256-times-scaled destination and its stored value come from different retained results.

The following invocation receives, in order, 64 times the newer result, that same scaled value again, and a high-bit-only expression formed from the retained source result’s masked value shifted right by 17 and then left by 31.

A later word publication stores the newer result masked with 0x03ffffff and shifted right by 13.

### Linked-address and byte-field processing

A later stage clears a word at sixteen times a retained result plus 0x24, then reads a word at sixteen times that result plus 0x74. The read word becomes a retained address.

At that retained address:

- Offset 0x24 receives a dependency result shifted right by 20.
- Offset 0x74 is read as a word and forwarded.
- Offset 1 is read as a selector for a low-address byte write.
- After intervening calls, offset 1 is reread for a separate zero-byte write.

The first selected byte write contains the low byte of another retained result masked with 0x3fffffff and shifted right by 20.

A 32-bit unsigned result shifted right by 20 is at most 0xfff; subsequently shifting that retained scalar right by 21 necessarily produces zero.

The following stages acquire bytes at offset 0x15 from three register-carried addresses. The first two acquired values are forwarded, including repeated forwarding of retained values. The third is forwarded as an unsigned word and subsequently published as a zero-extended word at a computed destination.

### Global clear and later scalar-zero effects

The first named global quantity, called **G-A** here, receives a zero assignment. It is explicitly read twice later, not continuously assumed to remain zero.

Many calls intervene before those reads. A retained scalar initialized to zero remains zero in the represented dataflow, but G-A and memory at address 1 need not remain unchanged.

Subsequent stages include further zero-word publications, scaled-value publications, and selector-based writes using address 1. Several arguments simplify to zero from retained scalar values. Those simplifications do not permit eliminating the receiving calls.

### Distinct fixed-target call and late byte data reads

A distinct fixed-target dependency, **F-D008**, receives the same retained dependency-produced word twice. The body later uses that full retained word as an address source; the call must not be assumed to replace or preserve any unrelated memory.

Three later acquisitions have separate dataflow roles:

1. **First data read:** an unsigned byte is read at that retained word plus one. Its zero-extended value is forwarded to a dependency and retained.
2. **Second data read:** after intervening calls and selector-based writes, another byte is read at the first acquired byte’s numeric value plus one. This targets an address from 1 through 0x100. The acquired byte is forwarded; its argument-register extension is not established.
3. **Third data read:** after further calls, a separate unsigned byte is read at a retained zero plus one, hence address 1. It is forwarded as a zero-extended word and, after another call, published as a zero-extended word at a computed destination.

None is merely a selector acquisition for an immediately associated low-address store. The third read cannot reuse an earlier observation of address 1.

A selector publication derived from the first unsigned byte masked with 0x3fffff and shifted right by 12 writes zero, because that retained byte is at most 255.

### Late links and nested word reads

Later byte writes include zero at a heavily scaled address plus 9 and the low byte of a retained address at that address plus 9. A word at the retained address plus 0x34 is read and forwarded alongside components of a two-word result.

Nested word acquisitions involve offset 0x14. In one sequence, a word is written at four times a register-carried address plus 0x14, then a word is read at the **unscaled** address plus 0x14. Those locations are not interchangeable.

Subsequent acquired words are shifted, used as addresses, and followed by further word publications. G-A’s first explicit read supplies a retained snapshot used in right shifts. For any unsigned 32-bit snapshot, shifting right by 23 and then right by another 23 produces zero, regardless of changes to G-A before that snapshot.

### Later global publication and 0x74 acquisitions

The second named global quantity, **G-B**, receives a dependency-produced word immediately before a call receives that same retained word.

Subsequent operations separately read words at offset 0x74 and write their low bytes at offset 1 of retained or scaled addresses. Two such reads use the same retained base across intervening calls and therefore may obtain different values.

Another 0x74 read preserves the entire word for later publication at separately computed destinations. Calls in this region use different shifts, scales, argument counts, and target offsets. This inventory does not enumerate all of them.

G-A’s second explicit read produces a fresh snapshot. That snapshot later supplies the base for a word read at offset 0x14. The acquired word contributes to two selector-derived byte writes: one containing its low byte, the other containing the low byte of its unsigned value shifted right by 22.

### Final sequence

After the last dependency returns an 8-bit value, the final effects are ordered:

1. Zero is written through a register-carried byte pointer.
2. A separately referenced 24-bit global quantity, **G-C**, is read.
3. G-B receives a 32-bit concatenation: G-C occupies bits 31–8 and the returned byte occupies bits 7–0.
4. A zero halfword is written at a retained word masked with 0x3fffffff and shifted right by seven.
5. **F-D009** is invoked at address zero with three explicit zero arguments.
6. The textual return is reached only if F-D009 returns.

The G-C read occurs **after** the pointer-targeted clear. If their storage overlaps, that clear can affect the concatenation. No earlier snapshot of G-C may be substituted.

## Side effects

### Direct global-access inventory

| Quantity | Explicit represented accesses |
|---|---|
| G-A | One zero assignment; two later reads |
| G-B | Two assignments; no explicit read through this symbol |
| G-C | One 24-bit read for the final concatenation |

These are separately referenced quantities, not necessarily disjoint storage regions. The supplied overlap warning prevents assuming independence between globals, smaller symbols, or computed accesses. Other accesses and dependencies may also affect their storage.

Direct computed-address effects include byte, halfword, and word accesses, including low addresses. No atomicity, synchronization, transaction boundary, or concurrent-access guarantee is established.

### Ownership and errors

No ownership transfer is established. Passing an address does not prove allocation, retention permission, release, or exclusive access. No dependency is assumed to preserve globals or own memory.

The body contains no represented bounds checks, status checks, null checks, rollback, or recovery path. A failed access, nonreturning call, or exceptional transfer may leave earlier effects visible. No structured error result is established.

## Dependencies

F-D001 through F-D009 are document-local opaque occurrence IDs, not published function identities. F-D008 denotes the distinct fixed-target invocation; F-D009 denotes the terminal address-zero invocation.

Other dependency occurrences remain unenumerated in this inventory. Their visible ordering and operands are not thereby unknown or optional.

For all dependencies, semantics, memory effects, ownership behavior, exceptional behavior, and termination remain **documented_only**. Similar argument patterns do not establish equal targets. Additional register-carried values require ABI evidence before they can be attributed to particular callees.

## Acceptance examples

These are conditional criteria, not executed tests.

| Condition | Required observation |
|---|---|
| Input is 0x12345678 | F-D001 receives that value, that value, zero, and that value, in order. |
| Incoming control-address value is 0x1000 | F-D001 targets 0x1ffe under the stated address model. |
| A selector is 0xff | Its associated byte write targets 0x110. |
| A selector is zero | Its associated byte write targets 0x11. |
| A byte write receives 0x1234 | The stored byte is 0x34. |
| The retained source result for the 256-times-scaled publication is 3 and the newer result is 2 | Subject to valid storage, the word at 0x214 receives 12. |
| Calls change the selector between the two offset-0x0d writes | Both offset-0x0d writes still occur; the later one stores 4. |
| The first late data byte is 0xff | The second late data acquisition addresses 0x100, not a selector-derived store destination. |
| A dependency changes address 1 before the third late data acquisition | That acquisition uses the then-current byte and later publishes its zero-extended value. |
| A dependency changes G-A after its clear | Later reads observe current storage; zero cannot be substituted merely from the clear. |
| The final byte pointer aliases the low-addressed byte of G-C | G-C is read after that byte has been cleared; the exact numeric consequence depends on byte order and overlap. |
| G-C is 0x123456 after the pointer clear and the returned byte is 0xab | G-B receives 0x123456ab. |
| Address zero is not executable | Normal completion is not guaranteed; no represented rollback removes preceding effects. |

## Uncertainties

The following remain **documented_only**:

- Application purpose and meaningful input restrictions.
- Complete ABI and implicit register dependencies.
- Dependency semantics and verified identities.
- Native byte order, alignment, memory accessibility, and executable mappings.
- Exact signed-arithmetic and pointer-arithmetic treatment.
- Complete layouts, bounds, and storage overlap.
- Global declaration extents beyond represented access widths.
- Stack-storage initialization, extent, and retention constraints.
- Outcomes of invalid accesses and the address-zero invocation.

Separately, this document intentionally does not enumerate every statically visible call and memory operation. That is a coverage limitation, not unresolved source behavior.

No claim of memory safety, reentrancy, thread safety, idempotence, or guaranteed termination follows.

## Evidence

This contract uses only the supplied static representation of F-db42fdbde845. Prior prose and reviewer feedback are not treated as independent behavioral evidence.

The global inventory follows the represented assignments and reads: G-A is cleared and read twice; G-B is assigned twice without a symbolic read; G-C supplies the final 24-bit operand after the pointer clear.

The late data-read chain, exact first-target increment, and stored operand of the 256-times-scaled publication are directly represented. Width-derived simplifications are distinguished from dependency assumptions.

No runtime execution, tests, dependency inspection, ABI validation, or memory-map validation has been performed.
