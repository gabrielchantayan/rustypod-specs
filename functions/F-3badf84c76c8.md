<!-- Trusted provenance: F-3badf84c76c8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3badf84c76c8

## Purpose

F-3badf84c76c8 applies an externally supplied byte-translation table to a mutable, zero-terminated byte sequence. The transformation is in place: existing storage receives replacement bytes, and no replacement buffer is produced.

A position is eligible for replacement when its termination check observes a nonzero byte. Its replacement is selected using an unsigned byte value from that position. Processing continues at the following position regardless of whether the replacement itself is zero.

The established behavior is byte translation, not a particular text operation. Case conversion, encoding conversion, normalization, and the treatment of multibyte characters are not established.

## Interface

### Argument and result

The sole argument identifies a readable storage slot containing the starting address of the sequence.

The slot occupies four bytes in the supplied representation. Its contents are interpreted as an address, not as a byte count, capacity, or inline sequence. Only that first slot supplies the starting address.

A zero address stored in a valid slot requests an early return. A null argument does not have the same meaning: the argument must be dereferenced to obtain the stored address, and no argument-validity check is established.

The function has no explicit result value. It provides no length, status, error code, or final traversal address. Observable successful output consists of changes to sequence storage.

### External table input

A global storage location supplies the table base. The function captures that base before obtaining the sequence address from the argument slot. That ordering also applies when the slot contains zero.

The table consists operationally of individually addressable replacement bytes. A complete table covering every possible byte value would contain 256 entries. The function neither receives nor validates an allocation size.

Under stable memory conditions, only entries numbered 1 through 255 can be selected: an observed zero ends traversal before replacement. An empty sequence therefore requires no readable table entry.

### Widths and address interpretation

The following distinctions are material:

- The argument slot holds a four-byte value.
- Sequence elements and table entries are eight-bit bytes.
- Lookup indices are unsigned, so values from 128 through 255 select positive offsets.
- Successive sequence positions are one byte apart.
- The table base is represented as an integer before address formation.

The representation is consistent with a 32-bit address and integer model. Under that model, combining the base with the unsigned index uses unsigned 32-bit arithmetic, with reduction modulo 2³². It is not a checked or widened address calculation. Numeric wraparound does not establish that the resulting address is valid.

The exact target ABI and machine-level address behavior remain documented_only. The four-byte slot alone must not be generalized into an independently verified description of every pointer or integer on the target.

### Caller obligations

For normal finite completion, the caller and environment must provide:

- A readable argument slot.
- A readable, nonzero sequence address, unless the slot requests early return.
- Readable sequence positions through an eventually encountered zero.
- Writable sequence positions whose termination checks observe nonzero values.
- Readable table entries for all selected indices.
- Valid address calculations throughout traversal.

There is no capacity parameter or built-in protection against crossing an allocation boundary.

## Behavior

### Transformation rule

For stable memory with no relevant overlap, the function replaces every byte preceding the first zero with the corresponding table entry. Positions are processed in increasing address order. The terminating zero is examined but not explicitly rewritten, and positions beyond it are not examined.

Each position receives at most one replacement during an ordinary forward traversal. The replacement is not recursively translated. Consequently, if one entry maps to a byte that has another mapping, the second mapping is irrelevant to that visit.

A replacement of zero affects the resulting sequence but does not stop this invocation. Termination depends on the byte observed at the next position reached, not on the value just stored. The final buffer can therefore contain an earlier zero followed by bytes that were nevertheless transformed.

There is no preliminary length calculation and no snapshot of the original sequence or table.

### Observable access ordering

The static representation establishes the following ordering constraints:

| Events | Required order |
|---|---|
| Capture of table base and acquisition of sequence address | Table base first |
| Acquisition of sequence address and null-sequence decision | Slot read first |
| Termination decision and replacement at a position | Termination check first |
| Lookup and replacement at an eligible position | Table-entry read before sequence write |
| Replacement and examination of the following position | Current write first |

A zero sequence address permits no sequence-byte access, table-entry access, or sequence write. It does not avoid the initial table-base and argument-slot reads.

At a nonterminating position, the supplied representation distinguishes the byte read for the termination decision from the byte read for the lookup index. With stable ordinary memory, both yield the same value. They must not be treated as proof of two particular machine instructions, volatile semantics, or an atomic operation.

If the current byte changes between those conceptual reads, the lookup index need not equal the byte that allowed processing to continue. Index zero is possible in that circumstance. This is a limit on the stable-memory rule, not a guarantee that concurrent mutation is supported.

### Captured addresses and live contents

The starting address and table base are captured once. Traversal does not reacquire the starting address from the slot, and later changes to the global base storage do not select a new table for this invocation.

Table contents are different: replacement bytes are obtained when needed. An earlier sequence write can affect a later lookup if the sequence overlaps the table.

Likewise, overlap with the argument slot or global base storage can change those storage bytes. Such changes do not, by themselves, replace the already captured addresses. Thus “not explicitly assigned” does not mean that these regions are protected against modification through aliasing.

### Completion and invalid memory

Normal completion occurs either because the captured sequence address is zero or because a termination check observes a zero byte.

No recoverable error protocol is established. Invalid addresses, inaccessible table entries, missing terminators, unsuitable permissions, or invalid address arithmetic do not produce a specified status result.

If an access fails after earlier replacements have occurred, no rollback is established. The exact fault, language-level consequence, or recovery behavior is documented_only.

## Side effects

