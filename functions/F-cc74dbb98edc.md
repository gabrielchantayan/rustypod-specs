<!-- Trusted provenance: F-cc74dbb98edc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cc74dbb98edc

## Purpose

F-cc74dbb98edc is represented as a non-returning sequence of indirect dependency calls interleaved with byte-sized and 32-bit memory accesses. After five initial calls, execution repeatedly traverses one fixed cycle.

The supplied representation establishes the arithmetic, ordering, and explicit memory effects described below. It does not establish a higher-level purpose such as device initialization, allocation, serialization, or queue processing. In particular, the small numeric addresses and repeated offsets do not, by themselves, establish a conventional object layout.

This contract describes the visible sequence conditional on dependencies returning normally and memory accesses completing. It does not promise progress, memory safety, or preservation of state across opaque calls.

## Interface

### Explicit and implicit inputs

The apparent interface has no declared parameters and no returned value. It nevertheless consumes incoming execution state:

- An incoming control-link value participates in calculating the first indirect call target.
- An incoming 32-bit address-like value supplies the base for the first selector-byte read.
- An incoming 32-bit word supplies the first cycle’s opening extracted byte.
- Another incoming 32-bit word supplies the first cycle’s opening shifted dependency argument.
- A secondary result associated with a later dependency call is passed to another dependency. Its exact calling-convention meaning is unresolved.
- One externally stored 32-bit word is read during each cycle.

The entry state is therefore not fully specified by the apparent parameterless signature. Calling this function using an ordinary no-argument convention is not established as sufficient.

### Width and arithmetic model

The representation uses:

- 8-bit byte reads and writes.
- 32-bit words, addresses, integer results, and ordinary call arguments.
- Some 64-bit dependency results, explicitly separated into low and high 32-bit halves.

For describing the apparent machine behavior, multiplication, left shifts, and address additions below retain the low 32 bits. Right shifts of unsigned words are logical. A byte write retains only the low eight bits.

This is a machine-width interpretation, not a claim that every displayed signed arithmetic operation would be defined in a portable source language. Overflow, alignment, and instruction-level behavior require independent confirmation.

The following notation describes observable accesses:

- **Selector write at base B:** read one unsigned byte at address B + 1, add 17 to that byte’s numeric value, and write one byte at the resulting address.
- **Word slot at X:** the address X + 20, used for a 32-bit read or write.
- **Scaled slot:** a word slot whose X is an explicitly stated multiple of a dependency result.

A selector byte is a numeric address component, not an established pointer or object index. Its destination lies between addresses 17 and 272 inclusive.

### Preconditions and completion

For uninterrupted traversal:

- Every reached indirect target must be executable and compatible with the actual machine calling convention.
- Every explicit read and write must be permitted at its computed address.
- Every dependency must return.
- Required incoming machine state must be meaningful.

No checks enforce these conditions.

There is no ordinary successful return, termination flag, or explicit error result. A dependency that does not return, an invalid control transfer, or a failing memory access can prevent later events.

## Behavior

### Dependency notation

The IDs below identify opaque call occurrences or ordered groups, not resolved implementations. Within a group, a numeric suffix identifies an individual call; for example, F-D03-02 is the second call in F-D03.

Different IDs do not imply different underlying functions. Target aliasing and dependency behavior are unknown. Calls shown without arguments have no explicit arguments in the supplied representation; this is not proof that they consume no machine state.

All ordering below is conditional on preceding calls returning and preceding accesses completing.

### 1. Entry and cycle-carried state

Five indirect calls occur before the loop, identified in order as F-D00-01 through F-D00-05. The first target depends on the incoming control link. The other four use internally formed targets. The fifth call’s 32-bit result becomes the cycle’s **entry word**.

For the first cycle, retain the incoming selector base, extraction word, and shift word described in the interface. Later cycles replace these with values explicitly identified in phase 13.

The initial five calls are not repeated by the visible loop.

### 2. Opening selector update and paired-result forwarding

Form the **entry scale** as four times the entry word.

Read the selector byte at the current selector base plus 1. Write the low byte of bits 20–29 of the current extraction word to that selector’s destination. Because the extracted quantity is ten bits wide, this byte store retains bits 20–27 only.

