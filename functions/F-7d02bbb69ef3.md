<!-- Trusted provenance: F-7d02bbb69ef3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7d02bbb69ef3

## Purpose

F-7d02bbb69ef3 forwards an opaque 32-bit input to an indirect dependency, performs an ordered sequence of dependency calls and mixed-width memory accesses, updates shared state, and finally attempts a call to address zero with three zero arguments.

**Established static evidence:** the supplied representation exposes one explicit input, no explicit return value, and no explicit conditional branches or loops. Direct accesses include bytes, halfwords, and 32-bit words. Several call results are represented as 64-bit quantities whose low and high 32-bit components have separate uses.

**Inferred intent:** selector-directed writes, scaled-index publications, and linked address calculations suggest a stateful processing protocol. They do not establish an application purpose, object model, or success condition.

Normal completion is conditional on every reached operation completing normally, including the address-zero call. The representation does not establish whether that call faults, transfers control to an environment-provided service, or returns.

## Interface

### Explicit input and output

- **Input:** one opaque 32-bit value, forwarded unchanged to the first dependency.
- **Output:** no explicit return value or status.
- **Interpretation:** the body does not establish whether the input is an address, count, flag set, or signed quantity.

### Implicit inputs and shared storage

Execution also consumes incoming control context, retained register-carried values, secondary call-result state, memory contents, and executable targets. A call with no displayed arguments is not necessarily independent of machine-level inputs.

Distinguishable storage references include:

| Storage | Established use |
|---|---|
| Absolute address 1 | Unsigned-byte selector source, including a late retained capture |
| Absolute address 0x30 | Four displayed 32-bit argument observations in one early call |
| Three-byte source beginning at address 9 | Terminal packed assignment |
| Shared location A | Cleared, then independently captured twice as a 32-bit value |
| Shared location B | Assigned a dependency-derived word, then a packed word |
| Stack storage | Its address is passed in four argument positions |

These labels distinguish represented references, not necessarily disjoint storage. An explicit overlap warning prevents assuming that larger shared locations are independent of smaller symbols.

### Widths, layouts, and arithmetic

- Ordinary words and the explicit input are 32 bits.
- Byte accesses are 8 bits; halfword accesses are 16 bits.
- Represented paired results are 64 bits, split into low and high 32-bit components.
- Explicit unsigned-byte widening yields values from 0 through 255.
- Byte stores retain the low eight bits of their supplied values.
- Right shifts of explicitly unsigned words are logical.
- Unsigned 32-bit calculations remain 32-bit calculations; shifts do not implicitly create wider address intermediates.
- Signed multiplication and left shifts also occur. Their overflow and exceptional cases remain **documented_only**; a universal wrapping rule is not established.
- Some opaque narrow values have established access widths but unspecified signedness or argument-extension rules.

The terminal assignment to B places a numeric three-byte value in bits 31–8 and a dependency-produced byte in bits 7–0. This describes numeric composition, not memory byte order.

Endianness, alignment requirements, low-address accessibility, and the mapping from integer-derived addresses to accessible memory remain unknown.

### Preconditions

Normal completion requires every reached address to support its specified access width, every reached call target to be callable, and every dependency to return normally. No visible validation establishes these conditions.

The explicit input alone cannot determine whether these requirements hold.

## Behavior

### Ordering and retained observations

The visible sequence is straight-line. Later operations are reached only if earlier operations complete normally.

Every dependency invocation is an effect boundary, including calls whose results are discarded. Unknown dependencies may mutate memory and shared state. Repeated reads separated by calls must therefore remain distinct observations.

A retained scalar capture is not a fresh observation of its source. This applies especially to both captures of A and the late saved byte from absolute address 1. Changes to the source after a capture do not replace that captured value.

The following regions describe observable relationships and order. Their grouping does not authorize deleting intervening calls, combining observations, or reordering effects.

### Selector-directed byte writes

A recurring operation reads an unsigned selector byte and stores a byte at the selector’s numeric value plus 0x11.

The destination range is **0x11 through 0x110**, inclusive. The addition is not narrowed to eight bits. The destination is not the selector-source address plus 0x11.

Selector sources include incoming or dependency-derived bases plus 1, shifted or scaled quantities plus 1, and absolute address 1. Stored values include zero and low-byte truncations of retained calculations.

