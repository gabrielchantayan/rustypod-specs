<!-- Trusted provenance: F-bd060bfc52e1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bd060bfc52e1

## Purpose

F-bd060bfc52e1 constructs several overlapping-capable groups of 32-bit context words around caller-supplied anchors. On the represented continuing path, it:

- Rewrites eleven words ending at the input anchor.
- Obtains two destination anchors and additional context values from lower positions relative to that input.
- Writes four words at the first destination.
- Writes nineteen words at the second destination, including fixed marker values, incoming context, captured data, and derived pointers.

**Inferred intent:** the arrangement resembles construction of saved execution contexts or continuation frames. This interpretation does not establish an ABI, scheduler role, ownership transfer, or permission to execute any stored marker.

**Evidence limitation:** the supplied representation reports invalid instruction data and extensive removal of allegedly unreachable regions. This contract therefore distinguishes the behavior explicitly represented from actual execution semantics that remain unresolved.

## Interface

### Explicit input

The function has one explicit pointer-like input: the **input anchor**.

All offsets below are byte offsets relative to an anchor. A word is **32 bits**, occupying four bytes. Offset zero identifies the word at the anchor; negative offsets identify preceding words.

The represented path accesses the following input-relative locations:

| Offset | Role |
|---:|---|
| 0 through −20, in four-byte steps | Initially captured data |
| 0 through −40, in four-byte steps | First write region |
| −44 | Additional context word |
| −48 | Additional context word |
| −52 | Additional context word |
| −56 | Encoded first-destination pointer |
| −60 | Encoded second-destination pointer |
| −68 | Additional payload word |

The location at −64 is not explicitly read or written through the input anchor. It can nevertheless be affected if a destination aliases it.

### Ambient inputs

The represented behavior also depends on three incoming machine-context values not supplied as explicit parameters:

- A condition-state bit, described here as the **entry condition**.
- A 32-bit **auxiliary context word**.
- A 32-bit **return-link word**.

Their origins and caller obligations are not established. The explicit pointer alone is insufficient to determine the result.

### Outputs

There is no explicit returned value in the supplied interface. On the continuing path, the observable result consists of memory writes followed by a represented return.

Three distinct fixed 32-bit values are stored:

| Role in this contract | Value |
|---|---:|
| Input-region marker | 0x0899e314 |
| First frame marker | 0x0899e330 |
| Second frame marker | 0x0899e338 |

These are data values in this contract. Their interpretation as executable addresses or continuation locations is not established.

### Representation and validity preconditions

The representation treats destination pointers as values read from 32-bit words. It also performs pointer-relative arithmetic in units of four-byte words. A compatible environment must support that representation, or provide a separately justified interpretation.

For the complete normal path:

- Every explicit read must address readable storage.
- Every explicit write must address writable storage.
- Derived addresses must be meaningful in the execution environment.
- Required alignment, address-space rules, and pointer encoding must be satisfied.

No validation of these conditions is represented. Null pointers, inaccessible memory, or unsuitable alignment have no defined recovery behavior here.

## Behavior

### Entry-condition gate

The entry condition is examined before the represented data accesses.

- If the entry condition is set, execution proceeds to the context transformation.
- If it is clear, the representation terminates at an invalid-instruction operation. No input-relative data read or write is shown before that termination.

The second case is not established as an intentional error API. Whether actual execution traps, transfers elsewhere, or differs because of incomplete instruction recovery is **documented_only**.

### Capture phase

Before performing any represented memory write, the function captures nine 32-bit words in this order:

1. Input offset −8.
2. Input offset −12.
3. Input offset −16.
4. Input offset 0.
5. Input offset −4.
6. Input offset −8 again.
7. Input offset −12 again.
8. Input offset −16 again.
9. Input offset −20.

The repeated reads matter to the represented access sequence. Under stable ordinary-memory conditions, each repeated pair has the same value. The evidence does not justify collapsing repeated accesses for memory whose reads have externally visible behavior or whose contents can change independently.

For subsequent descriptions:

- **Early captures** mean the first reads of offsets −8, −12, and −16.
- **Later captures** mean the subsequent reads at offsets 0 through −20.

These are descriptive labels, not additional storage requirements.

### Input-region rewrite

The first write phase proceeds in the table’s order. All values come from the preceding captures or ambient context; they are not obtained by rereading the locations being overwritten.

