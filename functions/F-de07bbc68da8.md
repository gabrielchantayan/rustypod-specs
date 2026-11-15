<!-- Trusted provenance: F-de07bbc68da8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-de07bbc68da8

## Purpose

F-de07bbc68da8 represents an unconditional loop of indirect dependency calls, byte accesses, and 32-bit word accesses. Each completed iteration establishes argument-like and address-base state for the next iteration.

The available static evidence establishes an operation sequence, not a domain purpose. It does not establish that the function initializes an object, serializes data, allocates storage, or configures hardware. The terms “index,” “scale,” and “replicated word” below describe arithmetic roles only.

This contract distinguishes:

- **Established static evidence:** represented calls, access widths, data dependencies, and ordering.
- **Conditional machine interpretation:** arithmetic consequences when 32-bit operations wrap modulo \(2^{32}\).
- **documented_only:** unresolved executable behavior, dependency contracts, memory validity, and calling-convention details.

There is no represented normal return or loop termination condition. Whether this representation accurately captures reachable executable control flow remains **documented_only**.

## Interface

### Explicit arguments and result

The represented interface has four 32-bit argument slots and no return value.

| Argument | Represented use |
|---|---|
| First | No explicit use. |
| Second | Passed to the first dependency call on entry; replaced at each completed back edge. |
| Third | No explicit use. |
| Fourth | Its low eight bits are captured at iteration entry for two subsequent byte writes. Its carried value is set to zero later in the iteration. |

“No explicit use” does not establish irrelevance to the actual executable ABI. Hidden machine inputs and incomplete call information are present.

No success code, failure code, returned pointer, or normal completion is represented. A dependency that does not return prevents subsequent operations from being reached. Invalid calls or accesses have no represented recovery path.

### Additional required state

The body also consumes:

- An incoming control-address value selecting the first indirect target.
- An incoming 32-bit auxiliary value used in an early left shift.
- An incoming unsigned 32-bit auxiliary value used in a logical right shift.
- An incoming base used for initial byte writes and later selector reads.
- A global 32-bit storage location.
- An additional register-carried call operand without an explicit defining assignment.

The initial values and ABI meanings of these inputs are **documented_only**. They are not established defaults.

### Widths and notation

Bytes are eight bits. Word accesses and ordinary arithmetic operands are represented as 32 bits.

There are **four explicitly represented 64-bit call results**: F-X07.5, F-X12.5, F-X13.4, and F-X19.4. Their low and high halves are each 32 bits.

For the descriptions below:

- Scaling by four means a 32-bit left shift by two.
- A **word destination for an index** is address 20 plus four times that index.
- A **selected byte destination** is address 17 plus an unsigned selector byte. Its range is therefore 17 through 272, inclusive.
- A masked extraction means masking a word to its low 30 bits, logically shifting right by 20, and truncating to eight bits for a byte write.

A selected byte destination is absolute; it is not relative to the base used to read its selector.

Arithmetic consequences use 32-bit wraparound. This is a conditional machine interpretation, not a claim of portable signed arithmetic. Actual address width, alignment requirements, byte order, and memory mapping are **documented_only**.

## Behavior

### Dependency notation and ordering

Opaque labels **F-X01 through F-X26** identify call groups in execution order. A suffix identifies a particular call occurrence, such as F-X03.7. These labels do not establish distinct target identities or imply shared implementations.

Unless otherwise specified, a group contains five calls:

1. Its first call receives the stated explicit arguments.
2. Four further calls have no explicit arguments.
3. The fifth call provides any result subsequently described.

“No explicit arguments” does not establish an argument-free executable ABI. Intermediate results are unused unless stated otherwise.

Every call in a group precedes the following direct memory event. Calls and repeated writes must not be omitted or coalesced on the assumption that they have no side effects.

### Entry and initial word operations

At iteration entry, the low byte of the current fourth argument is captured.

**F-X01 contains seven calls.** Its first receives the current second argument. Its second produces a word passed to its third. Four calls without explicit arguments follow; the seventh produces the initial scale value.

