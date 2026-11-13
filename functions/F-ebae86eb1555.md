<!-- Trusted provenance: F-ebae86eb1555; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ebae86eb1555

## Purpose

F-ebae86eb1555 specifies an ordered sequence of byte and 32-bit memory accesses involving four explicit inputs, additional incoming address state, pointer-following reads, computed destinations, and text-associated storage. It publishes five words through the explicit output address, then continues reading and modifying other storage.

An application-level purpose is not established. Text associations do not establish rendering, timing, calibration, movement, or a valid object model.

The supplied static representation reports invalid instruction data and terminates at a truncated-control-flow boundary. This contract describes the represented accesses, not a verified executable routine. Normal completion, a return value, and behavior beyond that boundary are **documented_only**.

## Interface

### Inputs

| Role | Represented width | Uses |
|---|---:|---|
| Primary address | 32 bits | Byte and word destinations; published address value |
| Initial value | 32 bits | Full-word and low-byte publication |
| Secondary address/value | 32 bits | Published value and writable address base |
| Output address | 32-bit address representation | Published address value and base for five word stores |

Three additional incoming address bases are used without definitions in the explicit interface:

- **Context:** receives fields and supplies linked addresses and captured words.
- **Provider:** supplies a word and participates in address arithmetic.
- **Companion:** receives fields and supplies traversal addresses.

The representation does not establish a validated distinction between integers and addresses. A byte used twice near the end has no established preceding definition. Its origin and executable treatment are **documented_only**.

### Storage and results

The explicit output stores occupy offsets 0, 4, 8, 12, and 16: 20 bytes when interpreted as one contiguous region. This is not the complete storage requirement. Traversal addresses and data-dependent displacements prevent deriving a fixed overall requirement from the explicit inputs.

No return value, normal return, validation result, or recovery result is established. No checks for null addresses, bounds, alignment, storage permissions, or allocation success are represented.

Reaching an access requires all preceding operations to have proceeded. Executing it requires the environment to permit the represented access. Address publication does not establish ownership transfer or lifetime extension.

## Behavior

### Access and aliasing rules

Offsets are byte offsets. A byte is eight bits; a word is 32 bits. A low-byte publication uses the least significant eight bits of its source. The byte-derived address described below is explicitly zero-extended to 32 bits.

The following sequence preserves represented read/write order. Captured values remain snapshots; fresh reads remain separate operations even when they address the same field.

Independently obtained bases may identify overlapping storage. Such overlap can make writes through one base affect later reads through another. This does **not** imply overlap between arbitrary offsets of one captured base. Under ordinary byte-addressed memory semantics, that requires intersecting byte ranges. In particular:

- A traversal byte at offset 9 does not overlap its word at offset 0x54.
- A chain byte at offset 9 does not overlap its word at offset 0x44.

Claims about observed memory values below assume sequential ordinary-memory semantics, without concurrent or external mutation. Different virtual addresses mapping to the same physical storage, device-memory behavior, and concurrent mutation are additional, unestablished environmental assumptions.

Four text-associated identities are used without assigning string or object semantics:

- **First text address:** one text-associated base plus eight bytes.
- **Second text address:** a different text-associated base plus eight bytes.
- **Third text address:** another text-associated base, without that displacement.
- **Text-associated record:** storage receiving five word stores.

### 1. Initial publication and capture

The initial value’s low byte is written to primary offset 1, its full word to context offset 0x74, and its low byte to context offset 9, in that order. Context offset 0x10 then receives the primary address’s low byte, followed by companion offset 0x54 receiving the secondary address/value.

The provider word at offset 0x64 is then captured. Its low byte replaces primary offset 1, its full word replaces context offset 0x74, and its low byte replaces context offset 9.

Primary offset 0x14 receives the output address. Provider offset 0x34 receives the companion address. Context offset 0x44 is read to establish the **first linked address**. Immediately afterward, context offset 9 again receives the captured provider word’s low byte. This repeated write is retained, including when intervening cross-base stores affect that byte.

The next stores occur in this order:

| Destination | Value and width |
|---|---|
| Primary + 0x54 | Context address, word |
| Companion + 0x54 | Output address, word |
| First linked + 0x14 | Secondary address/value, word |
| Context + 0x10 | Primary address, word |
| Companion + 0x14 | Primary address, word |
| Context + 0x14 | Twice the provider address, word |
| Context + 0x0d | First linked address’s low byte |
| Companion + 0x54 | Context address, word |
| Context + 0x10 | Byte 0x72 |
| Companion + 0x54 | Secondary address/value, word |

Only then is first-linked offset 0x64 captured as the **output-source word**. Feasible overlap between independently obtained bases can make earlier stores affect this capture.