| Write order | Input-relative destination | Written value |
|---:|---:|---|
| 1 | 0 | Input-region marker |
| 2 | −4 | Later capture from offset 0 |
| 3 | −8 | Later capture from offset −4 |
| 4 | −12 | Later capture from offset −8 |
| 5 | −16 | Later capture from offset −12 |
| 6 | −20 | Later capture from offset −16 |
| 7 | −24 | Later capture from offset −20 |
| 8 | −28 | Auxiliary context word |
| 9 | −32 | Early capture from offset −12 |
| 10 | −36 | Early capture from offset −8 |
| 11 | −40 | Early capture from offset −16 |

This phase is a specific rearrangement, not a general block move. In particular, the last three writes use early captures in a different order from their original locations.

### Reading destination descriptors

After all eleven input-region writes, the function reads six additional 32-bit words in this order:

1. Input offset −44: context word used in the second frame.
2. Input offset −48: context word used in both frame sections.
3. Input offset −52: context word used in both frame sections.
4. Input offset −56: first-destination pointer.
5. Input offset −60: second-destination pointer.
6. Input offset −68: payload word used twice.

All six reads precede writes through either destination pointer.

Consequently, a later destination write may overwrite a descriptor’s backing storage without changing the already captured descriptor value. This does not imply that destination writes are isolated from the input region or from one another.

### First-destination layout

Four writes occur at the first destination, in descending-address order:

| Destination-relative offset | Written value |
|---:|---|
| 0 | Later capture from input offset −20 |
| −4 | Early capture from input offset −12 |
| −8 | Early capture from input offset −8 |
| −12 | Early capture from input offset −16 |

The function also derives an address **16 bytes before the first destination**. It stores that address in the second-destination region. It does not explicitly access the derived address through this pointer during the represented path.

### Second-destination layout

After completing the four first-destination writes, the function writes nineteen words through the second destination. The following table gives their exact order.

| Write order | Destination-relative offset | Written value |
|---:|---:|---|
| 1 | 0 | First frame marker |
| 2 | −4 | Return-link word |
| 3 | −8 | Context read from input offset −48 |
| 4 | −12 | Context read from input offset −52 |
| 5 | −16 | Address 16 bytes before the first destination |
| 6 | −20 | Second-destination address itself |
| 7 | −24 | Later capture from input offset −20 |
| 8 | −28 | Early capture from input offset −12 |
| 9 | −32 | Payload read from input offset −68 |
| 10 | −36 | Second frame marker |
| 11 | −40 | Return-link word |
| 12 | −44 | Context read from input offset −44 |
| 13 | −48 | Context read from input offset −48 |
| 14 | −52 | Context read from input offset −52 |
| 15 | −56 | Address 16 bytes before the first destination |
| 16 | −60 | Address 36 bytes before the second destination |
| 17 | −64 | Early capture from input offset −12 |
| 18 | −68 | Early capture from input offset −8 |
| 19 | −72 | Payload read from input offset −68 |

The address stored at second-destination offset −60 identifies the location receiving the second frame marker. No indirect access through that stored address is represented.

### Completion and aliasing

The represented path returns after the final write. It shows no subsequent reads, validation, cleanup, or dispatch through a stored marker.

The order is significant if storage overlaps:

- Input-region writes occur first.
- First-destination writes occur second.
- Second-destination writes occur last.
- Within each phase, later writes prevail at a shared address.
- Captured values remain the values obtained during their respective read phases, even when their original memory locations are subsequently overwritten.

These rules describe sequential accesses to ordinary storage. They do not establish atomicity, synchronization, or memory-mapped-device semantics.

## Side effects

The continuing path explicitly represents:

- Fifteen 32-bit memory reads: nine initial captures and six descriptor reads.
- Thirty-four 32-bit memory writes: eleven input-region writes, four first-destination writes, and nineteen second-destination writes.
- Two pointer-relative address derivations: subtraction of 16 bytes from the first destination and subtraction of 36 bytes from the second destination.

Read and write counts describe represented operations, not necessarily distinct addresses or physical bus transactions.

There is no represented allocation, deallocation, reference-count update, ownership transfer, or memory release. Supplied storage is modified in place. The evidence does not establish how long embedded pointers or context words must remain valid after return.