Then perform:

1. F-D01-01 with the current shift word shifted left by 12.
2. F-D01-02, producing a 32-bit result.
3. F-D01-03 with that result and the entry scale shifted left by 28.
4. F-D01-04, producing a 64-bit result.
5. F-D01-05 with its low 32 bits first and high 32 bits second.
6. F-D01-06.
7. F-D01-07, producing a 32-bit result.

Re-read the selector byte at the current selector base plus 1 and write zero to its destination. This is a fresh read; the earlier selector value is not reused.

### 3. Initial word publication

F-D02 contains five ordered calls. Its first receives four times the last result from phase 2. Its fifth produces a new word.

Let the **first publication value** be four times that new word. Store it at the word slot whose X is 4096 times the entry scale.

F-D03 contains five ordered calls:

- Its first receives four arguments: 1024 times the entry scale, the same value again, the entry word shifted left by 14, and the first publication value.
- Its fifth returns a word.

Store the first publication value at the word slot whose X is four times that returned word.

F-D04 contains five ordered calls. Its first receives that X. Its fifth returns a word.

Let the **second publication value** be four times this latest result. Store it at the word slot whose X is sixteen times that result.

### 4. Repeated publication and direct byte clearing

F-D05 contains five ordered calls. Its first receives the second publication value three times. Its fifth returns a word. Store the second publication value at the word slot whose X is four times that returned word.

F-D06 contains five ordered calls. Its first receives that X. Its fifth returns a word.

Let the **third publication value** be four times this result. Store it at the word slot whose X is sixteen times the result. Also form the **direct byte base** as the result multiplied by 2³⁰.

F-D07 contains five ordered calls. Its first receives the third publication value three times. Its fifth returns a word. Store the third publication value at the word slot whose X is four times that result. Retain this result as the **parity source**.

F-D08 contains five ordered calls. Its first receives that X. Its fifth returns an unsigned word, retained as the **lookup source**.

Write zero directly to the direct byte base plus 17, then to the direct byte base plus 1. These are direct writes, not selector writes.

Under 32-bit arithmetic, the direct byte base depends only on the low two bits of its originating result.

### 5. Additional clearing and table read

Perform F-D09-01 with the lookup source shifted left by two. F-D09-02 returns a word, which is passed to F-D09-03. Four further calls follow, F-D09-04 through F-D09-07; the last returns a word.

Again write zero directly to the direct byte base plus 17, then to the direct byte base plus 1.

F-D10 contains five ordered calls. Its first receives four times the last result. Its fifth returns a word.

Retain four times this result as the **later call argument**. At the word slot whose X is sixteen times the result, store the parity source shifted left by 31. Only the parity source’s least significant bit can affect this stored word.

Next, read a 32-bit word from address:

- 20 plus bits 20–29 of the lookup source.

Retain this as the **table word**. This is an address increment of one per extracted value, not a four-byte table index. The computed address ranges from 20 through 1043 and need not be word-aligned.

### 6. Table-derived argument and zero publication

F-D11 contains seven calls:

1. Receive the later call argument.
2. Produce the **retained payload word**.
3. Receive the table word shifted right by 22, then left by 28.
4. No explicit arguments.
5. No explicit arguments.
6. No explicit arguments.
7. Produce a word.

Store a 32-bit zero at the word slot whose X is four times the seventh result.

The argument to the third call depends only on bits 22–25 of the table word under 32-bit truncation.

F-D12 contains five calls. Its first receives that X. Its fifth returns the **selector payload**.

### 7. Selector writes using the direct byte base

Read the byte at the direct byte base plus 1. Write the low byte of four times the selector payload to the selected destination.

F-D13 contains seven calls:

1. Receive four times the selector payload.
2. Produce a word.
3. Receive that word and the selector payload shifted left by 30.
4–6. No explicit arguments.
7. Produce a word.

Read the selector byte at the direct byte base plus 1 again. Retain this particular byte as the **saved selector**. Write the low byte of bits 20–29 of the selector payload to its destination.

F-D14 contains five calls. Its first receives four times the seventh result from F-D13.

