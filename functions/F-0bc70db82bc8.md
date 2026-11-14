<!-- Trusted provenance: F-0bc70db82bc8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0bc70db82bc8

## Purpose

F-0bc70db82bc8 is an externally state-dependent routine whose visible behavior interleaves indirect calls with byte and 32-bit memory accesses. It derives addresses and stored values from entry machine state, the explicit argument, memory contents, and call results.

The established effects include selector-directed byte writes, scaled-index word stores, repeated reads separated by calls, and writes to absolute byte locations `0x0D` and `0x11`. No application-level purpose, object model, or resource-management protocol is established.

This contract covers the **visible static prefix**, conditional on the supplied representation accurately describing executable behavior. The representation reports invalid instruction data and terminates its recovered control flow without showing a normal return. Successful execution or completion is not promised.

## Interface

### Explicit input

One argument is represented as a 32-bit integer, called the **input word**. It is used:

- Three times as arguments to the first indirect call.
- As the basis of an address for a 32-bit read at input word plus four.
- In two identical arguments formed by shifting it left by 19.

The input therefore has both scalar and address-bearing uses. Neither a scalar-only interpretation nor an ordinary-pointer-only interpretation captures the visible interface.

### Implicit inputs and preconditions

The visible behavior also consumes:

- A 32-bit **entry base**, used directly and after a logical right shift by two.
- An **entry byte**, stored before the first call.
- **Entry control state**, used to derive the first indirect-call target.
- Two secondary-register results consumed without ordinary explicit return assignments.
- Memory and executable targets reached through these values and subsequent results.

These are environmental inputs, not established additional parameters of a conventional calling convention.

Reaching any particular operation requires all preceding accesses and calls to permit continuation. The routine shows no validation of address ranges, alignment, target validity, arithmetic overflow, or memory lifetime. Readability, writability, and executable-target validity are therefore conditional preconditions for continuing through the corresponding prefix, not guarantees supplied by this routine.

### Outputs and widths

No ordinary return value or status result is represented. The contract’s outputs are the ordered memory effects and dependency invocations.

Byte accesses are eight bits. Word accesses and the principal address calculations are represented as 32 bits. **Three call results are represented as 64-bit values and split into low and high 32-bit halves:** one during the paired-field updates, one at the end of the linked-word sequence, and one immediately afterward in the final publication sequence.

A **low byte** is the least significant eight bits. Selector bytes used in address calculations are zero-extended. Unsigned right shifts are logical.

Unsigned 32-bit calculations have their represented bit-vector interpretation. Some other calculations involve signed-looking values, casts, left shifts, or multiplication whose executable overflow semantics are not independently established. Where this contract uses a uniform **32-bit wrapping model**, every such result is reduced modulo \(2^{32}\); conclusions relying on that model are conditional.

Byte order, alignment behavior, address-space interpretation, and the actual calling convention remain unresolved.

## Behavior

### Ordering and dependency notation

The phases below preserve the visible order of memory accesses and calls. A **five-call sequence** means one described call followed by four further calls, with the fifth result used where specified. Calls without displayed arguments do not establish that the actual callee receives no usable machine-state inputs.

Dependency invocation sites have opaque identifiers **F-X0001, F-X0002, …**, assigned in visible execution order. These identify sites, not distinct implementations. Every reference to a call or call sequence below denotes those ordered sites.

Calls may change memory or machine state, fault, or fail to return. No preservation, ownership, or release behavior is assumed. Retained values below describe the representation’s visible dataflow, not a general guarantee about callee preservation.

### 1. Entry write and initial selector

Before the first call, the routine reads a byte at entry base plus one. It adds `0x11` to that zero-extended byte and writes the entry byte to the resulting address.

F-X0001 receives three copies of the input word. Four calls follow, producing a result for a later scaled argument.

The routine then reads another byte at the entry base logically shifted right by two, plus one. This is the **initial selector**. It writes zero at initial selector plus `0x11`.

A call receives the preceding five-call sequence’s final result shifted left by two. Four more calls follow, producing another result.

The two entry reads use different addresses. Neither their values nor their destinations may be presumed equal.

### 2. Input-associated read and first snapshot

