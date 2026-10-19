<!-- Trusted provenance: F-5cb57bbb2e0e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5cb57bbb2e0e

## Purpose

F-5cb57bbb2e0e counts the consecutive nonzero character elements beginning at a supplied address, stopping at the first zero-valued element. It returns the count, excluding the terminator.

A null input is handled explicitly and returns zero without accessing character storage. Consequently, a zero result does not distinguish a null input from a non-null input whose first element is zero.

This contract describes the behavior supported by the supplied static representation. It does not establish runtime behavior for inaccessible storage, missing termination, arithmetic overflow, or concurrent mutation.

## Interface

### Input

The function accepts one pointer to character storage.

The input has two supported forms:

- **Null pointer:** no character storage is required.
- **Non-null pointer:** the address begins a contiguous sequence of readable character elements ending in a zero-valued element.

For a non-null input, every element through and including the first terminator must be accessible during the corresponding read. Merely having a valid starting address is insufficient.

There is no length argument, capacity argument, encoding argument, or destination buffer. The function cannot enforce a caller-provided boundary because no such boundary is supplied.

### Output

The result is a signed integer count:

- Null input produces zero.
- A zero-valued first element produces zero.
- Otherwise, the result is the number of consecutive nonzero elements before the first zero.

The count is in character elements, not decoded characters, words, grapheme clusters, or storage bits. Every nonzero element contributes one, regardless of its interpretation in a character encoding.

No distinct error result is defined. Negative results are not established as an error-reporting mechanism.

### Widths and layout

The supplied representation identifies a character-pointer input and a signed integer result, but does not independently establish their machine widths.

The supported layout is a contiguous, zero-terminated sequence. Each successive element is one character-storage unit after the previous element. There is no header, embedded length field, alignment-dependent record layout, or secondary pointer in the visible operation.

The following distinctions matter:

- The terminator is a zero-valued character element, not necessarily a textual numeral.
- Character signedness does not affect termination: both positive and negative nonzero character values continue the scan.
- An eight-bit character width is not independently established.
- A 32-bit signed result is not independently established.
- Pointer width and calling convention are not supplied.

If the result type is a 32-bit signed integer, its largest representable nonnegative count is 2,147,483,647. That is a conditional arithmetic limit, not an assertion that the supplied target uses that width.

## Behavior

### Null input

The function initializes its count to zero and checks whether the input pointer is null.

For a null pointer, it returns zero. It performs no character-storage read and does not require a readable terminator.

Null is therefore a supported input, not an error that is forwarded to another operation.

### Non-null input

The function examines character elements in increasing address order, beginning with the element designated by the supplied pointer.

For each examined element:

- A zero value ends the scan.
- A nonzero value contributes one to the count.
- After counting a nonzero element, the scan advances by one character element.

The zero terminator is read but is not counted. No element after that terminator is read by the visible scan.

For a stable input containing \(N\) nonzero elements followed by zero, and with \(N\) representable in the result type, normal completion returns \(N\). The scan performs \(N+1\) character-value inspections in the supplied representation: one for each counted element and one for the terminator.

These are abstract accesses supported by the representation. They are not a claim about the exact number or width of physical machine loads in a compiled artifact.

### Observable ordering

The established order is:

1. The null check precedes any character read.
2. A character is inspected before it can contribute to the count.
3. A nonzero character contributes to the count before the scan advances to the next element.
4. The next character is inspected only after that advancement.
5. Encountering zero ends the scan without advancing beyond that terminator.

There is no preliminary full-buffer validation and no separate pass to locate the terminator. If an access becomes invalid partway through the scan, earlier character inspections may already have occurred.

### Termination and arithmetic boundaries

Normal completion requires either a null input or a reachable zero terminator in a valid sequence.

A non-null sequence without a reachable terminator has no specified successful result. The visible operation has no maximum scan length, timeout, or alternate stopping condition.

The count increases using signed integer arithmetic. A normal result is supported only while the number of counted elements remains representable. If the sequence would require counting more elements than the maximum signed result value, the supplied material does not establish a valid returned value.

In particular, this contract does not promise:

- Modular wraparound.
- Saturation at the maximum count.
- A negative overflow sentinel.
- An overflow exception.
- Continued scanning after overflow.