There is no explicit rollback. If an access fails after earlier writes complete, restoration of earlier contents is not represented. The exact failure mechanism and externally visible partial state are **documented_only**.

No explicit named global storage is accessed. Pointer aliasing may nevertheless direct writes into shared or global storage. No broader claim about processor-state preservation is justified by the pointer-only interface.

## Dependencies

No identified external function call appears on the continuing path of F-bd060bfc52e1. The supplied dependency information establishes no other opaque function ID.

The invalid-instruction terminal operation has no supplied opaque dependency ID. It is not treated here as a known callable service, exception handler, or memory-management routine.

The behavior depends on:

- The entry condition and incoming context words.
- The environment’s interpretation of 32-bit pointer-bearing words.
- Accessible storage at all effective addresses.
- The environment’s handling of invalid instructions and failed accesses.

No callee preservation or ownership guarantees are assumed.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence. **No runtime tests have been performed.**

### Stable, disjoint storage

Assume the entry condition is set, all storage is valid ordinary memory, no concurrent mutation occurs, and the three write regions do not overlap.

Let the initial input words at offsets 0, −4, −8, −12, −16, and −20 be 10, 20, 30, 40, 50, and 60 respectively. Let the auxiliary context word be 70.

The input region must then contain:

| Offset | Expected value |
|---:|---:|
| 0 | 0x0899e314 |
| −4 | 10 |
| −8 | 20 |
| −12 | 30 |
| −16 | 40 |
| −20 | 50 |
| −24 | 60 |
| −28 | 70 |
| −32 | 40 |
| −36 | 30 |
| −40 | 50 |

The first destination must contain 60, 40, 30, and 50 at offsets 0, −4, −8, and −12 respectively.

### Captured data survives overwriting

Under the same conditions, the input offset −8 is overwritten with 20 during the first phase. Nevertheless, the first-destination word at offset −8 must receive 30, because it uses the earlier capture rather than the overwritten input word.

An implementation that rereads the input there would violate the represented behavior.

### Embedded pointer arithmetic

Assume valid 32-bit address encodings without boundary ambiguity. If the first destination is 0x1000 and the second is 0x2000:

- Second-destination offsets −16 and −56 must receive 0x0ff0.
- Second-destination offset −20 must receive 0x2000.
- Second-destination offset −60 must receive 0x1fdc.

These are byte-address results of four-word and nine-word backward displacements.

### Identical destination anchors

Assume both destination pointers designate the same valid anchor and all required surrounding storage is writable.

The four first-destination writes still occur. The subsequent second-destination writes replace all four words at offsets 0 through −12. Their final values must therefore be the first frame marker, the return-link word, the context from input offset −48, and the context from input offset −52.

### Clear entry condition

If the entry condition is clear, the supplied representation reaches invalid-instruction termination without performing the listed data accesses. Acceptance of a particular hardware exception, process exit, or diagnostic requires evidence not supplied here.

## Uncertainties

- **documented_only — actual control-flow completeness:** invalid instruction data and extensive discarded regions prevent treating the supplied representation as a complete execution model.
- **documented_only — ambient-input ABI:** the source of the entry condition, auxiliary context word, and return-link word is unknown.
- **documented_only — pointer model:** 32-bit pointer-bearing storage is represented, but address-space details, byte order, alignment requirements, and invalid-address handling are not established.
- **documented_only — boundary arithmetic:** behavior when a derived pointer crosses an address-space boundary is unknown; mathematical subtraction must not be promoted to a guaranteed modular-pointer rule.
- **documented_only — context semantics:** the meanings of saved words and fixed markers are unknown.
- **documented_only — concurrency:** no synchronization, volatile-access contract, or atomicity guarantee is established.
- **documented_only — ownership and lifetime:** no storage acquisition, release, transfer, or post-return lifetime rule is established.
- **documented_only — failure and machine-state effects:** exception behavior, partial-write visibility, and preservation of processor state beyond the shown accesses remain unknown.

## Evidence

The supplied static representation supports the access widths, capture order, write order, offsets, fixed values, derived-address displacements, condition gate, and represented normal return described above.

The context-frame interpretation is inferred intent, not established semantics. The warnings accompanying the representation are reasons to limit confidence, not proof of a particular runtime failure.

No implementation access, independent instruction inspection, execution trace, or runtime test was available. The acceptance examples are derived criteria, not observed results.
