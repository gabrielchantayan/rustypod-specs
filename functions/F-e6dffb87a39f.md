<!-- Trusted provenance: F-e6dffb87a39f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e6dffb87a39f

## Purpose

F-e6dffb87a39f determines whether an unsigned input value occurs in a zero-terminated byte sequence reached through a global base reference.

The terminating zero is not a member of the searchable sequence. Encountering it ends the search with a negative result before an equality comparison can accept it. Consequently, an input of zero does not match the terminator.

The established behavior is a forward, first-match search. Interpreting the sequence as a character set, classification table, or list of permitted values would be inferred intent; none of those application meanings is established.

## Interface

**Input**

One unsigned integer argument is supplied by value. The supplied representation identifies this argument as a 32-bit unsigned value, giving it a nominal range of 0 through 4,294,967,295.

The comparison uses the argument at its represented integer width. It does not establish an operation that first truncates the argument to eight bits. Values outside the byte range therefore cannot equal an ordinary byte read.

**Implicit input**

A global reference supplies the base address of a byte sequence. No length, capacity, ownership token, or explicit sequence pointer is passed as an argument.

The sequence layout is:

- One byte per element.
- Consecutive elements at consecutive byte addresses.
- A zero-valued byte marks the end.
- No header or length field is consulted.
- No bytes following the terminating zero participate in a normal search.

A one-byte sequence containing only zero represents an empty searchable sequence.

**Output**

The supplied representation declares a four-byte return quantity. Its signedness is not established, but every normal return has one of two values:

| Return value | Meaning |
|---|---|
| 1 | A nonzero byte was accepted as equal to the full input value before termination. |
| 0 | A zero byte was encountered before an equality comparison accepted a match. |

There is no separate normal error result. The routine does not return a matching offset, sequence length, pointer, or count.

**Preconditions for a well-defined normal result**

The global reference and every sequence byte reached by the search must be readable. The search must reach either an accepted match or a zero terminator before address or index arithmetic leaves its valid domain.

For the simple interpretation “membership in one sequence,” the global reference and relevant bytes must remain stable during the call. Behavior with changing storage requires the qualifications below.

## Behavior

### Traversal and stopping conditions

The search begins at offset zero and proceeds in increasing byte-offset order. At each visited offset, it first examines the byte for zero.

If that examination yields zero, the function immediately returns 0. It does not perform the equality examination at that offset and does not advance to the next element.

If the zero examination yields a nonzero byte, the supplied representation contains a separate byte examination at the same offset for equality with the argument. An equal value causes an immediate return of 1.

When the equality examination does not match, the offset advances by one and the search repeats. No explicit sequence-length bound appears.

Under stable storage, the function therefore stops at the earliest matching nonzero byte or at the first zero byte, whichever comes first. A successful search does not need to inspect the remainder of the sequence or verify that a later terminator exists.

### Comparison width

The sequence element is represented as an unsigned byte. For comparison, its value is in the range 0 through 255; it is not treated as a negative signed character.

The input remains a full-width unsigned value. Under stable storage:

- Inputs 1 through 255 can match corresponding nonzero bytes.
- Input 0 cannot match because the zero test takes precedence.
- Inputs greater than 255 cannot match any byte.

For example, input 257 is not equivalent to input 1. A stored byte with value 1 can match the latter but not the former.

An out-of-range input does not cause an immediate range rejection. The function still traverses the sequence until it observes a terminator. This distinction matters if storage is invalid or lacks a reachable zero: a mathematically impossible match does not make the traversal safe.

### Read ordering

The supplied representation establishes this logical examination order at each visited position:

1. Examine the current byte for zero.
2. Only after a nonzero observation, examine the byte at that position for equality.
3. Only after an unequal observation, advance the offset.

The routine contains no sequence writes.

These are source-level observations, not a guarantee of a particular number of hardware loads. The supplied material does not establish volatile access, atomic access, a memory model, or the exact machine instructions.

### Changing storage

The routine establishes neither a snapshot nor synchronization around the sequence.

If an execution permits the two byte examinations to observe different values, the equality decision need not use the value that passed the zero test. For example, a nonzero first observation followed by a zero equality observation could accept input zero. That possibility does not contradict the stable-storage rule; it violates the stability condition on which that rule depends.

The source also refers to the global base expression at both examination sites. It does not establish that the base is captured once for the entire call. If external changes to that reference are possible, the routine does not itself guarantee traversal of one consistent allocation.

Whether concurrent changes have defined execution semantics is unknown. These observations describe the absence of a snapshot guarantee, not a supported concurrency interface.

