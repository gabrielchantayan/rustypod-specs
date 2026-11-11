<!-- Trusted provenance: F-6cfa98d77c0f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6cfa98d77c0f

## Purpose

F-6cfa98d77c0f performs an ordered sequence of mixed-width memory accesses, prepares stack-resident values, invokes an incoming indirect target, and then invokes F-cdec09f5d432. If both calls return normally, the represented path reaches an operation presented as non-returning.

No application-level purpose is established. Repeated offsets, self-valued stores, and address-like values do not establish a container, allocator, protocol, or conventional object layout. In particular, multiplication-derived addresses are part of the represented behavior; replacing them with ordinary pointer traversal would change that behavior.

This contract distinguishes:

- **Established static evidence:** represented access widths, value dependencies, accesses, and ordering.
- **Inferred intent:** possible interpretations that are not acceptance requirements.
- **documented_only:** unresolved behavior requiring additional machine, interface, memory-map, or dependency evidence.

No runtime tests have been performed. The access sequence below describes the supplied representation, not a guarantee that every represented store is externally observable.

## Interface

### Explicit inputs and result

The apparent interface has seven positional inputs.

| Input | Represented width | Established use |
|---|---:|---|
| 1 | 32 bits | Supplies a base for word reads at byte offsets 20, 100, and 116; also supplies values shifted left by 4 and 8 bits. |
| 2 | 32 bits | No explicit use is visible. |
| 3 | 32 bits | Contributes to the destination address of the initial halfword store. |
| 4 | 8 bits | Supplies an early byte store, followed later by another store to the same represented address. |
| 5 | 32 bits | The address of its local argument storage is passed to the incoming indirect target. |
| 6 | 32 bits | No explicit use is visible. |
| 7 | 32 bits | Its local argument storage is assigned zero after the incoming indirect call returns. |

No result value or normal return is established.

### Ambient inputs and data locations

Four additional 32-bit entry values are consumed without visible initialization. This contract calls them **entry word one**, **entry word two**, **entry word three**, and **entry word four**. These are descriptive labels, not recovered semantic roles.

An incoming indirect call target is also consumed without visible initialization. It is designated **F-unknown-entry**.

Three opaque data locations participate:

- **Slot A:** read twice in the first mutation group.
- **Slot B:** read twice in the second mutation group.
- **Slot C:** read after F-unknown-entry returns, for the call to F-cdec09f5d432.

Separate reads of A or B must not automatically be treated as a single snapshot. Their storage properties and stability are unknown.

### Widths, layouts, and preconditions

All offsets below are byte offsets. “Word,” “halfword,” and “byte” mean 32, 16, and 8 bits respectively.

Both halfword reads are signed. Their values are sign-extended into the represented 32-bit integer domain before further use. Byte and halfword stores select the low 8 or 16 source bits respectively.

The representation uses 32-bit integer values for address calculations, products, and shifts. This alone does not establish portable overflow, address wrapping, signed-shift, or integer-to-address conversion rules. Those rules remain documented_only; no modulo-addressing model is assumed.

Successful traversal of the described path requires accessible storage at every reached address and valid execution targets. No explicit null, bounds, alignment, arithmetic-overflow, or target-validity checks are visible.

The complete calling convention and stack layout are unavailable. Distinct represented stack locations are described below, but their full relationship to the apparent function boundary and incoming storage is not established.

## Behavior

### Ordering, aliasing, and observability

The following sequence preserves the represented order of accesses and their value dependencies. It is not an atomic transaction.

A captured value remains the source of its later uses unless another load explicitly replaces it. A later memory read, however, is not an entry-time snapshot: earlier writes may affect it through aliasing.

The same limitation applies to opaque slots and stack-resident storage. No general non-aliasing guarantee is available.

The presence of a store in this sequence does not by itself prove external observability. In particular, an overwritten ordinary-memory store need not be observable. Observability depends on aliasing, intervening observations, and memory-access semantics. Hardware ordering, concurrency, access atomicity, and optimization constraints remain documented_only.

### Initial mutations and recurring-byte selection

The first represented memory effect is a halfword store at the sum of input 3 and entry word two. Its value is the low halfword of entry word one.

A word read at entry word two plus 68 captures an **initial link**. Next, entry word three is stored at entry word three plus 84. This is a self-valued store based on entry word three, not entry word two.

The first read of slot A captures an **A base**. A second read of slot A supplies the word stored at entry word one plus 68. The initial link is then stored at A base plus 20. If the two A reads differ, the stored word and captured addressing base therefore differ.

Input 4 is stored as a byte at entry word three plus 17. A word read at input 1 plus 100 then supplies a **recurring byte**, taken from its low eight bits.

Before the recurring byte is stored, the routine:

1. Reads a word at entry word three plus 68, capturing a **first loaded base**.
2. Reads a signed halfword at the sum of entry word three and A base.
3. Stores entry word three at initial link plus 84.

The recurring byte is then stored at entry word three plus 17.

The representation thus contains both the input-4 store and the later recurring-byte store, in that order. It does not establish that the earlier store is externally observable. Its visibility depends on aliasing, any intervening observation, and the semantics of the accessed memory.