Repeated source expressions ordinarily denote repeated observations. The explicitly retained late selector described below is an exception.

### Initial forwarding and publications

The first dependency receives the explicit input unchanged. Four further calls precede the first direct byte write. That write selects its destination using a byte at an incoming base plus 1 and stores the low byte of another incoming unsigned word shifted right by 22.

Early calls include duplicated fourfold-scaled arguments, a three-argument call ending in zero, and a four-zero-argument call. Another call displays four 32-bit reads from absolute address 0x30. The relative evaluation order and coherence of those four observations are not established.

Two early zero-byte writes independently obtain their selectors from an incoming unsigned quantity shifted right by 9, then offset by 1.

Early word publications include:

- A retained index stored at four times that index plus 0x14.
- The same retained value stored at four times a later result plus 0x14.
- Four times an index stored at sixteen times that index plus 0x14.
- Retained fourfold-scaled values stored at later fourfold-indexed destinations plus 0x14.

These accesses do not establish a common array or record layout.

One unsigned word observation uses a dependency result shifted right by 18, multiplied by 1024, and offset by 0x14. The observed word is shifted right by 19 before forwarding. Another call receives a retained fourfold-scaled value, a word observed at that value plus 0x14 and shifted right by 19, zero, and the retained scaled value again.

### Masked extraction and paired results

A subsequent unsigned result supplies a fourfold-scaled value and three extractions: the result masked to 30 bits, then shifted right by 17, 20, or 7.

Separate selector writes store the low byte of the scaled value and, later, the low byte of the right-by-20 extraction.

A seven-argument call receives four copies of one result, the right-by-17 extraction, the right-by-7 extraction, and a control-address-derived value. Their positions and duplication are established; semantic roles are not.

A following 64-bit result is split. A selector-directed zero write precedes a call receiving its high word twice and a retained argument. Its low word remains available for later extraction and addressing.

### Mixed-width observations

The middle sequence includes these separate observations:

| Address relationship | Observation and subsequent use |
|---|---|
| Twice a retained unsigned result | Unsigned byte, widened and repeated in four argument positions |
| Retained low result plus a high-bit offset | Word, repeated in four positions |
| Secondary-result base plus 0x0e | Byte, repeated in four positions |
| Retained low result plus a signed secondary result | Unsigned byte, widened and repeated in four positions |
| Right-by-19 extraction plus an earlier widened byte | Halfword, repeated in four positions |

Calls and writes separate these observations.

The high-bit offset is formed by shifting an unsigned right-by-19 extraction left by 31 at 32-bit width. It is therefore zero or 0x80000000, not an unrestricted large scaled index.

This region also forwards a top-nibble extraction and passes the same stack-storage address in four argument positions. The storage’s extent, initial contents, and permissible retention period are unresolved. Passing its address does not establish ownership transfer.

After halfword forwarding, paired-result components participate in word publications retaining an earlier extraction.

### Interleaved byte stores and word reads

A retained low component supplies a **byte-update base**. A later unsigned result supplies a **word-read index**.

The observable order is:

1. A byte store at the byte-update base plus 0x0d writes the low byte of twice that base.
2. A word at four times the word-read index plus 0x44 is observed and passed as the sole displayed argument to a dependency.
3. Four additional calls execute; the fourth produces a retained result.
4. A byte store writes literal 4 at the same byte-update destination.
5. Another word is observed at the word-read index masked to 30 bits, shifted right by 20, and offset by 0x44. It is the second argument of a call whose first argument is four times the retained result from the fourth intervening call.

Only the second word observation follows the literal-4 store. Aliasing can make the first observation’s placement between the stores observable.

### Subsequent publications and linked accesses

Later publications use scales including 256, 4, and 16. Published values include fourfold-scaled results and a result masked to 26 bits and shifted right by 13.

A linked-access region writes a zero word at sixteen times a retained index plus 0x24, then reads a word at sixteen times that index plus 0x74 to obtain another base. Calls intervene before a dependency result shifted right by 20 is stored at the new base plus 0x24. A word at the new base plus 0x74 is subsequently observed and forwarded.

That unsigned right-by-20 extraction is at most 4095. Shifting its retained value right by another 21 necessarily yields zero.

Three later unsigned-byte observations use secondary-result bases plus 0x15. Calls and writes separate them. Their values feed dependencies; the third byte is also widened and published as a word. They are not a coherent snapshot.

