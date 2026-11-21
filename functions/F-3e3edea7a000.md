<!-- Trusted provenance: F-3e3edea7a000; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3e3edea7a000

## Purpose

F-3e3edea7a000 is a state-mutating operation whose application-level purpose is not established. The supplied static body shows dependency invocations interleaved with direct byte, halfword, and word accesses. Its behavior depends on two explicit inputs, incoming machine state, shared memory, and values exposed by dependencies.

The supported contract is an **ordered access-and-invocation trace, conditional on valid execution context, accessible memory, and returning dependencies**. It is not a pure transformation of its explicit inputs.

Repeated masks, bit extraction, scaled addresses, and recurring offsets suggest manipulation of encoded values and associated storage. This is inferred intent only. The evidence does not establish allocation, serialization, object construction, reference counting, or cleanup.

The supplied body ends at an invalid-instruction boundary. Normal return and a completed final state are not established.

## Interface

### Explicit inputs

| Input | Width | Established use |
|---|---:|---|
| First input | 32 bits | Supplies a masked and shifted value for an early byte write. |
| Second input | 32 bits | Passed to the first visible indirect invocation. |

Neither input is established as a pointer, length, handle, enumeration, or ownership-bearing value.

For the early first-input-derived write, bit 31 is cleared, the result is shifted right by 21, and the destination retains eight bits. The stored byte therefore reflects original bits 21–28. Original bits 29–30 survive the shift but not the narrowing.

### Implicit inputs

The visible body also consumes:

- An incoming 32-bit address-like value used for a byte lookup and a right-shifted call argument.
- An incoming byte used by the first visible store.
- Incoming control-flow state selecting the first indirect target.
- Additional machine-state values exposed after dependency invocations.
- Directly read shared bytes and words, including two late reads from a shared word whose address is not supplied.

These inputs prevent treating the recovered two-argument interface as a complete calling convention.

The late shared word is a distinct implicit input. Its first read supplies a retained 32-bit value for subsequent masked arithmetic; its second read supplies a separately retained, four-times-scaled address base. Its address, meaning, ownership, and stability remain unknown.

### Outputs and completion

No explicit returned value is established. Observable effects include direct memory writes, dependency invocations with represented arguments, and whatever additional effects those dependencies perform.

There is no established success flag, output buffer, error code, exception interface, or normal completion condition.

### Preconditions

Following the visible trace without a fault requires:

- Valid incoming implicit state.
- Callable indirect targets under the actual machine convention.
- Readable or writable memory for every executed access at its indicated width.
- Usable dependency-produced values wherever they become addresses or arithmetic operands.
- Storage that remains accessible when retained address-like values are used after calls.

The body shows no general validation of nulls, bounds, alignment, permissions, lifetimes, or dependency failures. These are environmental requirements, not promised checks.

## Behavior

### Widths and arithmetic

The representation distinguishes 8-bit bytes, 16-bit halfwords, and 32-bit words. Selector bytes are unsigned and range from 0 through 255. Narrow stores retain only the low bits required by the destination width.

Unsigned 32-bit shifts and arithmetic must be interpreted at their represented widths, not as unbounded arithmetic. In particular, unsigned multiplication and left shifts can discard high bits. Signed-looking scaling, left shifts, and address calculations do not establish portable signed-overflow behavior; their exact machine interpretation is documented_only.

Several dependencies expose a result represented as 64 bits. The body separately consumes low and high 32-bit portions. This establishes two result components, not necessarily an application-level 64-bit scalar ABI.

### Entry and first input-derived update

The first visible access reads a byte one byte beyond the incoming implicit address-like value. Adding 17 to that unsigned byte gives the absolute destination of a byte store containing the incoming byte.

The destination is therefore between absolute addresses 17 and 272 inclusive. It is not an offset from the incoming address-like value.

The first indirect invocation then receives the second explicit input. Further invocations follow, including one receiving the incoming address-like value shifted right by 11.

The selector at the original address plus one is subsequently read again. That second read selects the destination for the byte extracted from the first explicit input.