After all five calls, store four times the retained payload word at address saved selector plus 20.

The saved selector is deliberately reused after these calls; it is not re-read for this word store.

### 8. External word read and paired result

Read the external 32-bit global word and retain this observation as the **external snapshot**.

F-D15 contains five calls. Its first receives the later call argument. Its fifth returns a 64-bit result.

Let the **pair scale** be four times the low half of that result.

Read the selector byte at the direct byte base plus 1 and write the low byte of the external snapshot shifted right by 22 to the selected destination. This byte contains original snapshot bits 22–29.

F-D16 contains seven calls:

1. Receive the high half of the 64-bit result.
2. Produce a word.
3. Receive that word and the pair scale shifted left by 28.
4–6. No explicit arguments.
7. Produce a word.

Re-read the selector byte at the direct byte base plus 1 and write zero to its destination.

### 9. Publication based on the paired result

F-D17 contains five calls. Its first receives four times the last result from F-D16. Its fifth returns a word.

F-D18 contains five calls. Its first receives four times that word as both arguments. Its fifth returns another word.

Let the **fourth publication value** be four times this latter result. Store it at the word slot whose X is 4096 times the pair scale.

F-D19 contains five calls. Its first receives the pair scale shifted left by 14. Its fifth returns a word.

Store the fourth publication value at the word slot whose X is four times that word.

F-D20 contains five calls. Its first receives that X. Its fifth returns a 64-bit result.

Let the **next pair scale** be four times the low half of this result.

### 10. Selector writes through two differently scaled bases

Read the selector byte at address 16384 times the pair scale, plus 1. Write the low byte of bits 20–29 of the fourth publication value to its selected destination.

F-D21 contains seven calls:

1. Receive the high half of the latest 64-bit result.
2. Produce a word.
3. Receive that word.
4. Produce another 64-bit result.
5. Receive that result’s low half, then high half.
6. No explicit arguments.
7. Produce a word.

Read the selector byte at address 2²⁸ times the next pair scale, plus 1. Write zero to its selected destination.

F-D22 contains five calls. Its first receives four times the last result. Its fifth returns the **cycle payload source**.

Set the **next selector base** to four times the next pair scale. Read the selector byte at that base plus 1 and write zero to its destination.

These three selector bases are distinct arithmetic expressions; no equivalence is assumed.

### 11. Late dependency sequence

F-D23 contains seven calls:

1. Receive the next pair scale shifted left by 10.
2. Produce a word.
3. Receive that word and the retained payload word shifted left by 28.
4–6. No explicit arguments.
7. Produce a word.

Re-read the selector byte at the next selector base plus 1 and write zero to its destination.

F-D24 contains five calls. Its first receives four times the last result. No explicit result from its fifth call is subsequently used.

### 12. Final publication and secondary result forwarding

Form the **cycle publication value** by shifting the cycle payload source left by 14.

Store it at the word slot whose X is 4096 times four times the cycle payload source.

F-D25 contains seven calls:

1. Receive four times the cycle payload source shifted left by 14.
2. No explicit arguments.
3. Receive the cycle publication value shifted left by 28 and an unresolved secondary result associated with the preceding call.
4–6. No explicit arguments.
7. Produce a word.

Store the cycle publication value at the word slot whose X is four times the seventh result.

The first argument to F-D25-03 is necessarily zero under the stated 32-bit arithmetic: the cycle publication value already has fourteen low zero bits before the additional 28-bit shift.

### 13. Transition to the next cycle

F-D26 contains five calls. Its first receives the X used for the final store. Its fifth returns the next cycle’s entry word.

The next cycle uses:

- The next selector base from phase 10.
- The cycle publication value as both its extraction word and shift word.
- The final internal call anchor when forming its first call target.

The opening call target is therefore not established as identical between the first and subsequent cycles. No exit condition follows.

## Side effects

Explicit effects comprise selector-byte reads, byte writes, word reads, word writes, one external global read per cycle, and all opaque dependency effects.

Important ordering properties:

