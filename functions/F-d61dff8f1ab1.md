<!-- Trusted provenance: F-d61dff8f1ab1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d61dff8f1ab1

## Purpose

F-d61dff8f1ab1 is represented as a nonterminating sequence of indirect calls, byte accesses, and 32-bit memory accesses. It repeatedly transforms values returned by unresolved dependencies, writes those values to computed addresses, and carries selected results into the next iteration.

No application-level purpose is established. In particular, the evidence does not establish that the computed addresses denote ordinary objects, device registers, descriptors, or allocated storage. Small integer values loaded from memory are used directly as address bases; they are not shown being translated through an object table.

This contract describes the supplied static representation. Its unusual control transfers, hidden entry state, and arithmetic require qualifications documented below. It does not establish that the representation is an accurate executable reconstruction.

## Interface

### Explicit inputs

| Input | Represented width | Established use |
|---|---:|---|
| First argument | 32 bits | Passed to the first indirect call. Replaced near each iteration boundary with four times a dependency result. |
| Second argument | 32 bits | No explicit use appears in the supplied body. |
| Third argument | 32 bits | Shifted left by 28 for an early call. Replaced later by a dependency result; that replacement is stored after multiplication by four and persists into subsequent iterations. |
| Fourth argument | 32 bits, unsigned | Supplies an early byte write through bit extraction. Replaced near the iteration boundary by four times a dependency result. |

There is no represented normal return and no defined return value.

### Additional required entry state

The explicit arguments are insufficient to determine behavior:

- An incoming control-address value determines the first indirect-call target.
- Another incoming machine-state value determines the source address of the first byte read.
- An additional result-register value is later supplied as a call argument without an ordinary defining assignment.
- A 32-bit global value is read once per represented iteration.
- All indirect-call targets, accessed memory, and dependency side effects affect execution.

These are operational dependencies, not additional verified public parameters.

### Width and address notation

For the descriptions below:

- “Low word” and “high word” mean the lower and upper 32 bits of a represented 64-bit dependency result.
- Byte reads are unsigned eight-bit reads.
- Byte writes retain only the lowest eight bits.
- Word reads and writes transfer 32 bits.
- Scaled values and shifted values are described using their low 32-bit bit patterns.

The last convention makes the intended machine-width effects explicit. It is not a claim that every signed operation in the supplied representation has portable language semantics. Signed overflow and left shifts need confirmation against the underlying execution model.

A byte-derived destination means: read one byte from the stated source address, zero-extend it, then add 17 to obtain the destination address. The destination therefore lies between 17 and 272.

### Preconditions

Successful traversal requires executable indirect-call targets and accessible memory for every reached access. Word accesses may require alignment or support for unaligned transfers; no alignment validation is shown.

No explicit argument validation, bounds checking, error return, allocation protocol, or ownership transfer is established.

## Behavior

### Dependency notation

The opaque identifiers F-U01 through F-U25 below identify **ordered groups of unresolved call occurrences**, not verified shared implementations.

Within a group:

- Calls occur in the stated order.
- An argumentless call has no explicit arguments in the supplied representation; hidden machine-state inputs remain possible.
- A “four-call chain” means four distinct call occurrences, the first three having no explicitly used result and the fourth supplying the stated result.
- Discarding a result does not make a call side-effect-free.

Repeated addresses or algebraically equal values do not authorize removing calls, reads, or writes.

### 1. Initial dispatch and first byte update — F-U01

The first call receives the current first argument. Its target depends on the incoming control-address state. Four further calls follow without explicit arguments. The last supplies a 64-bit result.

Call the low word of that result the **primary seed**, and its high word the **primary companion**. A scaled primary value is formed as four times the primary seed.

Next, a byte is read from:

- the incoming address-state value multiplied by 4096, plus 1.

That byte selects a destination by adding 17. One byte is written there, using the fourth argument shifted right by 20 after clearing its top two bits, then truncated to eight bits.

Thus this write retains input bits 20 through 27. Bits 28 and 29 pass the intermediate mask but are discarded by byte truncation.

### 2. Companion forwarding and first clearing sequence — F-U02

A call receives the primary companion. An argumentless call produces a 32-bit token, which is passed to the next call. Another argumentless call produces a 64-bit pair; the following call receives its low and high words in that order. Two more argumentless calls follow, the second producing a new word.

Before that word is forwarded, the function reads a byte at:

- scaled primary value multiplied by \(2^{28}\), plus 1.

It writes zero to the byte-derived destination. It then calls the next dependency with four times the newly returned word.

The address multiplication retains only its machine-width result. Under 32-bit modular arithmetic, the source address depends on only the lowest two bits of the primary seed.

### 3. Primary-derived clearing and argument transformation — F-U03

A four-call chain produces the **secondary seed**. Its value multiplied by four becomes the scaled secondary value.

A separate primary base is formed as sixteen times the primary seed. A byte is read at this primary base plus 1, and zero is written to its byte-derived destination.

The next call receives the primary seed shifted left by 12. An argumentless call returns a token. The following call receives that token and the current third argument shifted left by 28.