### Index arithmetic

The traversal offset is represented as a signed integer, nominally 32 bits in the supplied representation. It starts at zero and advances by one for each visited nonmatching, nonzero position.

No overflow check is shown. A contract for normal execution must therefore require termination before offset progression or effective-address formation becomes invalid. The function must not be specified as safely searching an arbitrarily large unterminated region.

The byte-offset layout establishes unit-stride traversal. Pointer width, address-space wraparound behavior, and an exact maximum safe allocation size are not established.

## Side effects

The established explicit effects are reads of the global base reference and sequence storage, followed by a scalar return.

The supplied representation contains:

- No writes to the sequence.
- No assignment to the global base reference.
- No allocation or release.
- No ownership transfer.
- No dependency calls.
- No explicit I/O, logging, or error-state update.

Ownership of the referenced storage remains outside this routine. The function requires readable access but neither acquires nor relinquishes responsibility for the storage.

No null-reference check or bounds check is shown. An invalid base, inaccessible element, or invalid address calculation has no specified recovery path. The resulting platform behavior is documented_only; it must not be represented as a guaranteed return of 0.

Likewise, a missing terminator does not have a defined “not found” result unless a match is reached first. Continued traversal may encounter invalid storage or arithmetic limits.

## Dependencies

No callable dependencies appear in the supplied function body. No additional opaque function IDs are therefore required.

The routine depends on three pieces of external state or execution context:

| Dependency | Required property |
|---|---|
| Global base reference | Supplies a usable address for each access that is reached. |
| Byte-sequence storage | Remains readable through the stopping position. Stability is required for ordinary membership semantics. |
| Execution environment | Supports the represented byte reads, integer comparisons, offset progression, and return convention. |

The initializer, owner, lifetime controller, and permitted mutators of the global reference are unknown. No preservation or memory-management behavior is attributed to an unseen helper.

## Acceptance examples

Unless stated otherwise, these examples assume stable, readable storage and valid arithmetic. Bracketed values denote successive byte values, not implementation syntax.

| Sequence beginning | Input | Required result and stopping behavior |
|---|---:|---|
| [0] | 65 | Return 0 at the initial zero examination. |
| [0] | 0 | Return 0; the terminator is not accepted as a match. |
| [65, 0] | 65 | Return 1 at offset zero, without examining the terminator. |
| [65, 66, 0] | 66 | Pass the first byte and return 1 at offset one. |
| [65, 66, 0] | 67 | Return 0 upon reaching the terminator. |
| [65, 0, 66] | 66 | Return 0; the byte after the terminator is not searched. |
| [255, 0] | 255 | Return 1; the byte is compared as value 255. |
| [1, 0] | 257 | Return 0; the input is not reduced to its low byte. |
| [65, 66, 0] | 0 | Return 0 at the terminator, with no successful comparison. |

Additional conditional cases:

- If the initial readable byte equals a nonzero input, normal success requires no readable byte after that position. The routine does not inspect later storage.
- If duplicate matching bytes occur before the terminator, the earliest one ends the search. Later duplicates are irrelevant.
- If a readable region contains neither the requested value nor a zero, no normal outcome is specified beyond that region. Failure must not be inferred merely from the absence of a match within the allocation.
- If an input exceeds 255 and a reachable terminator exists, the result is 0. The routine still visits preceding positions rather than rejecting solely from the input range.

## Uncertainties

The following remain documented_only:

- The global sequence’s contents, initializer, capacity, and application meaning.
- The global reference’s ownership, lifetime, and mutability rules.
- The exact ABI, pointer width, and independent confirmation of the nominal integer widths.
- Exact physical load counts and caching of the global reference.
- Defined behavior, if any, under concurrent mutation.
- Fault handling for invalid addresses and behavior after arithmetic overflow.
- Guarantees supplied by callers about termination or accessible storage.

These gaps do not justify adding validation, synchronization, truncation, bounded scanning, or recovery behavior to the contract.

## Evidence

This contract is based solely on static inspection of the supplied representation of F-e6dffb87a39f.

Established evidence includes the zero-initialized traversal offset, zero check before equality, separate byte-access expressions, unit offset increments, full argument comparison, two constant return values, and absence of explicit writes or dependency calls.

The description as a zero-terminated membership search is an inferred functional interpretation of that control flow. Application-specific purpose and external storage guarantees are not established.

No runtime tests have been performed. The acceptance examples are conditional consequences of the static evidence, not observed execution results.
