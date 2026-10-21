<!-- Trusted provenance: F-3df2c2cdc615; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3df2c2cdc615

## Purpose

F-3df2c2cdc615 searches a zero-terminated sequence of character units and returns a pointer to the last unit equal to a supplied target. The terminating zero participates in the search.

The result is either an address within the supplied sequence, including its terminator, or a null pointer if the target is absent. It is not a copy, an index, a count, or a newly allocated object.

**Established static behavior:** scanning proceeds toward increasing addresses, compares every encountered character with the target, and stops after processing the first zero character.

**Inferred intent:** this is a last-occurrence search over a zero-terminated character sequence. That interpretation does not establish an external library identity, a text encoding, or additional validation requirements.

## Interface

### Inputs

- **Sequence pointer:** a pointer to the first character unit to examine.
- **Target character:** a single value of the same character type as the sequence elements.

The interface has no explicit length, capacity, encoding, allocation, or error-reporting argument.

For defined successful traversal, the sequence pointer must identify a readable character element, and successive elements must remain accessible through a terminating zero. Pointer advancement must also be valid under the execution environment’s object and pointer rules.

A null sequence pointer is not an empty sequence. The function attempts to read its first element without a preliminary null check.

### Output

The function returns a character pointer with one of two meanings:

- **Non-null:** the address of the last element equal to the target among all elements examined, including the first zero.
- **Null:** no examined element equals the target.

The output does not distinguish why a target is absent. It does not report how many elements were examined or how many matches occurred.

### Widths and representation

Each comparison concerns one character storage unit, not a multi-unit code point or a whole string. The supplied representation does not independently establish an eight-bit character width, the signedness of the character type, or the machine pointer width. Those details remain environment-dependent.

Both the read element and the target have the same character type. Equality therefore compares their character values; there is no case folding, locale conversion, encoding normalization, or widening into a separate text representation.

The stopping value is the character value zero. A textual digit representing zero is not a terminator unless its character value is itself zero.

Traversal advances by one character element at a time. No integer index, length counter, or accumulated byte count is part of the established behavior.

## Behavior

### Search boundary

The examined range begins exactly at the supplied pointer and ends at the first zero character, inclusive. The function neither searches backward nor examines elements after that terminator.

A pointer into the middle of a larger allocation establishes a search starting at that interior position. Earlier elements are outside this invocation’s search range, even if they contain matching characters.

### Selection rule

Before examining the sequence, the function has no selected match.

Each encountered character is compared with the target. On equality, its address becomes the selected result, replacing any earlier selected address. A nonmatching character leaves the selected result unchanged.

Consequently:

- A single match produces that match’s address.
- Multiple matches produce the highest-addressed match within the examined range.
- No match produces a null pointer.
- A matching prefix does not cause early return; later characters still determine whether a later match replaces it.

The function stops because it encounters a zero character, not because it finds a match.

### Terminator handling

The terminating character is compared with the target before traversal ends. This order is observable in the returned value.

If the target is zero, the returned pointer identifies the first zero character encountered. Earlier examined characters are necessarily nonzero, so there cannot be an earlier zero match within this invocation’s completed search.

An empty sequence is therefore handled as follows:

- With a zero target, the result is the input pointer.
- With a nonzero target, the result is null.

A zero followed in storage by additional characters still ends the search. Neither matching characters nor additional zero characters beyond it can affect the result.

### Logical access order

The established source-level order is:

1. Read the current character element.
2. Compare that captured value with the target and, if equal, select its address.
3. Advance the traversal pointer by one character element.
4. Decide whether to continue using the character value already read.

The terminating element is read and processed. The traversal pointer is then advanced once, but the next element is not read.

This distinction matters at object boundaries. A terminator in the last element of a valid array permits the traversal pointer to become the one-past position; it does not require an additional readable element after the terminator. Conversely, an invalid pointer increment is not made safe merely because no later dereference occurs.

These are logical source-level accesses. The supplied material does not establish exact generated load instructions, access timing, or synchronization semantics.

### Invalid and changing input