These selector reads are separate events with dependency calls between them. The second must not be replaced by the first retained selector: intervening effects may change the selected destination.

### Byte-selected update family

A recurring operation reads an unsigned byte from a computed location and writes one byte at the absolute address equal to that selector plus 17.

Selector sources include:

- The incoming address-like value plus one.
- Scaled dependency-produced values plus one.
- Masked and shifted values plus one.
- The fixed byte at absolute address 1.

Stored values include zero, low bytes of scaled values, narrowed field values, and shifted extracts. Important extraction boundaries include:

| Represented extraction before byte narrowing | Original bits contributing to the byte |
|---|---|
| Unsigned right shift by 22 | 22–29 |
| Mask to 30 bits, then right shift by 20 | 20–27 |
| Mask to 22 bits, then right shift by 12 | 12–19 |

A right shift can produce more than eight bits; that does not widen the store.

Zero writes are distinct operations. Some precede later writes selected through the same address expression, with calls and fresh selector reads between them. Neither equal source expressions nor an apparently redundant final value permits collapsing the represented access trace.

### Fixed word at address 124

A dependency-produced 32-bit value is stored at absolute address 124. A following invocation receives two copies of that retained value.

Those argument positions do not introduce additional reads from address 124. The location has no established semantic name, ownership role, or persistence guarantee.

### Three reads of the word at address 40

The address-40 interaction contains **three direct word reads**, not one read reused everywhere:

1. A 32-bit read captures a value for a later store.
2. The immediately following indirect invocation obtains one argument from a fresh 32-bit read at address 40.
3. Its other argument comes from another fresh 32-bit read at address 40.

The captured read precedes the invocation’s argument reads. The supplied expression does not establish the mutual evaluation order of the two argument reads.

After that invocation and another invocation, a direct 32-bit store writes the captured value to an address derived by multiplying a dependency-exposed component by four and adding 20.

The later store uses the first captured value. It does not use either argument-read value or reread address 40 at store time. The three reads must not be merged, and their values are not guaranteed equal.

### Derived storage accesses

The body includes 32-bit reads at offsets 4, 8, 12, 20, and 28 from different derived bases. It also contains:

- Numerous 32-bit stores at offset 20 from four-times- or sixteen-times-scaled values.
- Additional stores at offset 20 from larger scaled values.
- A byte store at offset 28 from a dependency-produced high result component.
- Word stores at offset 52.
- A separate byte store through a masked-and-shifted result-derived address.

These access shapes do not establish a common record type, array, field meaning, or allocation layout.

In one interaction, a high result component supplies a base for a word read at offset 8 and then a word read at offset 12. The captured offset-8 value later supplies a stored byte; the offset-12 value participates in selecting its destination. Calls intervene before that byte store.

### Two argument reads from address 1

The body contains a later indirect invocation whose two argument positions each directly read the fixed byte at absolute address 1. Each read supplies an unsigned byte value widened for the argument.

This invocation is distinct from the call immediately following the offset-28 byte store:

- The immediate post-store invocation receives two copies of a retained low result component.
- Additional calls and byte-selected updates follow.
- The later invocation independently reads address 1 in both argument positions.
- After another invocation, the next byte-selected zero store reads its selector from a separately computed location.

The two address-1 argument reads have no established mutual evaluation order. They must not be described as two copies of one captured selector, and equality is not guaranteed. The subsequent zero store must not be modeled as reusing either argument read.

### Zero-address stores

A retained unsigned quantity is explicitly set to zero and remains zero through the relevant uses.

A dependency-produced word is stored through its eight-times-scaled destination, yielding a 32-bit store at absolute address zero.

Later, masking that same zero quantity and shifting right by 11 yields a halfword destination at address zero. A subsequent dependency result supplies the low 16 bits stored there.

These are conditional static consequences, not observed successful accesses. The stores have different widths and occur at different points, with intervening effects. The later halfword store affects two bytes, not the entire earlier word.

Whether low memory is mapped, whether these accesses fault, and whether the recovered entry context is valid remain unresolved.

