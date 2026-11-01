<!-- Trusted provenance: F-cd6a6ecb5056; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cd6a6ecb5056

## Purpose

F-cd6a6ecb5056 measures the consecutive nonzero character-sized storage units beginning at a supplied address. It returns their count, excluding the terminating zero. A null pointer returns zero without accessing sequence storage.

The result is a storage-unit count, not a count of decoded characters, graphemes, or displayed symbols. Nonzero units contribute equally regardless of their textual meaning or encoding.

The inferred intent is a null-tolerant string-length operation. The established behavior does not require textual content: any sequence satisfying the storage, termination, traversal, and arithmetic preconditions has the same counting behavior.

## Interface

### Input

The function accepts one pointer to character storage, passed by value. There is no capacity, maximum scan length, destination buffer, or separate error-output parameter.

Supported input forms include:

- A null pointer.
- A pointer to the first unit of a readable, zero-terminated sequence.
- A pointer to an interior position of such storage, provided the sequence starting at that position independently satisfies the preconditions.
- A pointer directly to a readable zero, representing an empty sequence.

The function advances its invocation-local pointer value. It does not update the pointer object from which an argument may have been obtained.

### Output and widths

The result has a signed integer type. For a valid, stable non-null sequence, it equals the number of consecutive nonzero units before the first zero.

The supplied representation does not establish the exact bit widths of the signed result, pointers, or character units. An eight-bit character unit, a 32-bit result, and any particular pointer width therefore remain unspecified.

Let **M** be the maximum positive value representable by the signed result type. Defined successful counts range from zero through **M**, inclusive, subject to valid storage and traversal.

No separate status result or explicit error code is provided. In particular, a negative result is not an established error convention.

Null input and non-null empty input both return zero. Their return values do not distinguish absence of storage from an empty sequence.

### Layout, lifetime, and ownership

The sequence consists of consecutive character-sized units. Its endpoint is the first zero-valued unit encountered from the supplied position. There is no represented length prefix, capacity field, header, or separate terminator pointer.

For a defined non-null measurement:

- Every unit through and including the first zero must be readable.
- The relevant storage must remain alive throughout the operation.
- Pointer traversal between the examined positions must be valid under the applicable execution model.
- The number of preceding nonzero units must not exceed **M**.
- Access must satisfy applicable concurrency and memory-access rules.

Input storage ownership is unspecified. The storage could be borrowed, static, or owned by another component. The function neither establishes ownership nor performs an allocation, release, ownership transfer, or other explicit ownership-management operation. Valid lifetime and accessibility are requirements regardless of who owns the storage.

## Behavior

### Null input

A null pointer produces zero.

No sequence-storage read occurs on this path. Consequently, the null case does not require an accessible character-storage object.

### Non-null input

The function examines successive character-sized units beginning at the supplied position. Each nonzero unit increases the count by one. The first zero ends the measurement and is excluded from the result.

For stable, valid input containing **n** nonzero units followed by zero, the result is **n**.

Only zero terminates the scan. Nonzero spaces, line endings, punctuation, control values, and high-bit-set values are counted like any other nonzero unit. Interpreting a nonzero unit as signed rather than unsigned does not make it a terminator.

No decoding, normalization, locale processing, encoding validation, or grapheme recognition is represented. If one encoded character occupies several nonzero units, each contributes separately.

An interior pointer measures only the suffix beginning at that position. The function does not search backward, recover the original object’s beginning, or include any preceding content.

### Observable logical access order

For a valid, stable sequence of **n** nonzero units followed by zero, the represented evaluation examines **n + 1** positions: the **n** counted positions and the terminating position.

The examinations occur in increasing sequence order:

- The supplied position is examined first.
- A nonzero position is counted before the pointer advances to its successor.
- That successor is then examined.
- A zero ends examination without another pointer advance.

No logical sequence read before the supplied position or after the terminating zero is represented. Storage after the terminator is not required for this measurement. In particular, the zero may occupy the final readable character position, provided all preceding accesses and traversal are valid.

These are source-level logical access properties. They do not establish physical instruction widths, cache-line activity, speculative accesses, or the number of hardware memory transactions. They do not independently establish suitability for memory-mapped devices or other storage whose reads have special effects.

### Arithmetic limits

The count begins at zero and increases by one for each nonzero unit. It uses a signed integer type; no overflow check or alternative large-result path is represented.

Exactly **M** nonzero units followed by a readable zero can produce **M**. On that path, the count reaches **M**, the pointer advances to the terminator, and the zero ends the scan without a further increment.

If another nonzero unit follows those **M** units, counting that unit would exceed the signed range. Under ordinary source-language signed arithmetic rules, this does not produce a defined wrapped result. The contract therefore does not accept a wrapped negative count, saturation, or an overflow error code as established behavior.

The logical total **n + 1** used to describe examined positions is a mathematical count. It does not imply that the function computes or stores that total in its signed result type.

### Invalid and changing storage

A non-null pointer is not validated merely by passing the null check. If its first unit is unreadable, there is no required successful result or controlled failure.

A readable nonzero prefix is insufficient without a reachable zero and valid traversal. There is no independent length limit or capacity check that stops scanning before the first zero. For an invalid unterminated invocation, neither a particular fault nor indefinite execution is guaranteed.

The function provides no synchronization or snapshot mechanism. If storage changes during measurement, the stable-sequence result rule does not guarantee the length of the initial contents, final contents, or any single intermediate version. Conflicting concurrent accesses may themselves violate the applicable execution rules.