The **arithmetic value** used subsequently is twice the provider address minus 0x6f. Its represented integer width is 32 bits. Overflow and executable address-arithmetic semantics are unresolved; unlimited-precision arithmetic is not an established substitute.

### 2. Output publication

Before the five output stores, the following stores occur in order:

1. Secondary offset 0x54 receives the output address as a word.
2. Context offset 0x10 receives word 0x72.
3. Companion offset 0x14 receives word 0x72.
4. Context offset 0x14 receives the arithmetic value as a word.
5. Context offset 0x0d receives the first linked address’s low byte.
6. Companion offset 0x14 receives the context address as a word.
7. Context offset 0x0c receives byte 0x72.
8. Context offset 0x0d again receives the first linked address’s low byte.

The output then receives five word stores in increasing offset order:

| Output offset | Value |
|---:|---|
| 0 | 0x00000072 |
| 4 | Captured output-source word |
| 8 | Context address |
| 12 | First linked address |
| 16 | Companion address |

Publication is not a completion boundary.

Context offset 0x78 is captured as the **context payload**. Subsequent stores write, in order: the first linked address’s low byte to context offset 0x0d; the context address to companion offset 0x54; byte 0x2c to context offset 8; the first linked address to context offset 0x74; and the output-source word’s low byte to first-linked offset 0x0d.

### 3. Traversal and text-associated publication

Companion offset 0x74 supplies the **traversal address**. Traversal offset 0x40 receives word 0x2c. Companion offset 9 receives the traversal address’s low byte.

Two consecutive word reads establish the **nested address**: traversal offset 0x14 supplies an intermediate address, whose offset 0x74 supplies the nested address.

Next, traversal offset 0x0c receives byte 0x65. Traversal offset 0x14 is freshly read, and that word’s low byte is written to traversal offset 9. Traversal offset 8 then receives byte 0x65. Only afterward is traversal offset 0x54 read as the **display address**.

These traversal-local byte stores do not overlap the display-address field under ordinary memory semantics. Their order and the separate offset-0x14 read nevertheless remain part of the represented behavior.

Display offset 0x19 receives the arithmetic value’s low byte. Display offset 0x44 receives the first text address as a word. Traversal offset 0x74 then supplies the **first working address**.

Nested offset 0x11 receives the context payload’s low byte. First-working offset 0x34 receives the first text address. The text-associated record receives the first text address at offset 0 and its own address at offset 8.

First-working offset 0x78 is now captured as the **working payload**. Only afterward does the record receive the secondary address/value at offset 4, the display address at offset 12, and the nested address at offset 16, all as words.

First-working offset 0 receives byte 0x6f. Display offset 0x44 is replaced by the second text address. Nested offset 0x54 receives the working payload; first-working offset 0x54 receives its own base address; nested offset 0x64 receives the working payload again.

### 4. Chain construction and dynamic displacement

First-working offset 0x14 supplies the **chain address**. Nested offset 0x14 receives the first working address. Secondary offset 9 receives byte 0x72.

Chain offset 0x54 supplies the **chain-side target**. That target’s offset 0x54 receives the chain address. Chain offset 0x15 receives the chain address’s low byte.

First-working offset 0 receives byte 0x61, then offset 9 receives the chain-side target’s low byte. Chain offset 0x54 receives the first working address and is immediately read back into a captured word.

A fresh word read at chain offset 0x64 supplies the low byte written to first-working offset 9. First-working offset 0x70 is captured as the **working displacement**, whose low byte is written to first-working offset 0x10. Chain offset 0x54 receives the previously captured read-back word.

A separate read of chain offset 0x64 establishes the **captured chain word**. A word containing the working displacement is then stored at first-working base plus that captured displacement.

Chain offset 0x14 is captured as the **former chain link**, then replaced with the chain address itself. A fresh word from first-working offset 0x70 is written to first-working offset 0x20. Chain offset 0x11 is cleared. Former-chain-link offset 0x54 receives its own base address.

Chain offset 9 receives byte 0x6f. Chain offset 0x44 then receives the second text address as a word. Finally, chain offset 9 is read as a byte and zero-extended to form the **byte-derived base**.

Under sequential ordinary-memory semantics, this read observes 0x6f, so the byte-derived base is 0x0000006f. The intervening word store covers offsets 0x44–0x47, not offset 9. General cross-base aliasing does not change this conclusion: these operations use the same captured chain base, with no intervening store through another base.

### 5. Byte-derived addressing and working-address replacement

The captured chain word’s low byte is written to byte-derived-base offset 9. The second text address is stored as a word at byte-derived-base offset 0x6f. The captured chain word’s low byte then goes to first-working offset 0x15.