If a sequence contains exactly the maximum representable number of nonzero elements followed by zero, no additional increment is needed to inspect that final terminator. Subject to valid storage and pointer traversal, that boundary count remains representable.

## Side effects

### Memory and ownership

The visible function reads the input sequence but does not write to it. It does not allocate, resize, free, retain, or transfer ownership of storage.

The caller remains responsible for the storage lifetime and for making every required element accessible. The function provides no lifetime extension and establishes no synchronization.

The scan advances its own pointer value. This does not update a caller-held pointer object or return the address of the terminator.

### Other state

No explicit global-variable access, external call, I/O operation, or error-state update appears in the supplied representation.

The visible state changes are limited to the function’s count and scan position. This does not establish that arbitrary platform facilities, fault handlers, or instrumentation cannot affect process state during execution.

### Failure conditions

The function contains no explicit recovery for:

- An invalid non-null address.
- A sequence that crosses into inaccessible storage.
- Storage whose lifetime ends during the scan.
- A missing terminator.
- A count exceeding the signed result range.

No particular fault, signal, diagnostic, or return value is established for those conditions. They are outside the supported normal-completion contract.

Concurrent mutation is also outside that contract. The supplied operation provides no snapshot semantics or locking; it does not establish which sequence of values would be observed under unsynchronized changes.

## Dependencies

No opaque function dependency is present in the supplied representation. F-5cb57bbb2e0e performs the visible scan without calling another identified function.

Its environmental prerequisites are:

- A representation of null pointers.
- Readable character storage for non-null inputs.
- Valid traversal of the sequence through its terminator.
- Signed integer arithmetic sufficient to represent the count.

These prerequisites are not helper-function contracts. No assumption about an unknown callee preserving globals, retaining memory, or releasing memory is needed, because no such callee is shown.

## Acceptance examples

These examples are conditional acceptance criteria derived from static evidence. They have not been executed.

| Condition | Required normal behavior |
|---|---|
| The supplied pointer is null. | Return zero without reading character storage. |
| A valid non-null pointer designates a zero-valued first element. | Read that element and return zero; do not inspect the following element. |
| Three nonzero elements are followed by zero. | Return three; inspect the three elements and the terminator in address order. |
| Two nonzero elements are followed by zero, with additional nonzero data afterward. | Return two; the trailing data does not contribute and is not reached by the visible scan. |
| A valid sequence contains nonzero elements representing spaces, punctuation, or line breaks. | Count each element; none receives special treatment. |
| A multielement encoding uses several nonzero character elements for one displayed character. | Count the elements separately, not the displayed character. |
| The character type is signed and a sequence element has a negative nonzero value. | Continue scanning and count that element. |
| A valid terminated sequence has a length equal to the signed result maximum. | Return that maximum, provided traversal and all reads remain valid. |

The following conditions do not support a required successful result:

- A readable initial region contains no zero before the next inaccessible address.
- A zero exists somewhere in memory but cannot be reached by valid traversal of the input sequence.
- The sequence length exceeds the signed result range.
- Another execution context changes or releases the storage without an applicable synchronization guarantee.

An implementation that substitutes a character-decoding count, includes the terminator in the result, writes a terminator, or rejects null instead of returning zero would not match the established behavior.

## Uncertainties

### Established static evidence

The supplied representation supports an explicit null branch, a zero-terminated forward scan, one count increment per nonzero element, and a signed integer return. It shows no explicit input writes or external calls.

### Inferred intent

The operation appears intended to measure the length of a zero-terminated character sequence while treating null as empty. That intent does not add validation, encoding awareness, bounded scanning, or ownership guarantees.

### documented_only

The following remain unresolved:

- Exact character, integer, and pointer widths.
- Calling convention and binary interface details.
- Physical memory-load widths and optimization behavior.
- Target behavior on signed overflow.
- Fault behavior for invalid accesses.
- Behavior under concurrent mutation.
- Any caller-level constraints narrower than the prerequisites stated here.

These unknowns must not be replaced with conventional platform assumptions without additional evidence.

## Evidence

This contract is based solely on the supplied static function representation. That representation shows count initialization, a null check, sequential character inspection, advancement after nonzero elements, and return of the accumulated count.

No runtime tests have been performed. The acceptance examples are specification conditions, not observed execution results. No independent binary, platform ABI, caller behavior, or external dependency implementation was available to establish the documented uncertainties.