- Repeated selector reads can observe intervening dependency writes.
- Direct clearing of base plus 1 precedes later selector reads, but opaque calls intervene. A later selector value is not guaranteed to remain zero.
- The saved selector in phase 7 is used after intervening calls without being refreshed.
- The external snapshot is read once at the specified point; earlier calls may affect the value observed.
- No rollback restores earlier stores after a later failure.
- Stores may overlap, including across widths. No disjointness guarantee is established.
- A selector destination can alias the selector byte itself for particular address relationships.

No explicit allocation, deallocation, ownership transfer, lock operation, or cleanup is visible. Dependencies may perform any of these; their absence cannot be assumed. No memory ownership guarantee follows from computing or passing an address-like value.

## Dependencies

F-D00-01 through F-D26-05, with group sizes defined above, denote unresolved indirect-call occurrences.

Their established contract is limited to:

- Execution order.
- Explicit arguments.
- Explicitly consumed primary or paired results.
- Their position relative to explicit memory accesses.

No dependency implementation or independent dependency specification is supplied. Register preservation, global preservation, reentrancy, memory ownership, exceptions, blocking, and nonlocal control transfer are unknown.

Some targets use different fixed arithmetic adjustments from the common pattern. This contract does not collapse them into one presumed helper. Resolving actual target identities requires additional evidence.

## Acceptance examples

These are conditional static acceptance examples, not executed tests.

1. **Opening extraction truncation.** If the opening extraction word is 0x3FF00000, the opening selector write stores 0xFF, not a ten-bit value. The 8-bit destination discards the upper two extracted bits.

2. **Fresh selector observation.** If the opening selector read yields 5, its destination is address 22. If intervening calls change the selector byte to 9, the later clearing write targets address 26, not address 22.

3. **Saved selector behavior.** If phase 7 saves selector value 7, its later word store targets address 27 even if F-D14 changes the selector byte. Whether that unaligned word store succeeds is platform-dependent.

4. **Lookup stride.** If bits 20–29 of the lookup source equal 3, phase 5 reads a word at address 23, not address 32.

5. **Parity publication.** An even parity source produces zero in phase 5’s parity-derived word store. An odd source produces 0x80000000.

6. **Scaled-base wraparound.** If the result forming the direct byte base is 4, multiplication by 2³⁰ yields zero under 32-bit arithmetic. The direct clearing writes then target addresses 17 and 1.

7. **External snapshot extraction.** An external snapshot of 0xFFC00000 produces byte 0xFF for phase 8’s selector write; bits 30–31 are lost when narrowing to a byte.

8. **No progress guarantee.** If F-D11-02 never returns, no subsequent event is required. Earlier writes remain observable; no cleanup path is established.

9. **Repeated cycles.** If every operation completes, F-D26-05 is followed by the next cycle’s opening selector read, not a normal return.

## Uncertainties

**documented_only — execution fidelity.** The supplied representation reports overlapping global symbols and removal of an unreachable block. Original instruction boundaries, all reachable entry points, and the removed region’s significance are not independently established.

**documented_only — machine convention.** Hidden incoming state, secondary results, preserved registers, and call-target interpretation require architecture and calling-convention evidence.

**documented_only — memory model.** Endianness, alignment enforcement, access permissions, memory-mapped device behavior, and concurrent modification are unspecified. Word-level values above do not establish constituent byte order.

**documented_only — arithmetic fidelity.** Low-32-bit arithmetic is the apparent machine-width interpretation. Exact instruction semantics have not been independently checked.

**documented_only — semantic intent.** Repeated writes and scaled addresses may suggest structured state manipulation, but no domain interpretation is established.

**documented_only — errors and ownership.** Dependencies may fail, block, alter globals, release storage, or transfer control. No stronger guarantees are justified.

## Evidence

Established static evidence consists solely of the supplied function representation: a five-call entry sequence, an unconditional loop, ordered indirect calls, explicit byte and word accesses, paired-result forwarding, arithmetic transformations, and cycle-carried assignments.

The phase names and descriptive value names are explanatory labels, not recovered domain meanings. The modulo-width model is an interpretation made explicit above.

No implementation access, independent dependency evidence, instruction-level validation, or runtime tests have been performed.