The direct data writes are one-byte replacements in sequence storage. A replacement equal to the existing byte still corresponds to a write in the supplied representation; there is no explicit identity-mapping exemption.

Under valid, stable, non-overlapping conditions:

- Every nonterminating position is written once.
- Writes follow increasing sequence addresses.
- The terminating position is not written.
- Storage beyond that position is untouched.
- The argument slot, global base storage, and table are read-only roles for this function.

The last point is conditional on non-overlap. A sequence write may also be a write into storage serving one of those other roles.

No allocation, deallocation, external call, logging, or explicit synchronization is present. No ownership transfer is established. The function directly uses storage supplied by its caller and environment; it neither creates a replacement buffer nor visibly releases the original storage.

Ownership, lifetime management, initialization, and synchronization outside this function remain external responsibilities.

## Dependencies

There are no called-function dependencies in the supplied body. No additional opaque function ID is required.

The data dependencies are:

| Dependency | Behavioral role | Contract boundary |
|---|---|---|
| Argument slot | Supplies the initial sequence address | Must be readable even for null-sequence return |
| Global base storage | Supplies the captured table base | Read before the argument slot; not reacquired |
| Sequence storage | Supplies termination and index bytes; receives replacements | No supplied length or capacity |
| Table storage | Supplies replacement bytes at unsigned byte offsets | Entries are read live; extent is unchecked |

The table’s interpretation, preparation, allocation extent, and actual mutability are not established. A valid nonempty transformation needs readable entries for the indices actually selected, not necessarily evidence of a complete 256-byte allocation. An empty sequence or null sequence address selects no table entry.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests. Except where overlap is the subject, assume stable memory, valid addresses, adequate permissions, and no relevant overlap. Byte sequences and table indices below use hexadecimal notation.

### 1. Null sequence address

The argument refers to a readable four-byte slot containing zero.

Expected behavior: the global table base is read before the slot; the function then returns without accessing sequence bytes or table entries and without writing memory.

A null argument is not an equivalent accepted input.

### 2. Empty sequence

The slot contains a nonzero address, and the first sequence byte is 00.

Expected behavior: the first byte is examined, but no replacement occurs. No table entry is needed, and the terminating byte remains unwritten.

### 3. Ordinary translation

The sequence is 41 42 00. Entry 41 contains 61, and entry 42 contains 62.

Expected result: 61 62 00. The first replacement precedes the second, and the final zero receives no explicit write.

### 4. A replacement introduces an early zero

The sequence is 41 42 00. Entry 41 contains 00, and entry 42 contains 62.

Expected result: 00 62 00. The zero written at the first position does not prevent processing of the second position.

A later zero-terminated consumer may regard the resulting sequence as empty. That consumer’s behavior is outside this contract.

### 5. Translation is not recursive

The sequence is 41 00. Entry 41 contains 42, and entry 42 contains 43.

Expected result: 42 00, not 43 00. The replacement value is not translated again at that position.

### 6. Unsigned high-byte indexing

The sequence is FF 00, and entry FF contains 7E.

Expected result: 7E 00. The selected entry is at offset 255, not at a negative offset.

### 7. Table overlap changes a later lookup

The sequence starts at table offset 41 and initially contains 42 41 00. Consequently, entry 41 initially contains 42, and entry 42 initially contains 41.

Expected result in those sequence positions: 41 41 00.

The first replacement reads entry 42 and writes 41 into entry 41. The second position then selects entry 41 and receives its updated value. An immutable snapshot would instead produce 41 42 00 and would not satisfy this overlapping-storage case.

### 8. Identity translation

The sequence is 41 00, and entry 41 contains 41.

Expected final bytes remain 41 00. The supplied representation nevertheless includes a replacement write at the first position. Exact hardware write transactions are not established.

### 9. No reachable terminator within valid storage

The sequence contains no zero before the end of its readable storage.

No bounded successful result or recoverable error return is promised. The function has no length-based stopping rule. Any writes completed before an invalid access are not covered by a rollback guarantee.

## Uncertainties

The following are **documented_only**:

- The application-level meaning of the translation table.
- Its actual allocation size, initialization procedure, and lifetime.
- The exact target ABI beyond the widths supported by the supplied representation.
- Exact machine access counts and optimization of repeated ordinary-memory reads.
- Whether memory-mapped storage is intended or supported.
- Whether concurrent modification is intended or supported.
- Actual overlap relationships in callers.
- Fault handling and recovery after unsuitable memory access.
- Ownership and synchronization policies outside this function.

**Inferred intent:** the function is intended to translate a mutable zero-terminated byte sequence using a prepared lookup table. This interpretation does not establish character encoding, locale behavior, textual validity, or a guarantee that the output remains a single zero-terminated sequence of the original apparent length.

## Evidence

**Established static evidence:** the supplied representation supports capture of an external table base before a four-byte sequence-address slot is read; a null-sequence early return; bytewise termination checks; unsigned byte-indexed replacement; in-place writes; and forward traversal in one-byte increments.

It also establishes the absence of visible function calls, allocation, release, capacity checks, and an explicit error result. The access structure supports the distinction between captured addresses and live table contents, including the stated overlap consequences.

**Evidence boundary:** this is a source-facing behavioral contract, not a report of machine execution. Width-dependent arithmetic is qualified by the represented data model. Compiler transformations, target-specific faults, concurrency guarantees, and external ownership policies are not inferred from the traversal pattern.

No runtime tests have been performed. All acceptance examples are conditional expectations derived from static evidence.