### First loaded-base relationships

A base is stored at first loaded base plus 68. A word read at the sign-extended first halfword value plus 20 captures a **second loaded base**.

The subsequent represented operations are:

- Store first loaded base at its own offset 84.
- Store entry word four at the sum of A base and initial link.
- Store first loaded base at A base plus 20.
- Read a word at entry word four plus 68, then store its low byte at A base plus 13.
- Store the low byte of first loaded base at second loaded base plus 13.
- Store the low halfword of first loaded base at the sum of the first signed-halfword value and A base.

The read through entry word four occurs after the preceding stores. An aliased earlier store may therefore affect its value.

A word read at A base plus 68 captures an **A-associated word**. The routine then stores second loaded base at its own offset 84, before reading slot B.

### Second loaded-base relationships

The first read of slot B captures a **B base**. A second read supplies the word stored at first loaded base plus 68.

The A-associated word is stored at B base plus 20. The recurring byte is then stored at second loaded base plus 17.

Two reads follow:

- A word at second loaded base plus 68 supplies a **traversal factor**.
- A signed halfword at the sum of second loaded base and B base supplies a **signed traversal value**, after sign extension.

The representation then contains two consecutive repetitions of this ordered pair:

1. Store the signed traversal value as a word at that value plus 84.
2. Store the low byte of second loaded base at B base plus 17.

Neither repetition is omitted from the represented sequence. Whether equal repeated writes are distinguishable externally depends on the memory and observation model.

### Multiplication-derived address chain

The product of signed traversal value and traversal factor supplies the base for a word read at offset 68. That word supplies the base for a word read at offset 116. The resulting word supplies the base for a word read at offset 20. The last result is an **update base**.

At update base plus 20, the existing word is captured as a **preserved link**. That location is then overwritten with second loaded base shifted left by one bit.

The next word read uses preserved link plus 20. It does not use the replacement value just stored at update base plus 20.

The product of this newly read word and signed traversal value is an **intermediate product**. Before reading through it, the routine:

- Stores signed traversal value at that value plus 84.
- Stores the low byte of preserved link at B base plus 17.

A word read at intermediate product plus 116 captures a **retained traversal word** in a represented stack location.

The product of signed traversal value and intermediate product becomes a **later product**. The routine again stores signed traversal value at that value plus 84, then stores the low byte of retained traversal word at B base plus 17.

### Remaining data-memory accesses

The remaining accesses preserve distinct captured values and fresh reads:

1. Read a word at later product plus 68.
2. Read a word at input 1 plus 116, capturing an **input-derived word**.
3. Store retained traversal word as a full word at B base plus 100.
4. Store the low byte of the first later-product offset-68 read at B base plus 9.
5. Read later product plus 68 again, then store this fresh read’s low byte at later product plus 17.
6. Store signed traversal value at that value plus 84.
7. Store the low byte of retained traversal word at twice later product plus 17.
8. Read a word at the product of signed traversal value and later product, plus 68.
9. Read a word at input 1 plus 20, capturing a **final base**.
10. Store the low byte of input-derived word at retained traversal word plus 1.
11. Store the low byte of the final multiplication-derived read at final base plus 17.
12. Store the low byte of twice later product at final base plus 9.

The full-word store at B base plus 100 uses retained traversal word, not input-derived word.

The two reads at later product plus 68 are separate. The second occurs after the intervening full-word and byte stores; aliasing or other permitted memory effects may make their results differ.

### Stack preparation

Before F-unknown-entry is invoked, the representation prepares:

- Two locations containing input 1 shifted left by 4 bits.
- Four locations containing input 1 shifted left by 8 bits.
- Four zero-valued words.
- Three locations containing retained traversal word, including the location populated by its earlier read.

The visible preparation order is:

1. First shift-by-4 location.
2. First shift-by-8 location.
3. Four zero locations, in their represented order.
4. Second shift-by-4 location.
5. Second shift-by-8 location.
6. Second retained-word location.
7. Third shift-by-8 location.
8. Third retained-word location.
9. Fourth shift-by-8 location.

The retained word’s first location was populated earlier in the address chain. These assignments establish stored values and ordering, not a known aggregate type or argument structure.

### Calls and terminal operation

F-unknown-entry receives three visible arguments:

- The address of input 5’s local argument storage.
- Zero.
- Zero.

Only if that invocation returns normally does the represented continuation:

1. Assign zero to input 7’s local argument storage.
2. Assign the fixed code-location value `0x08400E1C` to another stack-resident word.
3. Assign zero to three further stack-resident words, in order.
4. Read slot C.
5. Invoke F-cdec09f5d432 with the slot-C value followed by three zeros.

Slot C is read after F-unknown-entry returns. The incoming target is not assumed to preserve it.

If F-cdec09f5d432 returns normally, the path reaches **F-terminal-opaque**, with represented operands zero and `0x08401154`. The supplied representation presents this operation as non-returning. It also depicts an indirect invocation through an apparent result of that operation.

No returning path from F-terminal-opaque to the apparent indirect invocation is established. Its actual runtime semantics remain documented_only; a representation-level non-returning designation is not independent proof of machine behavior.