After F-X01, the captured byte is written first at current base plus 17, then at current base plus 1. The value is not recaptured between writes.

**F-X02** receives four times the initial scale value. Its final result is the **first index**.

The following direct accesses occur in order:

1. A word write at address 20 plus 16 times the first index, storing the incoming auxiliary value shifted left by 28.
2. A word read at address 20 plus the incoming unsigned auxiliary value logically shifted right by 22.

These addresses are not relative to the current base.

**F-X03 contains seven calls:**

- F-X03.1 receives four times the first index.
- F-X03.2 produces the **retained word**.
- F-X03.3 receives the just-read word logically shifted right by 22 and then shifted left by 28.
- F-X03.4, F-X03.5, F-X03.6, and F-X03.7 have no explicit arguments.
- F-X03.7 produces the **clear index**.

All four calls after F-X03.3 intervene before the next write; none is assumed effect-free. A zero word is then written to the word destination for the clear index.

### Initial selector operations

**F-X04** receives four times the clear index. Its final result is the **first packed value**.

The byte at current base plus 1 is read. The low byte of four times the first packed value is written to the selected byte destination.

**F-X05 contains seven calls.** Its first receives four times the first packed value. Its second returns a word passed as the first argument of its third; the third’s second argument is the first packed value shifted left by 30. Four calls without explicit arguments follow, with the seventh producing the **next index**.

The byte at current base plus 1 is read again and saved. Its selected destination receives the masked extraction of the first packed value.

**F-X06** receives four times the next index. Its final result has no explicit use.

A word equal to four times the retained word is written at address 20 plus the saved selector. This uses the saved byte, not a new selector read. The address need not be word-aligned.

The global word is then read and saved. This read follows the word write and precedes F-X07.

### First wide result and first replicated word

**F-X07** receives four times the first index. F-X07.5 returns a 64-bit value. Four times its low half becomes the **first wide-derived scale**.

The byte at current base plus 1 is reread. Its selected destination receives the low byte of the saved global word logically shifted right by 22. The global word is not reread after F-X07.

**F-X08 contains seven calls.** Its first receives the high half of F-X07.5. Its second returns a word passed to its third, alongside the first wide-derived scale shifted left by 28. Four calls without explicit arguments follow; the seventh returns an index.

The byte at current base plus 1 is reread, and its selected destination is cleared.

**F-X09** receives four times that index. Four times its final result is supplied twice to the first call of **F-X10**. F-X10 otherwise follows the five-call pattern.

Four times F-X10’s final result becomes the **first replicated word**. It is written at address 20 plus 4096 times the first wide-derived scale.

**F-X11** receives the first wide-derived scale shifted left by 14. Its final result selects a word destination, which receives the first replicated word.

**F-X12** receives four times that last index. F-X12.5 returns a 64-bit value. Four times its low half becomes the **second wide-derived scale**.

A selector byte is read at address 1 plus 16384 times the first wide-derived scale. Its selected destination receives the masked extraction of the first replicated word.

### Second wide result and second replicated word

**F-X13 contains seven calls:**

- Its first receives the high half of F-X12.5.
- Its second returns a word passed to its third as its sole explicit argument.
- F-X13.4 has no explicit arguments and returns a 64-bit value.
- F-X13.5 receives that value’s low and high halves, in that order.
- Its sixth and seventh calls have no explicit arguments; the seventh returns an index.

A selector byte is read at address 1 plus 268435456 times the second wide-derived scale. Its selected destination is cleared.

**F-X14** receives four times the new index. Its final result is the **replication index**.

A selector byte is read at address 1 plus four times the second wide-derived scale. Its selected destination is cleared.

**F-X15 contains seven calls.** Its first receives the second wide-derived scale shifted left by 10. Its second returns a word passed to its third, alongside the retained word shifted left by 28. Four calls without explicit arguments follow; the seventh returns an index.

The selector at address 1 plus four times the second wide-derived scale is reread, and its selected destination is cleared again. This is a separate read and write, not a redundant operation that may be removed.

