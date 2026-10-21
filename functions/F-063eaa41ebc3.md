<!-- Trusted provenance: F-063eaa41ebc3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-063eaa41ebc3

## Purpose

F-063eaa41ebc3 performs a forward, element-by-element transfer of 32-bit words from a source interval to a destination sequence. It returns the destination position reached after processing that interval.

The source interval is controlled by equality between a current source position and an exclusive endpoint. It is not controlled by a byte count, a length parameter, or an ordering comparison. For each processed source position, the destination word is assigned only if the current destination pointer is non-null. Both positions then advance by one 32-bit word.

This is a description of the supplied static evidence, not a claim that the operation is a safe general-purpose copying primitive. In particular:

- A null destination is not a persistent “discard output” mode.
- Overlapping ranges are processed forward, without overlap protection.
- No bounds, alignment, accessibility, or endpoint-reachability checks are present.
- Successful completion depends on the pointer progression reaching the supplied source endpoint.

## Interface

### Inputs

The interface accepts three pointer-valued arguments, each represented as pointing to a 32-bit word:

1. **Source start** — the first source position considered for processing.
2. **Source endpoint** — the exclusive termination position.
3. **Destination start** — the initial destination position.

The word width is established as 32 bits, or four bytes. Each pointer increment advances by one such word, not by one byte.

The supplied evidence does not establish the machine pointer width, calling convention, address-space representation, or alignment rules. A 32-bit pointed-to word does not itself establish a 32-bit pointer representation.

### Output

The result is the final destination pointer.

For an ordinary valid traversal of \(N\) source words, the returned position is \(N\) words, or \(4N\) bytes, beyond the initial destination position. This relation assumes that the relevant pointer progression is defined and representable. It does not imply that wrapping or out-of-object arithmetic is supported.

The result is a position, not a word count, byte count, success flag, or allocation handle.

### Ordinary-use preconditions

For a conventional successful transfer:

- The source endpoint must be reachable from the source start by a finite number of forward, four-byte steps.
- Every source word actually read must be readable.
- Every destination word actually written must be writable.
- Address progression and any one-past positions used by the operation must be valid under the applicable execution model.
- Alignment must satisfy the target’s requirements for the represented word accesses.

These are caller obligations implied by the accesses, not validations implemented by the function.

An empty interval requires no source or destination word access. The evidence nevertheless does not establish a language-level permission to construct or compare arbitrary invalid pointer values.

## Behavior

### Termination and empty intervals

The function first compares the current source position with the supplied endpoint.

If they are equal, it immediately returns the current destination pointer. In this case:

- No source word is read.
- No destination word is written.
- Neither position advances.
- The returned pointer equals the destination input, including when that input is null.

If the positions differ, one iteration is performed.

The termination condition is equality only. The function does not check whether the source start is before the endpoint, whether the remaining byte distance is divisible by four, or whether a valid allocation contains both positions.

### Per-iteration transfer

Each iteration has the following logical order in the supplied representation:

1. Examine whether the current destination pointer is null.
2. If it is non-null, read the 32-bit word at the current source position and assign that word to the current destination position.
3. Advance the destination position by one 32-bit word.
4. Advance the source position by one 32-bit word.
5. Compare the advanced source position with the endpoint before deciding whether another iteration occurs.

The null check is repeated for the current destination on every iteration. It is not limited to the original destination argument.

When the destination is null, the guarded assignment is omitted. That iteration therefore contains no explicit source-word read and no explicit destination-word write. Position advancement still follows.

The evidence establishes this logical access order. It does not establish instruction-level ordering, volatile semantics, atomicity, or memory-order guarantees between threads.

### Word contents and representation

The assignment transfers one complete 32-bit word. There is no explicit:

- Numeric conversion;
- Byte swapping;
- Sign extension;
- Content-dependent filtering;
- Terminator search;
- Checksum or other transformation.

The transfer count depends only on source-position progression and the endpoint comparison. Zero-valued source words do not terminate or shorten the operation.

Endianness is not specified. For ordinary same-representation storage, word assignment preserves the word value; the evidence does not define a separate serialized byte format.

### Overlapping storage

The source is read at the time its iteration executes. The function does not snapshot the source interval before writing, and it does not select a backward traversal when destination storage overlaps later source positions.

Consequently, an earlier destination write can change a word that a later iteration will read.

For example, if the destination begins one word after the source start, the first write replaces the next source word. The next iteration then reads that replacement. Repetition can propagate the first source word throughout the transferred destination region.

If the destination begins before the source start, forward copying may avoid overwriting unread source words for that particular arrangement. This is a consequence of the traversal order, not a general overlap-safety guarantee.

When source and destination positions are identical, each processed word is read and assigned back to the same position. The function still processes the interval and returns the advanced destination position.

### Null destination behavior

A null destination suppresses only the assignment for an iteration whose current destination position is null. It does not cause an early return, and it does not suppress pointer advancement.

The supplied representation increments the null destination by one word after such an iteration. The validity and result of that operation depend on the execution model. Under ordinary language-level pointer rules, arithmetic from a null pointer is not a valid basis for promising successful execution.

Under a separate address-level interpretation in which advancing null produces the address four bytes above zero, a subsequent iteration would see a non-null destination and attempt the guarded transfer there. That interpretation is conditional, not an established portable guarantee.