## Side effects

Direct represented effects include byte, halfword, and word writes through incoming values, loaded values, sign-extended halfwords, products, and doubled values. They also include writes to local argument and stack-resident storage.

There is no visible rollback, transaction boundary, cleanup path, or returned error status. If a memory access or dependency interrupts execution, completion of later effects is not established. Partial completion of a faulting access is platform-dependent.

No explicit allocation or release is identifiable in the visible operations. This does not establish that dependencies avoid allocation, release, or other resource effects.

Passing input 5’s local address does not establish ownership transfer, permission to retain the address, or validity beyond the storage’s lifetime. No dependency is assumed to preserve globals, pointed-to memory, argument storage, or other machine state.

## Dependencies

| Opaque dependency | Established interaction | Unresolved behavior |
|---|---|---|
| F-unknown-entry | Incoming indirect target; receives the local input-5 address and two zeros | Identity, complete ABI, memory effects, ownership, retention, errors, and return behavior |
| F-cdec09f5d432 | Receives the post-callback slot-C value and three zeros | Argument meanings, state changes, ownership, errors, and return behavior |
| F-terminal-opaque | Receives represented operands zero and `0x08401154`; presented as non-returning | Instruction or runtime semantics, exceptional control flow, and any actual return behavior |

F-unknown-entry and F-terminal-opaque are contract-local opaque identifiers, not recovered identities.

Slots A, B, and C are data dependencies, not function identities. Their storage classes, stability, visibility, and alias relationships remain unknown.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Each assumes the relevant accesses are valid under the eventual machine model.

1. **Initial self-valued store.** If entry words two and three designate distinct ordinary regions, the early offset-84 self-valued store targets the region based on entry word three. Substituting entry word two changes the represented behavior.

2. **Two reads of A.** If slot A yields different values on its two reads, A-based addressing uses the first value, while entry word one plus 68 receives the second.

3. **Two reads of B.** If slot B yields different values, B-based addressing uses the first, while first loaded base plus 68 receives the second.

4. **Signed halfword interpretation.** A read halfword pattern of `0xFFFF` contributes the sign-extended value −1, not 65535. Effective addresses still depend on the unresolved arithmetic and address rules.

5. **Overwritten byte.** If input 1 plus 100 yields `0x123456AB`, the recurring byte is `0xAB`. Without relevant aliasing or intervening modification, entry word three plus 17 receives input 4 and subsequently `0xAB`. This alone does not require the earlier ordinary-memory value to be externally observable.

6. **Observable repeated writes.** If the memory-access model exposes every write as an ordered device event, both initial repeated self-valued stores and both intervening B-based byte stores contribute events. Without such semantics, their external distinguishability is not established.

7. **Saved pre-overwrite link.** When update base plus 20 is overwritten, the subsequent dependent read still uses the previously captured preserved link, not the replacement.

8. **Fresh later-product read.** If an intervening aliased write changes later product plus 68, its second read supplies the byte stored at later product plus 17. That byte need not equal the byte previously stored at B base plus 9.

9. **Distinct retained and input-derived words.** If retained traversal word differs from the word read at input 1 plus 116, B base plus 100 receives retained traversal word. The latter input-derived word instead supplies the byte stored at retained traversal word plus 1.

10. **Callback modification.** If F-unknown-entry changes slot C and returns, F-cdec09f5d432 receives the subsequently read value, subject to applicable visibility rules.

11. **Control-flow boundaries.** A non-returning F-unknown-entry prevents the represented post-call assignments and second call. If both ordinary dependencies return normally, the terminal operation is reached; an ordinary successful return is not established.

## Uncertainties

The following remain **documented_only**:

- Completeness of the apparent function boundary, signature, and incoming machine state.
- Origins and validity of the four ambient entry words and incoming call target.
- Complete stack layout, storage lifetime, and exposure of stack assignments.
- Exact overflow, multiplication, shifting, address wrapping, and pointer-conversion semantics.
- Endianness, alignment, atomicity, access protection, and fault behavior.
- Whether touched memory is ordinary, volatile, device-backed, shared, or externally modified.
- Which represented writes are externally observable, particularly overwritten or repeated writes.
- Stability and aliasing of the opaque slots and other storage.
- Dependency ownership obligations, preservation guarantees, resource effects, and nonlocal transfers.
- Actual terminal-operation semantics and reachability of its apparent result-mediated invocation.
- Behavior outside the supplied path.

No higher-level intent is sufficiently supported to become an acceptance requirement.

## Evidence

This contract is based on the supplied static representation’s operations and value dependencies. Prior prose, comments, and feedback are not independent proof of runtime behavior.

Static evidence establishes the represented widths, signed halfword reads, separate slot-A and slot-B reads, self-valued stores, repeated stores, arithmetic-derived address chains, stack-value preparation, and call ordering.

The earlier input-4 byte store is present before the recurring-byte replacement. Static presence and order do not establish guaranteed external observability of that overwritten store.

No executable trace, dependency implementation, authoritative ABI, memory map, or architecture specification was supplied. No runtime tests have been performed.