Chain offset 0x14 receives the third text address. First-working offset 0 receives byte 0x6f, and offset 9 receives the captured chain word’s low byte. Chain offset 0x54 receives the former chain link.

Chain offset 0x74 supplies the **early chain target**. Before that target is modified, byte-derived-base offset 0x14 receives the byte-derived base as a word, former-chain-link offset 0x54 again receives its own base address, and chain offset 0x15 receives the captured chain word’s low byte.

The early chain target receives, in order:

- The captured byte-derived byte at offset 9.
- Byte 0x61 at offset 8.
- The byte-derived base as a word at offset 0x34.
- The captured byte-derived byte at offset 1.

The captured byte is 0x6f under the ordinary-memory assumptions above.

A fresh read of chain offset 0x74 establishes the **second working address**. It is not assumed equal to the early chain target or first working address.

A byte 0x61 is stored at twice the chain address plus 9. Byte-derived-base offset 0x54 supplies the **byte-base link**, whose offset 0x54 receives the byte-derived base as a word.

The remaining accesses occur in order:

1. Byte-derived-base offset 0x15 receives the chain address’s low byte.
2. Second-working offset 0 receives byte 0x61; offset 9 receives the byte-base link’s low byte.
3. Byte-derived-base offset 0x11 receives the chain address’s low byte.
4. Byte-derived-base offset 0x64 supplies a captured displacement.
5. Chain offset 0x14 receives the byte-derived base as a word.
6. A word containing the displacement is stored at byte-derived base plus that displacement.
7. Byte-derived-base offset 0x44 receives the second text address.
8. Second-working offset 0x64 is read as an address; its offset 0x11 receives the chain address’s low byte.

That final pointer read is not reused for subsequent reads of second-working offset 0x64.

### 6. Later captures and third working binding

A fresh read of chain offset 0x74 establishes the **later chain target**. Second-working offset 0x0c receives byte 0x75. A fresh read of second-working offset 0x64 establishes the **later linked address**. A separate read of chain offset 0x74 captures the **retained chain word**.

The later chain target receives byte 0x75 at offset 9 and the retained chain word at offset 0x54. Later-linked offset 0x54 receives the second working address. Second-working offset 0x34 receives the retained chain word.

Later-linked offset 0x74 is captured as the **later-linked payload**. Later-linked offset 0x14 receives the chain address. Chain offset 0x14 receives word 0x72 and then the later chain target. Second-working offset 0x30 receives word 0x72.

A fresh read of chain offset 0x14 now establishes the **third working address**. This new binding controls subsequent working-region accesses.

Later-linked offset 0x15 receives the chain address’s low byte. Third-working offset 9 receives the later-linked payload’s low byte. Third-working offset 0x10 receives word 0x72, followed by offset 0 receiving byte 0x72.

Chain offset 0x74 is freshly read as the **replacement link**. Third-working offset 0x15 receives the unresolved byte. Chain offset 0x14 receives the replacement link, whose offset 0x54 receives the third working address.

### 7. Final ordered mutations

Chain offset 0x54 is captured as the **saved chain link**. A fresh word from later-linked offset 0x64 supplies the byte written to third-working offset 9. Third-working offset 0x10 then receives byte 0x72, distinct from its earlier word store.

Chain offset 0x54 receives the saved chain link. Later-linked offset 0x64 is freshly read again, establishing the **final byte target**. The two offset-0x64 reads can differ when intervening writes overlap that field through another base.

Saved-chain-link offset 0x54 receives its own address. Chain offset 0x74 is freshly read as the **final chain target**. Later-linked offset 0x14 receives the later linked address itself. Chain offset 9 receives the final chain target’s low byte.

Two word reads establish the **final nested address**: final-chain-target offset 0x14 supplies an intermediate address, whose offset 0x74 supplies the final nested address.

Saved-chain-link offset 0x15 receives the unresolved byte. Both unresolved-byte stores precede the represented definition that replaces that byte holder with the retained chain word’s low byte.

Final-nested offset 1 receives the retained chain word’s low byte—not a fresh chain read or the final chain target’s low byte.

The final operations are ordered:

1. Final-chain-target offset 0x54 receives its own address.
2. Final-nested offset 9 receives the final chain target’s low byte.
3. Final-chain-target offset 0x14 is freshly read; its low byte is written to final-byte-target offset 0x15.
4. Final-chain-target offset 0x14 receives its own address.
5. Final-chain-target offset 0x54 again receives its own address.

The representation ends at an invalid-instruction boundary. An actual trap, handler, continuation, termination, or recovery is not established.

## Side effects

Established effects are memory reads and writes, including writes into text-associated storage and low numeric addresses. No allocation, release, ownership transfer, reference counting, external I/O, callback, or synchronization operation is established.