**F-X16** receives four times the last index. Its final result has no explicit use.

The **second replicated word** is the replication index shifted left by 14. It is written at address 20 plus 4096 times four times the replication index.

**F-X17 contains seven calls.** Its first receives the value obtained by scaling the replication index by four and then shifting left by 14. Its second has no explicit arguments. Its third receives:

1. The second replicated word shifted left by 28.
2. An additional register-carried operand without an explicit defining assignment.

Four calls without explicit arguments follow; the seventh returns an index. The second replicated word is written to that index’s word destination.

Under 32-bit wraparound, F-X17.3’s first argument is zero: the replication index has effectively been shifted left by 42. This does not resolve its second argument or identify which dependency produced that register state.

### Final selector operations and third replicated word

**F-X18** receives four times the preceding index. Four times its final result becomes the **third scale**.

The selector at address 1 plus four times the second wide-derived scale is reread. Its selected destination receives the masked extraction of the second replicated word.

**F-X19 contains seven calls.** Its first receives the replication index shifted left by 26. Its second returns a word passed to its third, alongside the third scale shifted left by 28. F-X19.4 returns a 64-bit value, and F-X19.5 receives its low and high halves in that order. Its sixth and seventh calls have no explicit arguments; the seventh returns an index.

The same selector address is reread, and its selected destination is cleared.

**F-X20** receives four times that index. Four times its final result becomes the **third replicated word**. That word is written at address 20 plus 4096 times the third scale.

**F-X21** receives four explicit arguments on its first call:

1. The third scale multiplied by 1024.
2. The same value again.
3. The third scale shifted left by 12.
4. The third replicated word.

The third argument is equivalent, under the stated width interpretation, to shifting the unscaled F-X18.5 result left by 14. F-X21’s final result selects a word destination, which receives the third replicated word.

### Terminal operations and carried state

**F-X22** receives four times the preceding index. Its final result is the **first terminal index**. Four times that index is written at address 20 plus 16 times the same index.

The carried fourth-argument value is then set to zero. No immediate byte write accompanies this assignment.

**F-X23** receives four times the first terminal index three times. Its final result selects a word destination, which receives that same scaled value.

**F-X24** receives four times the last index. Its final result is the **second terminal index**. Four times that index is written at address 20 plus 16 times the same index.

The next base is established as the second terminal index multiplied by 1073741824.

**F-X25** receives four times the second terminal index three times. Its final result selects a word destination, which receives that same scaled value.

**F-X26** receives four times the last index returned by F-X25. Its final result, scaled by four, becomes the next second argument.

Before the unconditional back edge:

- The next auxiliary value for the early shifted word write becomes eight times the final index returned by F-X25.
- Zero is written at next base plus 17.
- Zero is then written at next base plus 1.
- The next unsigned auxiliary value becomes the next second argument.

The next iteration captures zero from the carried fourth argument. Its initial indirect target uses control state established near the loop tail, rather than necessarily repeating the original entry target.

## Side effects

Each completed represented iteration directly performs:

| Access | Count |
|---|---:|
| 32-bit word writes | **13** |
| Byte writes | **14** |
| Computed 32-bit word reads | 1 |
| Global 32-bit word reads | 1 |
| Selector-byte reads | 10 |

These totals exclude all dependency accesses.

The 13 word writes follow F-X02, F-X03, F-X06, F-X10, F-X11, F-X16, F-X17, F-X20, F-X21, F-X22, F-X23, F-X24, and F-X25. The 14 byte writes comprise two initial writes, ten selector-directed writes, and two loop-tail writes.

All direct accesses retain the described order. A selector read precedes its selected write. Calls may change selectors or other memory between direct accesses; no preservation is assumed.

Aliasing is consequential. An earlier direct write can change a later selector or word read. Repeated reads can therefore produce different destinations. The selector saved before F-X06 is an exception: its saved value determines the subsequent word-write address despite intervening calls.