The routine reads, in order:

1. A 32-bit word at input word plus four: the **input-associated word**.
2. A 32-bit word at initial selector plus `0x74`: the **first snapshot**.

It writes the first snapshot’s low byte at initial selector plus one.

The next call receives, in order:

1. The first snapshot.
2. The input word shifted left by 19.
3. The same shifted input word.
4. The preceding call result shifted left by two.

Another call produces a word. A two-argument call receives that word and the first snapshot logically shifted right by two.

The input-associated word is logically shifted right by 22, yielding a ten-bit **input field**. This extraction does not reread input-associated memory.

### 3. Second snapshot and publication

Four calls follow the two-argument call. Four times their final result becomes the **second base**.

The routine reads a 32-bit **second snapshot** at second base plus `0x74`, then writes its low byte at second base plus one.

A call receives the second base; four calls follow. Secondary-register state from this sequence supplies a **publication index**. The complete second snapshot is stored at four times that index plus `0x14`.

A call receives the publication index. Another call produces a word for a three-argument call. That call receives the word followed by two identical arguments:

\[
(\text{input field}\mathbin{\&}0x1FF)\ll21.
\]

Only nine of the input field’s ten bits contribute to those duplicated arguments.

### 4. Input-field store and first masked index

Four calls follow. Their final result supplies an index. The routine stores the input field shifted left by 23 at four times that index plus `0x14`.

This is an unsigned 32-bit shift: the input field’s highest bit is discarded, leaving contributions from its lower nine bits.

A call receives four times the index. Four more calls produce the **first wide index**, a 32-bit quantity despite its descriptive name.

Before masking that index, the routine stores four times the first wide index at sixteen times the first wide index plus `0x14`.

It then retains the first wide index’s low 30 bits as the **first masked index**. The first masked index logically shifted right by two is retained as the **saved quarter**.

The source ordering is unmasked calculation and store, then masking. Under the 32-bit wrapping model, removing the upper two bits earlier would nevertheless leave this particular store’s numeric address and value unchanged: multiplication by sixteen and four discards those bits. That arithmetic equivalence does not alter the documented ordering.

### 5. Shared-base and paired-field updates

A call receives the first masked index. Four calls follow, producing the **shared index**.

The routine stores the first masked index at four times the shared index plus `0x14`. A call receives four times the shared index; four more produce the **second wide index**.

The **eightfold base** is eight times the shared index. The routine then:

- Writes the low byte of the eightfold base at first snapshot plus one.
- Stores the second wide index shifted left by two at eightfold base plus `0x34`.

The first snapshot is used here as an address base, not reread from its original location.

A call receives the shifted second wide index. Another call produces a word. Before that word is passed to a further call, the second wide index’s low 30 bits are retained as the **second masked index**.

Four calls after the forwarding call produce the **first split result**, represented as 64 bits.

The routine writes the low byte of the eightfold base at second masked index plus one. It then stores the first split result’s high half at eightfold base plus `0x34`, the same represented address as the earlier word store.

A call receives the first split result’s low half shifted left by two.

### 6. Sixteenfold-base byte writes

Four further calls produce another result. The **sixteenfold base** is sixteen times the shared index.

The routine writes, in order:

- The low byte of the sixteenfold base at `0x11` plus the second masked index logically shifted right by 20.
- The low byte of four times the first wide index at sixteenfold base plus `0x0D`.

A call receives two copies of the latest result shifted left by two. Another call produces a word; a subsequent call receives that word and its logical right shift by three.

Four more calls follow. The routine writes the low byte of the sixteenfold base to absolute location `0x11`, then repeats the byte write at sixteenfold base plus `0x0D`.

The repeated write is observable even if its value and represented destination match the earlier write. Calls intervene.

### 7. Thirty-twofold base and compact selector

A call receives the latest result shifted left by two. Four calls follow, producing the **third wide index**.

The routine stores four times the first wide index at second masked index plus `0x14`, then writes the low byte of twice the second masked index at second masked index plus `0x0D`.

Two quantities are derived from the third wide index:

- The **thirty-twofold base**: thirty-two times that index.
- The **compact selector**: that index masked with `0x07FFFFFF`, then logically shifted right by 17.