Consumers must therefore not treat this function as a supported way to count words or skip an entire copy by passing null as the destination.

### Invalid ranges and arithmetic limits

There is no explicit integer counter. Progress occurs through repeated pointer advancement.

The four-byte element width is established, but a maximum element count cannot be derived without pointer-width and execution-model information. In particular, the evidence does not justify assuming:

- A particular address modulus;
- Defined pointer wraparound;
- Eventual endpoint reachability after wraparound;
- A successful traversal of an endpoint behind the source start.

An endpoint not reachable by valid forward word steps does not produce a clean range error in the visible behavior. Execution may instead encounter an invalid access, invalid arithmetic, or continued traversal. The exact outcome is not established.

## Side effects

The explicit externally relevant data effects are reads of source words and writes of destination words, subject to the per-iteration null guard.

The function contains no explicit allocation, deallocation, resource acquisition, resource release, callback, or global-variable access. It does not explicitly retain a pointer after returning.

Ownership of the supplied storage is not transferred by any visible operation. The caller remains responsible for storage lifetime and access permissions. The returned destination position does not carry an ownership guarantee.

Writes occur incrementally. No rollback mechanism is visible. If execution is interrupted after earlier iterations have written destination words, the function provides no explicit restoration of the original contents.

There is no explicit error result, exception handler, or recovery path. Invalid memory accesses may have platform-dependent consequences; the evidence does not establish which fault, signal, exception, or other outcome would occur.

Concurrent modification is outside the established contract. Nothing in the supplied evidence makes the operation atomic or synchronizes it with other accesses.

## Dependencies

No external function calls are visible in the supplied body. There are therefore no dependency function IDs to enumerate.

Execution relies on the surrounding environment’s pointer comparison, pointer advancement, word-load, and word-store behavior. Those environmental rules are not supplied as separate dependency contracts.

No claim about unknown callees preserving global state or managing memory is required: no callee invocation is present in the evidence.

## Acceptance examples

These are conditional acceptance examples derived from static inspection. They have not been executed.

### Empty interval

Given identical source-start and source-endpoint positions, and any destination pointer whose passage through the interface is valid:

- The result equals the original destination pointer.
- No word is read or written.
- A null destination remains null.

The contents of source and destination storage do not affect this result.

### Three-word transfer into separate storage

Given a valid three-word source interval containing 17, 0, and 42, plus a separate writable destination of sufficient capacity:

- Destination words become 17, 0, and 42, in that order.
- The zero word is transferred normally.
- The result points three words beyond the destination start.
- No fourth source word is read and no fourth destination word is written.

### Identical source and destination

Given a valid two-word interval with the destination equal to the source start:

- Each word is assigned its own current value.
- Final word values remain unchanged in ordinary memory.
- The result is the destination advanced by two words.

Unchanged final contents do not imply that accesses were omitted.

### Destination one word after source

Given four writable words initially containing 11, 22, 33, and 44, a source interval covering the first three words, and a destination beginning at the second word:

- The first transfer replaces 22 with 11.
- The next transfer reads that replaced word and replaces 33 with 11.
- The last transfer similarly replaces 44 with 11.
- The final four words are 11, 11, 11, and 11.

This distinguishes forward incremental transfer from snapshot-based copying.

### Destination one word before source

Given four words initially containing 10, 20, 30, and 40, a source interval covering the last three words, and a destination beginning at the first word:

- The final words are 20, 30, 40, and 40.
- Each source word is read before a later write could replace it in this arrangement.
- The result is three words beyond the destination start.

### Nonempty interval with null destination

For a nonempty interval and a null destination, the first iteration omits the word assignment and then reaches destination advancement.

No unconditional successful-return expectation is established. Acceptance of any further outcome requires an explicitly supplied execution model for arithmetic from null. It would be incorrect to require all source words to be read, all writes to be suppressed, or the result to remain null.

## Uncertainties

### Inferred intent

The forward traversal, word assignment, and returned destination position suggest a low-level word-copy helper. This is inferred intent, not additional behavior.

The null guard may reflect an intended special case, but its unconditional subsequent advancement does not establish a safe null-output mode.

### Documented-only unknowns

The following remain unresolved:

- Pointer width and address-space layout;
- Concrete calling convention;
- Alignment requirements and unaligned-access behavior;
- Defined behavior, if any, for null or out-of-range pointer arithmetic;
- Whether nonordinary memory has additional access side effects;
- Instruction-level access widths, ordering, and atomicity;
- Fault reporting and the exact state left by a failing access;
- Any caller-enforced restrictions beyond those visible here.

No guarantee is inferred from these unknowns.

## Evidence

Established static evidence consists of:

- Three pointer inputs referring to 32-bit words;
- A source-position equality comparison controlling repetition;
- A destination-null guard around the word assignment;
- An unconditional destination-word advancement after that guard;
- A source-word advancement before the next termination comparison;
- Return of the final destination position;
- No visible external calls or explicit error-handling path.

The overlap examples and ordinary \(4N\)-byte advancement relation follow from those operations under their stated validity assumptions.

No runtime tests have been performed. Acceptance examples are specification cases, not observed execution results.