Captured addresses are reused without represented lifetime management. Self-links and address publication do not establish a data structure or ownership model.

No rollback is shown. Failure to proceed can leave earlier writes visible. The five-word output publication can be partial, and cross-base overlaps can modify output storage before or after those explicit stores.

Whole-function atomicity and concurrent safety are not established. Individual access atomicity, fault behavior, and visibility depend on the execution environment.

## Dependencies

No ordinary callable dependency is identified. The terminal invalid-instruction indication does not establish a callable dependency or handler contract.

Environmental prerequisites include accessible traversal storage, permitted writes to text-associated and low-address storage, supported alignment, and meaningful execution semantics for arithmetic-derived destinations.

No unknown callee or handler is assumed to preserve global state, own memory, or release memory.

## Acceptance examples

These are conditional deductions from static evidence, not runtime tests.

1. **Initial replacement remains observable.** With noninterfering storage, provider word 0x12345678, and initial low byte 0xab, primary offset 1 receives 0xab and later 0x78. Reporting only the final byte omits a represented write.

2. **Cross-base overlap can affect capture.** If context offset 0x10 identifies a byte within first-linked offset 0x64, the byte-0x72 store precedes the output-source capture. The capture reflects preceding writes according to the environment’s byte order, including any other overlapping stores.

3. **Repeated publication is retained.** The provider-byte store to context offset 9 immediately after the context-offset-0x44 read remains required even when it repeats an earlier value.

4. **A feasible display-field overlap uses different bases.** Suppose companion offset 9 identifies the first byte of the word at traversal offset 0x54. The companion-offset-9 store then changes that word before the display-address read, absent other overlapping writes. Traversal’s own offsets 8, 9, and 0x0c do not overlap its offset-0x54 word. The resulting numeric display address depends on byte order and the other three bytes.

5. **Output uses a captured word.** With noninterfering storage and output-source word 0xa1b2c3d4, the five stores publish 0x72, 0xa1b2c3d4, context, first linked, and companion, in that order. Later source-field changes do not refresh output offset 4.

6. **The byte-derived base is fixed under ordinary sequential semantics.** The chain-offset-9 read observes the preceding 0x6f store; the intervening chain-offset-0x44 word store cannot change it. The resulting base is 0x0000006f. Its offset-9 store targets 0x78, its offset-0x6f word store targets 0xde, and its offset-0x14 word store targets 0x83. Offset-0x54 and offset-0x64 reads target 0xc3 and 0xd3. Permission to access these addresses is not established.

7. **Working bindings remain distinct.** If the working-address reads yield different addresses, the offset-0x0c and offset-0x30 writes belong to the second working region. The later offset-0x10 writes and unresolved offset-0x15 write belong to the third.

8. **Final byte sources remain distinct.** Final-nested offset 1 receives the retained chain word’s low byte. The final byte target comes from the later of two separate later-linked-offset-0x64 reads. Neither source may be replaced by another capture.

## Uncertainties

- **Instruction validity — documented_only:** invalid instruction data limits executable interpretation of the entire representation.
- **Calling convention — documented_only:** the additional incoming addresses and unresolved byte lack established interface definitions.
- **Arithmetic — documented_only:** represented integer operations are 32-bit, but signed overflow, address wrapping, and displacement-to-address interpretation are not independently established.
- **Endianness and alignment — documented_only:** complete numeric outcomes of partial word overwrites and support for unaligned accesses are unresolved.
- **Aliasing:** independently obtained bases have no disjointness guarantee. This does not erase the distinct byte ranges of nonoverlapping offsets within one captured base.
- **Environmental mutation — documented_only:** concurrency, externally modified memory, device semantics, and unusual address-space aliasing are not established. They are not grounds for weakening the ordinary-memory conclusion that the byte-derived base is 0x6f.
- **Unresolved byte — documented_only:** its origin, value, and executable treatment remain unknown.
- **Storage and ownership — documented_only:** text association does not establish writability, object identity, ownership, or valid lifetime.
- **Intent and completion — documented_only:** application purpose, successful return, and boundary handling remain unknown.

## Evidence

The sole evidence is the supplied static representation: explicit byte and 32-bit accesses, ordered overwrites, pointer-following reads, arithmetic-derived destinations, five-word output publication, three successive working-address bindings, unresolved byte uses, and a terminal invalid-instruction indication.

Access order and data dependencies are established from that representation. The nonoverlap conclusions follow from the represented offsets under ordinary byte-addressed memory semantics. Application purpose and object interpretations are not established; no inferred application intent is adopted as a requirement.

No runtime tests, executable observations, fault observations, or independent calling-convention verification have been performed.