The compact selector ranges from zero through `0x3FF`.

Three calls follow. Call their final result the **retained intermediate**. The routine stores the third wide index shifted left by seven at thirty-twofold base plus `0x14`, then writes the low byte of the retained intermediate shifted left by two at thirty-twofold base plus `0x0D`.

A call receives that shifted intermediate; four calls follow.

The routine then writes the low byte of twice the saved quarter at compact selector plus seven.

It reads a byte at thirty-twofold base plus one, and writes the compact selector’s low byte at the observed byte plus `0x11`.

### 8. Selector re-read and absolute byte state

A call receives three copies of the retained intermediate shifted left by 21. These arguments derive from the retained intermediate, not from the last intervening call’s result.

Another call returns the **later word**, which is passed to a further call. Four calls follow.

The routine rereads the byte at thirty-twofold base plus one. This observation is the **late selector**. It writes zero at late selector plus `0x11`.

A call receives the latest sequence result shifted left by two; four calls follow.

The routine writes the low byte of the later word logically shifted right by three at thirty-twofold base plus `0x0D`. It then writes the late selector to absolute byte location `0x0D`.

A call receives, in order, twice the late selector, `0x98`, and `0x260`. Another call follows. A subsequent call receives secondary-register state logically shifted right by three; four calls follow.

Their final result, shifted left by two, is passed to another result-producing call.

The routine writes the previously derived later-word byte at twice the late selector plus `0x0D`, then writes `0xC0` to absolute byte location `0x0D`.

### 9. Linked word accesses

A call receives the latest result shifted left by two. Four calls follow, producing the **link index**.

The routine stores the link index shifted left by two at twice the late selector plus `0x24`. It then reads a 32-bit **link word** at twice the late selector plus `0x74`.

The next call receives, in order:

1. The link word.
2. The link word shifted left by one.
3. The link index shifted left by five.
4. The link word masked with `0x7FFFFFFF`, then logically shifted right by two.

Four calls follow. The routine stores the link index shifted left by five at link word plus `0x24`.

It reads a word at link word plus `0x74` and passes that observation to a call. Four further calls produce the **second split result**, represented as 64 bits.

### 10. Split-result stores and final selectors

The second split result’s high half supplies an index. The routine stores the full 32-bit later word logically shifted right by three at four times that index plus `0x14`. This is not merely the byte previously derived from that word.

The second split result’s low half shifted left by two becomes the **retained scaled result**. A call receives three copies of it, followed by the second split result’s high half logically shifted right by three.

Another call produces the **third split result**, also represented as 64 bits. A call receives its low half, high half, low half again, and high half logically shifted right by three.

Four calls follow, producing a destination index. The retained scaled result is stored at four times this index plus `0x14`.

A call receives four times the destination index. Four calls follow, producing the **final selector word**.

The routine reads a byte at link word plus one. At the observed byte plus `0x11`, it writes the low byte of the final selector word shifted left by two.

Three calls intervene. The routine rereads the byte at link word plus one and writes, through that new selector plus `0x11`, the low byte of:

\[
(\text{final selector word}\mathbin{\&}0x3FFFFFFF)\gg20.
\]

A call receives the preceding three-call sequence’s final result shifted left by two. Four calls follow.

The routine reads a selector byte at `0x1000` times the final selector word plus one under the 32-bit wrapping model, then writes zero at that selector plus `0x11`.

The next four-argument call receives, in order:

1. The final selector word shifted left by five.
2. The same value again.
3. The final selector word shifted left by two.
4. The latest four-call sequence’s final result shifted left by two.

The represented construction derives the first two arguments by shifting the already left-shifted-by-two selector word a further three bits. The combined shift-by-five description is subject to the arithmetic qualifications above.

Four calls follow. The routine repeats the selector read at the same represented address expression and the selector-directed zero write. It does not reuse the earlier selector byte.

A final call receives the latest result shifted left by two. Four further calls precede the invalid-instruction termination marker. No normal return is represented.

## Side effects

Every described store is a candidate externally visible effect. No rollback, transaction boundary, synchronization, or atomicity guarantee is established.