Four argumentless calls follow; the last produces a word. The function rereads the byte at the primary base plus 1, clears its newly selected destination, and then calls a dependency with four times that word.

The second read is not interchangeable with the earlier read: intervening dependencies may change the source byte.

### 4. First paired word writes — F-U04

A four-call chain runs. Its final result is not explicitly consumed.

The **secondary payload** is the secondary seed shifted left by 14. It is written as a word to:

- secondary seed multiplied by \(2^{14}\), plus 20.

A call then receives the secondary seed shifted left by 16. An argumentless call follows. The next call receives:

1. the secondary payload shifted left by 28;
2. an otherwise unresolved additional result-register value.

Under 32-bit truncating arithmetic, the first argument is zero: the secondary payload already has at least 14 low zero bits.

A four-call chain produces a word used as an index. The same secondary payload is written to four times that index plus 20. The next call receives four times that index.

The two word writes remain separately observable even when their addresses coincide.

### 5. Primary byte update and another pair-forwarding sequence — F-U05

A four-call chain produces the **tertiary seed**.

The byte at the primary base plus 1 is reread. Its selected destination receives the secondary payload shifted right by 20 after clearing the payload’s top two bits, truncated to a byte.

A call receives the secondary seed shifted left by 26. An argumentless call produces a token, and the next call receives that token together with the tertiary seed shifted left by 30.

An argumentless call produces a 64-bit pair. The next call receives the pair’s low and high words. Two argumentless calls follow, the second returning a word.

The byte at the primary base plus 1 is read again and its selected destination is cleared. A call then receives four times the last returned word.

### 6. Tertiary-associated word writes — F-U06

A four-call chain produces a word. Four times that word becomes the **tertiary payload**.

The tertiary payload is written to:

- tertiary seed multiplied by \(2^{14}\), plus 20.

The next call receives four arguments, in order:

1. tertiary seed multiplied by 4096;
2. the same value;
3. tertiary seed shifted left by 14;
4. the tertiary payload.

A four-call chain produces another word. The tertiary payload is written to four times that word plus 20. The next call receives four times that word.

### 7. First self-derived payload pair — F-U07

A four-call chain produces the **first self seed**. Its fourfold value becomes the **first self payload**.

The payload is written to sixteen times its seed plus 20. A call receives the payload three times as three separate arguments.

A four-call chain produces another word. The same payload is written to four times that word plus 20. A call receives four times that word.

### 8. Second self-derived payload pair — F-U08

A four-call chain produces the **second self seed**. Its fourfold value becomes the **second self payload**.

That payload is written to sixteen times its seed plus 20. A call receives the payload three times.

The **late base** is the second self seed multiplied by \(2^{30}\). Under 32-bit modular arithmetic it can only be zero, \(2^{30}\), \(2^{31}\), or \(3 \times 2^{30}\).

A four-call chain produces the **retained index**. The second self payload is written to four times this index plus 20. The next call receives four times this index.

### 9. Repeated late-base clearing — F-U09 and F-U10

A four-call chain produces the **selector**.

The function performs two byte writes, in this order:

1. zero at late base plus 17;
2. zero at late base plus 1.

A call receives four times the selector. An argumentless call returns a token, which is passed to the next call.

A four-call chain then produces a word. The two zero writes at late base plus 17 and late base plus 1 are repeated in that order. A call receives four times the newly returned word.

The two clearing pairs are separated by calls and must not be collapsed.

### 10. Word-derived selector read and third-argument replacement — F-U11

A four-call chain produces the **final addressing seed**. Four times this seed becomes the **retained addressing value**.

The function writes the retained index shifted left by 31 to sixteen times the final addressing seed plus 20. Under 32-bit arithmetic, this stored word is either zero or the top-bit-only value, according to the retained index’s least significant bit.

It then reads a word at:

- 20 plus the selector’s bits 20 through 29, interpreted as an unsigned integer.

The read address is therefore between 20 and 1043. It is not necessarily aligned.

A call receives the retained addressing value. The next argumentless call supplies the replacement third argument.

Another call receives the previously read word shifted right by 22 and then left by 28. With 32-bit truncation, only original bits 22 through 25 affect this argument.

### 11. Explicit zero word and late-base byte update — F-U12

A four-call chain produces an index. A zero word is written to four times that index plus 20. The next call receives four times the index.

Another four-call chain produces the **late token**.

The byte at late base plus 1 is read. Its selected destination receives the low byte of four times the late token. This read occurs after intervening dependencies; the earlier zeroing of late base plus 1 does not establish its current value.

A call receives four times the late token. An argumentless call returns a token. The next call receives that token and the late token shifted left by 30.

### 12. Captured-byte destination and delayed word store — F-U13

A four-call chain produces a word.

The function reads and retains one byte from late base plus 1. At the retained byte plus 17, it writes the low byte obtained by extracting the late token’s bits starting at bit 20 after clearing its top two bits.

A call receives four times the word produced by the chain. Four argumentless calls follow.

Only after those calls does the function write a word to the **retained byte plus 20**, using four times the replacement third argument.

The source byte is not reread for this word write. Dependencies may change memory in the interim, but the captured byte still determines its destination.