### Consecutive offset-52 stores

One derived offset-52 word address receives a widened selector value and then that value shifted left by one. Both stores occur consecutively without an intervening invocation.

Another derived offset-52 address receives the same shifted word value twice consecutively.

The represented trace contains both stores in each pair. Final-memory equivalence alone does not establish permission to remove either access.

### Late shared-word input

Late in the trace, after an offset-20 word store and before the next indirect invocation, the operation reads the same unknown-address shared word twice:

- The **first read** is retained as a 32-bit unsigned value. A later invocation receives that retained value masked to 30 bits and shifted right by 10.
- The **second read** is separately multiplied by four to form an address base. Subsequent selector reads use that retained base plus one.

The first read precedes the second in the supplied body. Their values are not guaranteed equal, even though no dependency invocation appears between them.

The scaled base survives intervening invocations. It supplies two distinct selector reads, with calls between those reads. The first selected byte store uses the low byte of a dependency result multiplied by four; the later selected store uses that retained result masked to 30 bits and shifted right by 20.

Thus, two independent distinctions must remain intact:

- The first and second shared-word reads serve different purposes.
- The later selector reads reuse a retained address base, not a retained selector byte.

The retained scaled base subsequently participates in left-shift-by-26 arithmetic, word reads at a further offset of 28, and additional scaling. These signed-looking operations require machine-level confirmation; they must not be silently simplified using unbounded arithmetic or an assumed overflow rule.

The invocation reading that derived offset-28 word in two argument positions contains two direct word reads. Their mutual evaluation order and equality are not established.

### Tail and termination boundary

The remaining body continues with dependency invocations, byte-selected writes, scaled offset-20 word stores, and word reads.

Similar-looking address expressions are not necessarily aliases. In particular, an offset-20 store from a sixteen-times-scaled value does not generally initialize an offset-20 read from the corresponding four-times-scaled value.

A later invocation also performs two separate direct reads of one derived offset-20 word for differently shifted arguments. It must not be reduced to a single captured read without further evidence.

The final visible operations include a byte-selected store followed by additional indirect invocations. Control flow then truncates at invalid instruction data. No normal return, final invariant, or cleanup phase is established.

## Side effects

### Ordering, rereads, and aliasing

The contract distinguishes:

- Retained values passed repeatedly as arguments.
- Multiple direct memory reads within one invocation’s argument expressions.
- Reads separated by dependency calls.
- Repeated writes to the same effective address.
- Different expressions that may resolve to overlapping storage.

Argument position does not establish evaluation order between direct reads in the same invocation. Conversely, a captured read in a preceding statement is distinct from those argument reads.

Dependencies are not assumed to preserve globals, selectors, referenced storage, or resource lifetimes. Retaining a numeric address across a call preserves the value, not the accessibility or contents of its target.

The supplied representation warns of overlapping global storage. Low-address byte, halfword, and word accesses must not be modeled as independent named variables.

These are distinctions in the supplied static access trace. They do not establish atomicity, hardware barriers, volatile qualification, or cross-thread visibility guarantees.

### Ownership and errors

No direct allocation or release operation is identifiable. Opaque dependencies may allocate, release, retain, or mutate storage; none of those contracts is supplied.

There is no established ownership transfer, borrowed lifetime, rollback, or restoration guarantee.

No explicit error-reporting or recovery path is visible. Earlier writes are not visibly undone when a later dependency fails or an access faults. Fault handling, exception propagation, nonreturning dependencies, and the runtime consequence of the terminal boundary are documented_only.

## Dependencies

Six externally named dependencies have supplied opaque IDs. Their relative order is:

| Order | Opaque function ID | Represented argument relationship |
|---:|---|---|
| 1 | F-23325cd5e50c | One dependency-produced 32-bit value. |
| 2 | F-bbb54b5f036d | Two copies of one retained 32-bit value. |
| 3 | F-f80b3ac065fc | Low result portion, high result portion, then two additional copies of the low portion. |
| 4 | F-de15789688a8 | Two copies of one retained 32-bit value. |
| 5 | F-ea83f9a17404 | Two copies of one retained 32-bit value. |
| 6 | F-674e75a953e8 | Two copies of one retained 32-bit value. |