There is no explicit handling for an unreadable pointer, an unterminated accessible region, or invalid pointer arithmetic. Such inputs have no guaranteed result under this contract. In particular, a zero target does not cause the function to manufacture a terminator or return null safely when no terminator is accessible.

The input must remain valid during traversal. Concurrent unsynchronized modification is not assigned a stable search result by the available evidence. The function does not take a snapshot or provide a consistency mechanism.

## Side effects

No writes to the sequence are present in the supplied function. Matching characters, the terminator, and nonmatching characters remain unchanged by this function’s established operations.

Traversal and match selection change only invocation-local state. Advancing the local traversal pointer does not update a caller-owned pointer variable.

The returned pointer is borrowed from the input storage:

- No new allocation is created.
- No ownership is transferred.
- No input storage is released.
- The result remains usable only while the underlying storage and applicable access permissions remain valid.

Although the return type is a character pointer, this function does not establish that the underlying storage is writable. A caller must determine that independently before writing through a returned address.

No explicit global-state access, external I/O, callback, allocation, or deallocation appears in the supplied function. This is a statement about the supplied body, not a claim about unspecified instrumentation or execution-environment behavior.

## Dependencies

No callee dependencies are present in the supplied function. There are therefore no dependent opaque function IDs whose ownership, global-state effects, or failure behavior must be incorporated.

The function relies on environmental primitives: character reads, equality comparison, pointer advancement, and null-pointer representation. Validity of the input storage and arithmetic is a caller/environment precondition, not a service performed by another identified function.

## Acceptance examples

These examples are conditional acceptance criteria derived from static evidence. They are not results of runtime tests. Character labels below denote distinct nonzero character values; positions are counted from the input pointer.

| Input sequence through its first terminator | Target | Required result |
|---|---|---|
| A, B, A, zero | A | Pointer to position 2, not position 0 |
| A, B, C, zero | B | Pointer to position 1 |
| A, B, C, zero | D | Null pointer |
| A, B, C, zero | zero | Pointer to position 3 |
| zero | zero | The input pointer |
| zero | A | Null pointer |

Additional boundary criteria:

- **Storage after the terminator:** Given A, zero, A in a valid larger object, searching for A returns the first element. The later A is outside the search range.
- **Interior start:** Given a pointer to B within A, B, A, zero, searching for A returns the later A. The A preceding the supplied pointer is not considered.
- **Repeated matches:** Given A, A, A, zero, searching for A returns the third A. Finding the first or second A must not end traversal.
- **Final-element terminator:** Given a valid array whose final element is zero, searching may finish at that zero without reading beyond the array.
- **Non-text character values:** For a representable nonzero character value with a set high-order bit, acceptance depends on equality with the supplied target value, not on whether the value is printable or valid in an encoding.
- **Missing terminator:** No acceptance result is specified when traversal cannot reach zero while preserving valid reads and pointer arithmetic.
- **Null input:** No acceptance result is specified. Null input is not required to return null or any diagnostic.

For every defined case, the sequence must remain unchanged by the function.

## Uncertainties

The following are **documented_only** rather than guessed:

- Exact character width, character signedness, pointer width, and ABI argument representation.
- How a caller’s value outside the target character type’s representable range is converted before entry.
- Exact machine instructions, optimizations, timing, and number of physical memory transactions.
- Fault manifestation for invalid addresses, missing termination, or invalid pointer arithmetic.
- Behavior under concurrent modification, memory-mapped I/O, or other access models not established by the supplied interface.
- Whether a particular returned address permits writing; the signature alone does not establish writable storage.

There is no evidence of explicit error recovery, bounds checking, or an auxiliary error channel. None is added to this contract.

## Evidence

The available evidence is the supplied function body associated with F-3df2c2cdc615. It establishes initialization to a null result, forward character reads, replacement of the selected address on equality, comparison of the terminator, and termination based on the value already read.

The search interpretation is inferred intent. Exact platform representation and invalid-input outcomes remain documented uncertainties.

No runtime tests have been performed. The acceptance examples describe required behavior under their stated preconditions, not observed executions.
