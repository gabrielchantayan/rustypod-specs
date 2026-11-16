<!-- Trusted provenance: F-e8d3bb3c75da; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e8d3bb3c75da

## Purpose

F-e8d3bb3c75da accepts three represented 32-bit inputs, executes an extensive ordered sequence of indirect calls, modifies computed-address and fixed-address memory, and ends with an unresolved indirect transfer carrying four visible arguments.

No domain-level purpose is established. Literal values corresponding to printable characters do not establish formatting, serialization, string termination, or an output-buffer interface.

This contract distinguishes:

- **Established static evidence:** operations, widths, and ordering explicitly represented in the supplied body.
- **[INFERENCE]:** interpretations not established as behavioral guarantees.
- **documented_only:** unresolved behavior requiring dependency implementations, target execution semantics, or other unavailable evidence.

The observable contract concerns calls and memory-access history, not merely final memory contents.

## Interface

### Explicit inputs

| Position | Represented width | Established consumption |
|---|---:|---|
| First | 32 bits | No explicit expression in the body consumes this input. |
| Second | 32 bits | No explicit expression in the body consumes this input. |
| Third | Signed 32 bits | An early indirect call receives this input shifted left by six positions. |

The first two inputs cannot be declared irrelevant to execution: complete indirect-call signatures and register-level argument flow are unavailable. No pointer, capacity, length, string, enumeration, or flag interpretation is established for any explicit input.

### Implicit inputs and outputs

Execution also depends on entry execution context, indirect-call results, additional call-associated register values, fixed-address state, and memory reached through computed addresses. Identical explicit inputs therefore do not establish identical behavior.

The represented interface declares no return value. Observable outputs include:

- Ordered indirect calls and their visibly supplied arguments.
- Memory reads and writes, including repeated and overlapping accesses.
- Changes to fixed-address state.
- Four visible arguments supplied to the terminal indirect destination.

No status code, returned object, output length, or ownership-bearing result is established.

### Preconditions

Every reached call destination and memory access must be supported by the execution environment for execution to continue. No local checks establish destination validity, mapping, permissions, allocation extent, or alignment.

Low and null-derived addresses are genuine represented access destinations, not optional-object conventions. Target byte order, alignment handling, and whether addressed state is ordinary memory or device state are **documented_only**.

## Behavior

### Execution structure

The represented body contains an indirect-call phase, a direct-memory mutation phase, and a terminal indirect transfer. No explicit conditional branch or loop appears. Repeated operations are explicitly represented rather than shown as data-dependent iteration.

Later effects are conditional on earlier operations completing. Dependencies may branch internally, change state, fail, or never return. Calls with unused visible results remain part of the represented behavior; their elimination is not justified.

Ordering described below is the ordering represented between separate operations. It is not an independently verified instruction trace or a guarantee of atomicity.

### Call-mediated arithmetic

Early calls include two groups that pass a returned 32-bit value shifted left by eight. A subsequent call receives the third input shifted left by six.

Later calls combine shifted 32-bit results, upper and lower words of represented 64-bit results, additional register-associated values, explicit zeros, and masked fields. Masks include selections of 24, 22, 21, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11, and one bit. These widths do not establish object types.

Wide-result extraction must retain its width. Unsigned 64-bit right shifts by 34, 36, and 44 leave at most 30, 28, and 20 significant bits respectively. These are not 32-bit oversized shifts.

Masking and shifting order also matters. Selecting the low 24 bits before shifting left by six produces at most 0x3FFFFFC0. Selecting the low 22 bits before shifting left by eight produces at most 0x3FFFFF00.

Several explicitly zero-initialized quantities remain zero through their visible shifts. This does not establish that adjacent dependency results, globals, or incompletely represented arguments are zero.

Many other shifts and multiplications use represented signed 32-bit operands. Overflow and signed-shift semantics require qualification; modular bit-vector calculations below are conditional, not a portable signed-arithmetic guarantee.

### Address roles at the memory-phase boundary

The following descriptive roles identify address calculations, not recovered objects:

| Role | Derivation |
|---|---|
| High-scale base | A returned 32-bit value multiplied by 0x02000000. |
| Compact field | A returned value shifted left by eight, masked with 0x7FFF, then shifted right by six. |
| Compact base | Twice the compact field. |
| Shifted token | A saved call-derived value shifted left by 25. |
| Last wide result | The represented 64-bit result immediately preceding direct mutation. |

Under 32-bit bit-vector interpretation, the compact field ranges from 0 through 508 in multiples of four. The compact base ranges from 0 through 1016 in multiples of eight. These are low computed addresses, not validated indices.

### Initial memory mutations

A word store at offset 0x14 from the upper word of the last wide result precedes a byte store at shifted-token offset 9. A word store at offset 0x54 from the last result’s lower word multiplied by 256 intervenes before a second byte store at the same shifted-token-relative destination. That second byte store supplies the shifted token’s low byte.

The body then stores a zero word at high-scale-base offset 0x14 and reads a byte at high-scale-base offset 5. After a zero word store at compact-base offset 0x14, the observed byte is zero-extended into a word store at high-scale-base offset 0x34.

The following word read at high-scale-base offset 0x14 is a fresh memory observation. It cannot be replaced unconditionally with zero: preceding accesses may alias. Another zero word store at compact-base offset 0x14 precedes two byte stores at compact-base offset 0x15. Those byte destinations lie inside the word-sized region beginning at offset 0x14.

Next, an absolute byte read at 0x05 supplies a zero-extended address token. An absolute byte write at 0x01 supplies the compact base’s low byte. Further accesses use the absolute-read token at offsets 0x15 and 0x54. The old word read from token offset 0x54 is retained before that location is overwritten.

### Pointer-dependent mutation and scaling

The retained old word becomes an important address and scaling source. Subsequent accesses include:

- Word reads at offsets 0x64 and 0x74, whose results become further address tokens.
- A byte read at offset 5 whose zero-extended result becomes the base of a word read at offset 0x74.
- Repeated self-address word stores at offset 0x24.
- Repeated byte stores at offset 9 and compact-base offset 0x15.
- Word stores at offsets 0x14, 0x34, 0x44, and 0x54.
- An absolute word write at 0x54 and an absolute byte write at 0x15.

One self-address expression contains two word reads of compact-base offset 0x74: one supplies the destination token, and the other supplies the stored value. Their internal evaluation order is not established. Replacing them with one cached read is not justified for changing or access-sensitive memory.

Several byte reads from the same computed offset 5 are likewise distinct observations separated by stores. Loaded words and bytes must retain their represented address roles; no shared structure layout or disjoint-allocation model is established.

A later group writes 0x3A as a byte at four times a loaded word, as a halfword at eight times that word, and as a word at offset 0x20 from the four-times-scaled address.

A different base receives word writes at offset 0x20 in the order 0x20, 0x2F, a zero-extended byte freshly read at base offset 4, and 0x20 again. These intermediate writes remain observable obligations where the memory model exposes access history.

### Repeated mutation region

For the remainder of this description:

- The **retained seed** is the earlier old word read through the absolute-0x05-derived token at offset 0x54.
- The **adjusted scale** is four times the retained seed, minus 0x96.
- The **working base** is four times the adjusted scale.
- The **anchor** is a word read once from a separate fixed state location.
- The **linked base** is twice a later pointer-dependent loaded word.
- The **repeated payload** is four times that same loaded word.

These values are saved address tokens; they are not object identities or ownership claims.

After saving the anchor, the body writes the word 0x20 at absolute address 0x60. A large fixed repeated region then performs ordered groups consisting of a payload word store at scaled-base offset 0x54, a low-working-base byte store at anchor offset 0x19, a low-anchor byte store at working-base offset 0x11, and a scaled-value word store at anchor offset 0x34.

The first group uses eight times the retained seed. Subsequent groups use eight times the adjusted scale. The identical later destinations do not advance, so this is not established as array traversal. Repetition cannot be collapsed solely because final ordinary-memory contents would match.

### Late reads, saved values, and absolute address 0x67

After the repeated region, the body reads a word at absolute address 0xA9 and a word at repeated-payload offset 0x74. It writes a byte at absolute address 0x3A derived from eight times the adjusted scale plus 0x30, subject to byte truncation.