### 13. Global sampling and tail pair — F-U14

The function reads the represented 32-bit global value, after the delayed word write and before the next call.

A call receives the retained addressing value. Four argumentless calls follow, the last producing a 64-bit pair. Four times its low word becomes the **tail scaled value**.

The function rereads the byte at late base plus 1. Its selected destination receives the global sample shifted right by 22 and truncated to eight bits. Consequently, global bits 22 through 29 determine the stored byte.

A call receives the pair’s high word. An argumentless call produces a token. The next call receives that token and the tail scaled value shifted left by 28.

### 14. Tail clearing and next-iteration address state — F-U15

A four-call chain produces a word.

The byte at late base plus 1 is read again, and its selected destination is cleared. A call receives four times the returned word.

Another four-call chain produces a word. A call receives four times that word twice.

The next iteration’s incoming address-state value is set to four times the tail scaled value.

### 15. Iteration closure — F-U16

A four-call chain produces a word. Four times that word becomes the **next payload**.

The next payload is written to:

- tail scaled value multiplied by 4096, plus 20.

The fourth argument is replaced with the next payload. A call receives the tail scaled value shifted left by 14.

A final four-call chain produces a word. Four times that word replaces the first argument. The next payload is written to this replacement first argument plus 20.

Control then repeats from the initial dispatch without a termination test. The control-address state used for the next first-call target is replaced by a fixed represented continuation value.

## Side effects

The function directly performs byte and word writes at computed addresses, one explicit global read, and the indirect calls described above.

Read/write order matters:

- Byte-derived destinations generally use fresh reads.
- One late byte is deliberately retained across calls for a subsequent word destination.
- The selector word is sampled before the calls that replace the third argument.
- The global value is sampled before the tail call chain, not immediately before its byte write.
- Repeated clearing writes remain distinct observable operations.

Memory regions may overlap. No disjointness is established between byte sources, byte destinations, word destinations, the global, or dependency-managed state.

There is no explicit allocation or release. This does not establish that dependencies allocate nothing, release nothing, preserve globals, or preserve ownership. No ownership guarantee can be derived from their unresolved calls.

No recovery, rollback, or cleanup path is shown. A fault or nonreturning dependency may stop progress after partial side effects.

## Dependencies

All dependencies are unresolved indirect calls.

The F-U identifiers organize call occurrences only. They do not assert that separate occurrences call different functions, or that similar-looking calls share an implementation.

Most targets are constructed from fixed represented continuation locations plus offsets. Several use different offsets. The first target additionally depends on incoming control state. Target validity, calling conventions, hidden inputs, exceptions, and memory effects are unknown.

Three dependency results are explicitly consumed as 64-bit low/high pairs: the initial primary pair, the early forwarded pair, and the later forwarded pair; the tail also consumes a 64-bit pair. Their application-level meanings are not established.

## Acceptance examples

These are conditional static acceptance examples, not executed tests.

1. **Initial extraction:** With a fourth argument of `0x0AB00000`, the first byte-derived destination receives `0xAB`, provided preceding dependencies return and the memory accesses succeed.

2. **Byte-derived addressing:** If a source-byte read returns `0xFE`, its corresponding byte destination is address 271. The evidence does not support treating `0xFE` as a pointer to some separately located object.

3. **Selector bounds:** A selector whose bits 20 through 29 are all zero selects word address 20. If those bits are all one, it selects word address 1043, regardless of selector bits outside that range.

4. **Truncated call argument:** Under 32-bit truncating arithmetic, shifting the secondary payload left by 28 supplies zero for every secondary seed. This does not make the associated call removable.

5. **Captured destination:** If the retained late byte is 7, the delayed word write targets address 27 even if an intervening dependency changes late base plus 1 to 9.

6. **Mutable reread:** If two reads of primary base plus 1 return different bytes, their clearing writes target different addresses. Reusing the first read would not satisfy this behavior.

7. **Continuation:** If every dependency returns and every access succeeds, the iteration-closing word write is followed by another initial-dispatch sequence, not a normal function return.

## Uncertainties

The following remain **documented_only**:

- Whether the supplied representation faithfully captures executable control flow.
- Exact architecture, byte order, address width, and unaligned-access behavior.
- Machine semantics for signed overflow and left shifts.
- Initial values and provenance of hidden machine state.
- The extra result-register argument’s producer and meaning.
- Whether indirect calls preserve any particular register, global, or memory region.
- Whether low-address accesses are valid, mapped hardware, or reconstruction artifacts.
- Global layout and overlap with other represented symbols.
- Exceptional termination, asynchronous interference, and dependency ownership effects.

No inferred application purpose is an acceptance requirement.

## Evidence

The established evidence is limited to the supplied static body: its represented argument widths, dependency-call order, explicit memory accesses, arithmetic expressions, global sampling, loop-carried assignments, and unconditional repetition.

Warnings accompanying that body indicate omitted unreachable material and overlapping global symbols. They limit confidence in completeness and layout; they do not establish additional behavior.

No implementation inspection, runtime tests, memory traces, dependency execution, or independent validation has been performed.