Byte-derived destinations of the form selector plus `0x11` range from `0x11` through `0x110`. Other derived addresses have different ranges; in particular, shifted masked-index destinations are not limited to byte-selector ranges. Small numeric addresses do not imply mapped or writable memory.

Repeated reads remain distinct observations. Intervening calls may change selector bytes, referenced words, or memory mappings. Different access expressions may also alias, including with absolute locations. No disjoint-object assumption is established.

The visible body contains no identifiable allocation, release, ownership transfer, or reference-count protocol. Dependencies may perform such actions, but their behavior is unknown. The contract does not infer that memory remains live across calls.

If an access or dependency prevents continuation, later effects are not required. No recovery status or represented exception-handling path is established.

## Dependencies

F-X0001 and subsequent opaque site IDs denote indirect invocations in their visible order. Repeated sites or different sites may resolve to the same implementation.

Their established roles are limited to receiving displayed arguments, producing consumed ordinary or secondary-register results, and separating the specified memory operations.

The first target depends on entry control state. Later target expressions use fixed control anchors and offsets. Actual identities, target validity, argument completeness, calling convention, memory effects, ownership behavior, and termination are **documented_only** unknowns.

A call without displayed arguments is not characterized as pure, parameterless at machine level, or state-preserving.

## Acceptance examples

These are conditional arithmetic and ordering criteria, not executed tests.

1. **Entry destination.** If the first byte read yields `0xFF`, the entry byte is written at `0x110` before F-X0001, provided that write succeeds.

2. **Input extraction.** For an input-associated word of `0xFFFFFFFF`, the input field is `0x3FF`. The duplicated masked arguments are `0x3FE00000`; the unsigned left-shift-by-23 store value is `0xFF800000`.

3. **Masking equivalence with preserved ordering.** For a first wide index of `0x40000001`, the 32-bit wrapping model gives store address `0x24` and value `0x00000004`. Masking first would yield index `1` and the same address and value. More generally, the removed upper two bits contribute nothing to either wrapped multiplication. The represented store nevertheless precedes masking.

4. **Independent selector observations.** If the thirty-twofold-base selector reads yield `2` and later `7`, the compact-selector byte write targets `0x13`, while the subsequent zero write targets `0x18`.

5. **Absolute overwrite.** If execution reaches both absolute writes to `0x0D`, the late selector is written first and `0xC0` later. The intermediate state is part of the contract.

6. **Second split result.** If that result has low half `3` and high half `5`, its high-half-indexed store targets `0x28`, and the retained scaled result is `12`.

7. **Final call arguments.** If the final selector word is `3` and the latest relevant sequence result is `7`, the final four-argument call receives `96`, `96`, `12`, and `28`, in that order.

8. **Final selector re-read.** If the last two selector reads yield `2` and `7`, their zero writes target `0x13` and `0x18`. The second write must use the second observation.

9. **Faulting prefix.** If an early access prevents continuation, later calls and stores are not required, and no recovery return value is specified.

## Uncertainties

**Established static evidence:** displayed access widths and ordering; masks, shifts, constants, and arguments; repeated selector reads; three split 64-bit results; secondary-register consumption; and absence of a represented normal return.

**Inferred intent:** the pattern resembles publication of scaled values into selected fields. No application semantics or coherent object layout follows from that resemblance.

**Documented_only:**

- Validity and completeness of the recovered instruction stream and control flow.
- Actual calling convention and completeness of the apparent interface.
- Executable semantics of signed-looking overflow and address conversion.
- Byte order, alignment handling, and memory-mapping rules.
- Dependency identities, effects, preservation behavior, and termination.
- Object boundaries, aliasing constraints, ownership, and memory lifetime.
- Meaning of absolute low-address writes.
- Whether execution traps, transfers elsewhere, or continues beyond the visible prefix.

## Evidence

This contract is derived solely from the supplied static representation. Its invalid-instruction warnings limit confidence in executable interpretation. Prior prose and reviewer feedback are not independent behavioral evidence.

No tools, external implementation inspection, runtime execution, or runtime tests have been performed. Acceptance examples express conditional consequences of the displayed dataflow and stated arithmetic model, not observed execution results.