Further publications include scaled values, masked extractions, and zeros. A is explicitly cleared. Later calls include a distinct fixed-target invocation receiving two copies of one result.

Some byte addressing is indirect through byte values: a byte observed at a retained result plus 1 is saved, and a later byte observation uses that saved byte’s numeric value plus 1.

### Offset-9 writes and offset-0x34 observation before the first A capture

The following operations occur **after A is cleared but before its first post-clear capture**. They therefore also precede the late saved-byte capture from address 1 and the first assignment to B.

First, a zero byte is written at an address formed from a retained fourfold-scaled result multiplied by 0x2000, then offset by 9. The signed arithmetic qualifications above apply.

After intervening calls, a byte is written at another retained base plus 9, using the low byte of that base. A word at this same retained base plus 0x34 is then observed as the third argument of a three-argument call. Its first two arguments derive from the low and high components of a paired result.

Additional calls and word publications follow. Linked word observations include:

- A word at a secondary-result base plus 0x14.
- A word at that captured word shifted right by 22, then offset by 0x14.
- A word at the resulting captured value plus 0x14.

The sequence also publishes a scaled dependency result using half of a captured word as an index. Only after these operations, further calls, and another publication does the first explicit post-clear read of A occur.

### First post-clear capture of A

The earlier clear does **not** establish that this read returns zero: intervening calls and potentially overlapping accesses have already occurred.

This read captures an unsigned 32-bit value. Five calls occur before the captured value is shifted right by 23. A separate retained word extraction is published before a four-argument dependency invocation receives:

- The same secondary-result value in its first two positions.
- The captured A value shifted right by 23 in its **third** position.
- The separate word extraction in its fourth position.

The third argument ranges from 0 through 511. It depends on A at the capture, not at the clear or the later call.

A further right shift by 23 of this already-shifted captured value necessarily yields zero. That zero participates in later addressing and publication; it does not make the earlier third argument universally zero.

### Late saved byte and first assignment to B

A later region captures the unsigned byte at absolute address 1 and immediately stores zero at its numeric value plus 0x11.

This byte remains retained across subsequent calls and the first assignment to B. A dependency produces a 32-bit result; B receives that result before it is forwarded to the next dependency. Another call occurs before the saved byte is reused.

The function then reads a word at **the saved byte plus 0x74** and writes that word’s low byte at **the saved byte plus 1**. Neither address uses a fresh observation of absolute address 1.

The saved value is between 0 and 255, so these later addresses range from 0x74 through 0x173 for the word read and from 1 through 0x100 for the byte store. If the saved value is zero, the byte store itself targets absolute address 1.

### Later updates and second capture of A

Further updates observe words at derived bases plus 0x74 and store their low bytes at those bases plus 1. One retained base is updated twice, with calls between the updates; each update requires its own word observation.

Another observed word is retained and published to additional scaled destinations. Further publications include a destination based on a retained unsigned value with its bottom two bits cleared, then offset by 0x14.

Near the end, A is independently read again. This second capture is not subjected to the earlier right-by-23 transformation.

Calls and another publication intervene before a word is observed at the second captured value plus 0x14. That word supplies truncated values for later selector writes. Mutating A after this capture does not change the retained address basis. The two captures of A need not agree.

### Terminal operations

The terminal region obtains an 8-bit dependency result, then performs these effects in order:

1. Stores a zero byte through a pointer supplied in secondary call-result state.
2. Observes the three-byte source beginning at absolute address 9 and assigns B its numeric value in bits 31–8 combined with the dependency byte in bits 7–0.
3. Stores a zero halfword at an address derived from a retained low 32-bit result masked to 30 bits and shifted right by 7.
4. Attempts the address-zero call with three zero arguments.

The pointer-directed store precedes the three-byte observation. Overlap can therefore affect B’s packed value. The representation does not establish the hardware transaction granularity of that three-byte observation.

The explicit valueless return is reached only if the final call returns normally.

## Side effects

Direct effects include selector-directed byte writes, scaled word publications, offset-based updates, the terminal halfword store, and assignments to A and B. Unknown dependencies may introduce additional effects.

No explicit allocation, release, resource-acquisition protocol, or ownership transfer is established. This does not establish that dependencies preserve memory ownership or refrain from retaining pointers.