A word read at linked-base offset 0x14 is saved as the terminal transfer’s **third argument**. A separate word read at working-base offset 4 is saved for a later store.

Byte writes then affect linked-base offset 0x0D and working-base offsets 8, 0x0D, and 9; the offset-8 value is 0x6C. A self-address word store at linked-base offset 0x54 precedes an absolute byte write at **0x67**, which supplies the low byte of the saved absolute-0xA9 word.

There are **three distinct subsequent byte reads of absolute address 0x67**:

1. **First read:** after a word read at linked-base offset 0x44, the byte at 0x67 supplies the byte write to absolute address 0x21.
2. **Second read:** after intervening computed-address writes and a word read at working-base offset 0x40, another read of 0x67 supplies a byte store at working-base offset 9.
3. **Third read:** after further stores and a word read at working-base offset 0x54, a fresh read of 0x67 supplies another byte store at working-base offset 9.

The intervening effects are significant. Between the first and second reads, the body writes a byte at working-base offset 0x10, writes and then replaces a word at anchor offset 0x54, writes a shifted byte at linked-base offset 1, and performs word stores at anchor offsets 0x74 and 0x44 and working-base offset 0x10. The offset-0x74 store uses the previously saved working-base-offset-4 word; the working-base-offset-0x10 word store uses the fresh offset-0x40 read.

Between the second and third reads, the body stores a byte at linked-base offset 9, stores the saved absolute-0xA9 word at working-base offset 0x44, and loads a word from working-base offset 0x54.

Any computed store may alias 0x67. Consequently, the three reads need not observe the same byte or the byte originally written there. Substitution of the saved absolute word’s low byte, a cached first observation, or another saved fragment is not justified.

The shifted byte at linked-base offset 1 uses a signed word and a shift count derived from the working base’s low byte. Counts may exceed 31; target behavior remains **documented_only**.

### Final mutation ordering and transfer

The word loaded from working-base offset 0x54 becomes the **late linked address**. Following the third read of 0x67, the body writes the low byte of the saved repeated-payload-offset-0x74 word at late-linked-address offset 0x0D. It restores the saved third argument at linked-base offset 0x54, reads a word at working-base offset 0x14, and writes another byte fragment at linked-base offset 0x11.

It then writes the late linked address’s low byte at working-base offset 9 and the full address at working-base offset 0x44.

A fresh word read at linked-base offset 4 supplies the **terminal source address**. A halfword store of 0x65 occurs at twice this address. The following word read at terminal-source-address offset 0x64 is saved as the terminal transfer’s **second argument**.

Before loading the fourth argument, the body writes the low byte of the saved working-base-offset-0x14 word at late-linked-address offset 0x11 and stores the word 100 at late-linked-address offset 0x54. It then reads terminal-source-address offset 0x74 and saves that word as the **fourth argument**.

Execution does **not** transfer immediately after this read. The final memory effects occur in this order:

- A word read at **linked-base offset 0x64** supplies a new destination address.
- A byte store at **linked-base offset 9** supplies the terminal source address’s low byte.
- A word store at **new-destination-address offset 0x54** supplies the saved second argument.
- A word store at **twice the saved second argument** supplies the new destination address.

Only afterward does the unresolved transfer receive:

| Argument position | Visible value |
|---|---|
| First | 0x72 |
| Second | Saved terminal-source-address-offset-0x64 word |
| Third | Earlier saved linked-base-offset-0x14 word |
| Fourth | Saved terminal-source-address-offset-0x74 word |

The final stores may alias the locations from which these arguments were read. The transfer nevertheless uses the saved values, not rereads of those locations.

The destination, complete calling convention, and behavior after transfer are **documented_only**. A represented return afterward does not prove that the destination returns normally.

## Side effects

### Memory and state

Explicit access widths are 8, 16, and 32 bits. Byte and halfword stores retain the low eight and sixteen bits respectively. Overlapping accesses must preserve their represented widths and order; final byte layout depends on target byte order.

Fixed-address effects include a byte read at 0x05; byte writes at 0x01, 0x15, 0x21, 0x3A, and 0x67; three separate byte reads at 0x67; word writes at 0x54 and 0x60; and a word read at 0xA9.