There is no explicit allocation, release, ownership transfer, synchronization, bounds validation, rollback, or error handling. Dependencies may nevertheless allocate, release, retain, or mutate memory.

Continued execution requires each reached target and access to be valid and each dependency to return. No failure response is established when these conditions fail.

## Dependencies

F-X01 through F-X26 identify all call groups used by this contract. Their suffixed occurrences distinguish sequencing and result use, not implementation identity.

No dependency contract is available. In particular:

- Discarded results do not establish success.
- Calls without explicit arguments may consume hidden machine state.
- Globals, selectors, and externally visible state need not be preserved.
- Ownership, retention, consumption, and release behavior are unknown.
- Repeated arithmetic patterns do not prove repeated target identity.
- Four represented results are 64-bit; this does not establish a complete executable return convention.

Targets are derived from control locations and fixed displacements in the representation. A conventional user-provided callback table is not established. Target validity and actual calling conventions remain **documented_only**.

## Acceptance examples

These are conditional static consequences, not executed tests. Each assumes the relevant dependencies return, reached accesses are valid, and arithmetic follows the stated 32-bit interpretation.

1. **Fourth-argument truncation.** With fourth argument 0x123456AB, the initial writes store 0xAB at base plus 17 and then base plus 1. After a completed back edge, subsequent iterations capture zero.

2. **Absolute selection.** A selector value of 0xFE selects byte address 271, regardless of the base from which the selector was read.

3. **Saved selector.** If the selector saved before F-X06 is 7, the following word write targets address 27. A dependency changing the selector storage to 9 does not change this saved address; later explicit reads can observe 9.

4. **Extraction and truncation.** For first packed value 0xFFFFFFFF, its scaled byte write stores 0xFC. Its masked extraction stores 0xFF: the shifted intermediate is 0x3FF, then truncated to eight bits.

5. **Unsigned read address.** Incoming unsigned auxiliary value 0xFFFFFFFF produces an early word-read address of 1043. Whether that unaligned address is permitted is unresolved.

6. **F-X03 ordering.** After F-X03.3 returns, F-X03.4 through F-X03.7 must all complete before the zero-word write. Effects from any of those calls can affect the state observed when the write occurs.

7. **Collapsed shift.** F-X17.3 receives zero as its first explicit argument for every replication index under 32-bit wraparound. Its second argument remains unresolved.

8. **Next-base range.** The second terminal index’s low two bits determine the next base: 0, 0x40000000, 0x80000000, or 0xC0000000.

9. **Non-returning dependency.** If F-X01.1 does not return, neither initial byte write is reached. The represented body produces no success or error result.

## Uncertainties

The following remain **documented_only**:

- The executable ABI, hidden inputs, register preservation, and additional result registers.
- Dependency identities, behavior, and ownership contracts.
- Actual pointer width, alignment enforcement, endianness, permissions, and concurrency semantics.
- The physical meaning and aliasing of the global storage.
- Whether low-address writes and unusual target calculations represent intended reachable execution.
- Whether the represented unconditional loop matches executable control flow.
- Fault, exception, interruption, cancellation, and external termination behavior.

The supplied warning about overlapping global symbols prevents treating the global word as an isolated, exclusively owned object. The warning about removed unreachable material supplies no behavioral contract for omitted material.

No domain-level intent is established. The arithmetic labels in this document are descriptive conveniences, not inferred object layouts or ownership models.

## Evidence

This contract derives from the supplied static representation for F-de07bbc68da8, independently of the prior draft’s claims.

Established evidence includes four explicit argument slots, the unconditional back edge, seven calls in F-X03, call groups F-X01 through F-X26, four explicitly represented 64-bit call results, direct access widths and ordering, saved versus reread selectors, and carried iteration state. Enumerating direct stores yields 13 word writes and 14 byte writes per completed represented iteration.

Wraparound simplifications and acceptance examples are conditional deductions from represented widths. They do not establish executable validity or dependency behavior.

No implementation access, tool inspection, or runtime tests have been performed.