No rollback, cleanup path, synchronization, transaction, status reporting, or exception translation is visible. A fault or nonlocal exit may leave earlier effects in place. Fault delivery and partial-access behavior remain unresolved.

## Dependencies

The supplied identity map establishes only **F-7d02bbb69ef3**. Opaque function IDs for its dependencies are unavailable and remain **documented_only**; no substitute identities are asserted.

Dependency roles include an initial context-derived target, numerous control-address-derived targets, a distinct fixed target, and the final address-zero target. Similar target-construction patterns do not establish equivalent identities or contracts.

Unknown contracts include implicit inputs, secondary-result conventions, shared-state mutation, pointer retention, ownership, failure, nontermination, and normal return. No dependency is assumed to preserve globals or own, retain, or release memory.

## Acceptance examples

These are conditional static obligations, not executed tests.

1. **Input forwarding:** input 0x89abcdef reaches the first dependency unchanged.

2. **Selector width:** selector 0xff selects address 0x110, not 0x10. Fourfold scaling of 0x43 supplies stored byte 0x0c.

3. **Interleaved aliasing:** with byte-update base 0x77 and word-read index 0x10, the first byte destination and first word source are both 0x84. Byte 0xee is stored before the word observation. Literal 4 is stored only after that observation, its consuming call, and four further calls. The complete word depends on surrounding bytes and endianness.

4. **Pre-capture chronology:** if execution reaches the first post-clear capture of A, it has already performed both described offset-9 byte writes and the offset-0x34 word observation. Placing those operations after the late saved-byte capture or first assignment to B violates the visible order.

5. **Nonzero first A capture:** if the first capture returns 0x80000000, the identified four-argument call receives 0x100 as its **third argument**. Replacing that third argument with zero violates the captured dependency. Its first two arguments instead repeat a secondary-result value. The further right-by-23 extraction of 0x100 is zero.

6. **Capture survives mutation:** if that first capture is 0x80000000 and a subsequent dependency changes A to zero, the identified third argument remains 0x100. The later independent capture may observe zero.

7. **Saved address-1 byte:** if the late capture is 0x20, the immediate zero store targets 0x31. Even if dependencies change address 1 to 0x40, the later update reads at 0x94 and writes at 0x21, not 0xb4 and 0x41.

8. **Zero saved byte:** a late saved byte of zero produces the later word read at 0x74 and byte store at absolute address 1.

9. **Second A capture:** a second capture of 0x200 supplies later word-source address 0x214, even if a dependency changes A before that word observation.

10. **Terminal packing:** three-byte numeric value 0x123456 and dependency byte 0xab produce B = 0x123456ab, without specifying endianness.

11. **Failure prefix and return:** failure of the terminal halfword store prevents reaching the address-zero call through the visible sequence. Earlier effects are not visibly undone. If every operation, including that call, returns normally, execution reaches the valueless return.

## Uncertainties

The following remain **documented_only**:

- Application purpose, valid input domain, and complete calling convention.
- Origins and validity of implicit incoming state.
- Dependency identities, effects, ownership, and return behavior.
- Signed-overflow, signed-shift, and address-space behavior.
- Endianness, alignment, protection, and low-address accessibility.
- Shared-location addresses, overlap relationships, and unspecified access properties.
- Stack-storage extent, initialization, and retention permissions.
- Extension rules for opaque narrow values.
- Fault delivery, partial effects, nonlocal exits, and nontermination.
- Concurrency, reentrancy, and synchronization requirements.

These unknowns prevent treating the function as a portable, fully determined API.

## Evidence

This contract derives solely from the supplied static representation. Dependency behavior and application semantics are not assumed.

Established evidence includes access widths, masks, offsets, argument positions, duplicated arguments, paired-result splitting, shared assignments, and relative ordering. In particular:

- Both offset-9 writes and the offset-0x34 word observation precede the first post-clear capture of A.
- The first A capture survives five calls before its right-by-23 transformation and occupies the third argument of the identified four-argument invocation.
- The second A capture independently supplies a later address basis.
- The late byte captured from absolute address 1 survives calls and the first assignment to B.
- The first word observation in the byte-update region occurs between its two byte stores.

Arithmetic bounds follow from established unsigned widths and masks. Application-level intent remains inferred.

No runtime execution or tests have been performed. Acceptance examples describe conditional obligations, not observed outcomes.