No transaction, rollback, locking, synchronization, or failure cleanup is visible. A saved address token does not establish that its source location remains unchanged.

### Ownership and lifetime

No explicit allocation, release, reference counting, or ownership transfer is identifiable. Unknown dependencies may nevertheless allocate, retain, release, or invalidate memory. No global-preservation or returned-address-lifetime guarantee may be assumed for them.

### Errors

No explicit error result or recovery branch is visible. Invalid accesses, unsupported indirect destinations, arithmetic-related execution failures, and dependency failures have no established translation into a local status.

A failure may leave an observable prefix of the represented effects. No all-or-nothing guarantee is supported.

## Dependencies

The supplied mapping identifies only F-e8d3bb3c75da. It supplies no opaque function IDs for its indirect dependencies; assigning identities to them would fabricate information.

Unresolved dependencies comprise the entry-context-derived destination, the later indirect-call destinations, and the terminal destination. Their signatures, identities, effects, error behavior, ownership rules, and global-preservation properties are **documented_only**.

Similar argument patterns do not prove that different call sites invoke the same dependency.

## Acceptance examples

These are conditional static criteria, not executed tests.

1. **Explicit-input transformation.** If execution reaches the relevant call and the represented shift is supported, a third input of 1 supplies 0x40 in the visibly derived argument position.

2. **Wide extraction.** A represented unsigned 64-bit value of 0x123456789ABCDEF0 shifted right by 44 yields 0x12345. A 32-bit interpretation does not satisfy this criterion.

3. **Compact-base boundary.** Under 32-bit bit-vector arithmetic, a relevant result of 0x7F gives compact base 1016; 0x80 gives zero. The calculation does not saturate.

4. **Unsigned byte widening.** An observed byte of 0xFE at high-scale-base offset 5 supplies the word value 254 at offset 0x34, not −2.

5. **Repeated write history.** If offset 0x20 is access-sensitive state, the writes 0x20, 0x2F, freshly loaded byte value, and 0x20 remain observable in that order.

6. **Three independent 0x67 observations.** If the three reads observe 0x12, 0x34, and 0x56 because permitted aliasing stores or external state changes intervene, absolute 0x21 receives 0x12; the two subsequent working-base-offset-9 stores receive 0x34 and 0x56 respectively. Later overwrites do not erase those access-history obligations.

7. **Final destination and saved arguments.** Suppose the saved second argument is 0x1000 and the final linked-base-offset-0x64 read yields 0x2000. Subject to supported address arithmetic, the final word stores supply 0x1000 at 0x2054, then 0x2000 at 0x2000. The intervening linked-base-offset-9 byte store precedes both. Aliasing changes to argument-source memory do not change the saved transfer arguments.

8. **Dependency non-return.** If an early dependency never returns, subsequent direct-memory effects are not required to occur.

## Uncertainties

The following remain **documented_only**:

- Domain purpose, intended consumer, and object model.
- Complete call graph, signatures, and register conventions.
- Target arithmetic, address conversion, byte order, and alignment behavior.
- Memory mapping, access sensitivity, permissions, validity, and lifetime.
- Oversized signed-right-shift behavior and sign propagation.
- Terminal-transfer semantics and normal-return behavior.
- Accuracy of the represented function boundary and recovered execution path.

Modular arithmetic is used only for explicitly conditional examples and bounds. Signed overflow, oversized shifts, and address arithmetic must not be assigned guessed target semantics.

[INFERENCE]: unusual low-address accesses and repeated transformations may reflect incomplete recovery of execution semantics. This possibility does not authorize removing represented effects or inventing conventional buffers or structures.

## Evidence

This contract derives from static inspection of the supplied representation, independently of the prior draft’s claims and reviewer assertions.

The represented operations establish three distinct reads of absolute byte address 0x67, their separate destinations and intervening accesses, and the final offset-0x64 read followed by one byte store and two word stores before the terminal transfer. They also establish that the transfer arguments are retained values despite possible later aliasing.

No dependency implementation, target ABI, memory map, or independent execution evidence was supplied. Source comments and recovery warnings are not independent runtime evidence.

No runtime tests have been performed. All acceptance examples are conditional static obligations, not observed executions.