These calls are not adjacent. Their argument relationships do not establish semantic types, return contracts, ownership effects, or global preservation.

Additional direct targets lack supplied opaque IDs. Numerous invocations use computed targets and zero through four represented arguments. Their identities, full calling conventions, effects, and argument completeness are documented_only.

No direct self-call is established.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Each assumes execution reaches the relevant operation and required accesses are valid.

1. **Absolute selector destination.** If the initial selector is 200 and the incoming byte is 90, the first store writes 90 at absolute address 217, not at an offset from the incoming base.

2. **Selector reload.** If the entry selector changes from 5 to 9 during intervening calls, the initial store targets address 22 and the later first-input-derived store targets address 26.

3. **Input narrowing.** For a first input of 0x7FFFFFFF, masking and shifting produces 1023; the byte store writes 255. Changing only bit 31 does not affect this particular extracted byte.

4. **Shift width.** A 32-bit value of 0xC0000000 shifted right by 22 produces 768. A byte store of that result writes zero.

5. **Address-40 read separation.** Suppose the captured read yields 11, the invocation’s first-position read yields 22, and its second-position read yields 33. The invocation receives 22 and 33, while the later captured-value store writes 11. This does not prescribe which argument read occurs first.

6. **Address-1 argument separation.** If the two direct byte reads yield 7 for the first argument position and 9 for the second, the invocation receives 7 and 9. A model requiring equal arguments from a single capture is inconsistent with the represented accesses.

7. **Late shared-word separation.** If the first late shared-word read yields 4096 and the second yields 32, the later masked argument is 4, while the four-times-scaled base is 128. The associated selector reads use address 129. These roles must not be exchanged or merged.

8. **Retained base, fresh selectors.** With that retained base of 128, if address 129 changes from 5 to 9 between the relevant selector reads, the selected stores target addresses 22 and 26. The shared word need not be reread to obtain either destination.

9. **Consecutive stores.** For a retained selector of 200, the relevant offset-52 stores write 200 and then 400. The other duplicate-store pair still represents two writes even when both values match.

10. **Partial overlap and interruption.** A successful halfword store at zero changes two bytes; endianness determines their numeric relationship to an earlier word. Intervening writes may also have changed that word. If a dependency does not return, subsequent accesses are not required, and rollback is not established.

## Uncertainties

The following remain documented_only:

- Application-level purpose and valid standalone entry conditions.
- Complete ABI and required incoming machine state.
- Unknown dependency identities, contracts, and lifetime effects.
- The late shared word’s address, meaning, ownership, and stability.
- Exact machine semantics for signed-looking shifts, scaling, and address overflow.
- Endianness, alignment requirements, memory permissions, and low-memory mapping.
- Volatility, device-backed storage, and concurrent mutation.
- Whether recurring offsets correspond to genuine fields or encoded layouts.
- Runtime consequences of invalid instruction data and incomplete control-flow recovery.
- Normal completion, final state, and any returned runtime result.

Independent reads are established access distinctions. A mechanism making their values differ is not established; neither is a stability guarantee making them equal.

## Evidence

Established evidence consists solely of the supplied static body and dependency mapping:

- Two explicit 32-bit inputs and additional implicit machine-state uses.
- Interleaved dependency invocations and direct 8-, 16-, and 32-bit accesses.
- Unsigned-byte selection of absolute destinations at selector plus 17.
- Three distinct address-40 word reads: one capture and two invocation-argument reads.
- Two direct address-1 byte reads within one later invocation.
- Two ordered late shared-word reads with separate retained-arithmetic and scaled-address roles.
- Further repeated direct word reads inside invocation arguments.
- Fixed low-address accesses, conditional zero-address stores, and consecutive duplicate stores.
- Overlapping-storage warnings and terminal control-flow truncation.

Encoded-data manipulation is inferred intent only. No runtime tests have been performed.