No precedence among invalid storage, invalid traversal, and arithmetic failure is guaranteed for an invocation that violates multiple preconditions.

## Side effects

The represented function performs sequence reads and updates invocation-local counting and traversal state. It performs no explicit write to the measured storage, any external pointer object, or global storage.

There are no represented:

- Allocation, reallocation, or release operations.
- Ownership-management operations.
- Dependency calls or callbacks.
- Input/output operations.
- Locking or synchronization operations.
- Error-indicator updates.
- Cleanup or recovery actions.

The absence of ownership-management operations does not establish who owns the input storage. Nor does it establish that storage remains valid without an external lifetime guarantee.

The absence of explicit external writes is a statement about this function’s represented operations, not every environmental consequence of a memory read. Special-purpose storage may attach effects to reads, and invalid accesses may have environment-dependent consequences.

## Dependencies

No callable dependencies are present. There are no opaque dependency function IDs to enumerate.

No allocator, release routine, string service, callback, or error reporter participates in the represented behavior. No unknown callee must be assumed to preserve globals, retain input, or manage storage lifetime.

The function depends on execution-environment rules for:

- Null-pointer comparison.
- Character-unit access and comparison with zero.
- Pointer advancement and object extent.
- Storage lifetime and readability.
- Signed integer range and overflow.
- Concurrency and unusual memory mappings.

These are environmental constraints, not identified callable dependencies.

## Acceptance examples

The following examples are conditional obligations derived from static evidence, not reports of executed tests. Unless otherwise stated, storage is stable and readable through the first zero, traversal is valid, and the count is representable.

| Input condition | Required result | Logical access requirement |
|---|---:|---|
| Null pointer | 0 | No sequence-storage read. |
| Non-null pointer directly to zero | 0 | Examine that zero; do not examine a successor. |
| Three nonzero units followed by zero | 3 | Examine the three units and then the zero, in order. |
| Two nonzero units, zero, then additional data | 2 | Stop at the first zero; do not examine the later data. |
| Interior pointer with two nonzero units before the next zero | 2 | Measure only the suffix starting at the supplied position. |
| Pointer to the terminator of a longer sequence | 0 | Examine the designated zero without examining preceding content. |
| Four nonzero units jointly encoding one text character, followed by zero | 4 | Count units without decoding the character. |
| Nonzero whitespace and control units followed by zero | Number of nonzero units | Treat those values as content, not additional delimiters. |
| Nonzero high-bit-set units followed by zero | Number of nonzero units | Count each unit regardless of its signed interpretation. |

Additional boundary and preservation criteria:

- **Maximum count:** Exactly **M** nonzero units followed by zero returns **M**, conditional on valid storage and traversal.
- **Excessive count:** More than **M** initial nonzero units is outside the defined counting precondition. No successful result, saturation, wrapping behavior, or controlled error is established.
- **Terminator at an object boundary:** A zero at the final valid character position is sufficient; no subsequent logical sequence read or pointer advance is required.
- **Unreadable first position:** A non-null address with an unreadable first unit has no required successful outcome.
- **Missing accessible terminator:** A finite readable nonzero prefix does not establish a valid bounded measurement.
- **Pointer preservation:** Advancing the local pointer does not modify the external pointer object used to supply the argument.
- **Input preservation:** A defined invocation performs no explicit write to the measured sequence.
- **Ownership independence:** Valid borrowed, static, and externally owned storage receive the same counting behavior. None of these cases requires ownership to belong to the invoker.
- **Absent versus empty:** Null input and a readable initial zero return the same value, but only the non-null case requires a storage read.

No acceptance example prescribes a particular outcome after a lifetime, accessibility, traversal, concurrency, or arithmetic precondition is violated.

## Uncertainties

### Established static evidence

The supplied representation establishes:

- A single character-storage pointer input.
- A signed integer result initialized to zero.
- A null path returning zero without a sequence read.
- Forward character-unit examination until the first zero.
- One count increment per encountered nonzero unit.
- No explicit dependency calls, external writes, or memory-management operations.

The suffix rule, exclusion of the terminator, and different access requirements for null and non-null empty input follow from these properties.

### Inferred intent

The likely intent is to measure a conventional zero-terminated string while treating null as length zero.

This interpretation does not establish a particular encoding, a standard-library identity, ownership of the storage, or the validity of any actual caller’s input.

### documented_only

The following remain unresolved:

- Exact integer, pointer, and character-unit widths and representations.
- Binary calling convention and target ABI details.
- Physical access widths, speculative behavior, and memory transactions.
- Target-specific consequences of invalid access, invalid traversal, or signed overflow.
- The identity of the input storage’s owner and its allocation or lifetime-management policy.
- Whether actual invocations satisfy termination, lifetime, accessibility, and representability requirements.
- Whether intended inputs include concurrently modified or special-purpose storage.
- Whether surrounding application logic distinguishes absent input from an empty sequence despite equal return values.

These uncertainties are not resolved by assuming a familiar platform, conventional ownership, or well-behaved callers.

## Evidence

This contract is based on static inspection of the supplied function representation. Its behavioral evidence is the null check, zero comparison, forward character-unit traversal, signed counting, and absence of explicit calls or external writes.

Ownership is not established by these operations. The supported guarantee is limited to the absence of explicit allocation, release, and ownership-management activity within the function.

Prior prose and reviewer feedback are not independent behavioral evidence. No caller implementations, ABI specification, or executable observations were supplied.

No runtime tests, access measurements, boundary experiments, or overflow experiments have been performed. The acceptance examples describe conditional requirements supported by static analysis, not observed execution results.
