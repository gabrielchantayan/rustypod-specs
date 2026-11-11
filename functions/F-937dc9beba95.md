<!-- Trusted provenance: F-937dc9beba95; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-937dc9beba95

## Purpose

F-937dc9beba95 interleaves indirect calls with computed-address memory stores, then enters an unconditional loop containing further calls and byte-clearing operations. Its application-level purpose is not established.

The represented operation performs three four-byte stores, several byte stores, and repeated one-byte selector reads. A selector byte determines a numerical destination address by adding 17 to its unsigned value. Two different expressions supply selector-read addresses; **those addresses may coincide**.

This contract describes the supplied static representation, subject to the arithmetic qualifications and reconstruction limitations below. It does not establish that the representation accurately recovers the executable’s function boundary, calling convention, or complete control flow.

## Interface

### Explicit inputs

The displayed interface contains four four-byte arguments:

| Input | Represented use |
|---|---|
| First word | No explicit use in the body. |
| Second word | No explicit use in the body. |
| Third word | Treated as unsigned, logically shifted right by seven, and supplied to an indirect call. |
| Fourth word | Treated as unsigned, logically shifted right by seven, used to form a byte-store address, and supplied to an indirect call. |

The third and fourth words have represented widths of 32 bits. Their right shifts insert zeros. Each shifted value is between zero and 33,554,431.

The absence of explicit uses of the first two words does not prove that actual execution ignores them. The indirect-call interfaces are incompletely recovered and may depend on incoming machine state not expressed as arguments.

### Ambient inputs

Three additional incoming quantities are referenced without initialization in the represented body:

- An unsigned address word whose value, shifted right by eight, supplies the primary selector-read base.
- A four-byte stored word used as the payload of the first direct store.
- A control-address value used to derive the first indirect-call destination.

Their origins and calling-convention roles are unknown. They are effective inputs to the represented execution, not locally generated values.

### Result and termination

No explicit result or normal return appears. The final loop has no represented local exit.

Reaching that loop does not establish uninterrupted execution forever. An indirect callee may not return, may transfer control elsewhere, or may terminate execution. Calls and memory accesses may fault. No local error status, exception handler, retry, or recovery action is shown.

### Preconditions

To continue through a particular portion of the represented path:

- Every reached indirect target must be callable under the actual machine convention.
- Every selector-read address must permit a one-byte read.
- Every selected destination and fourth-input-derived destination must permit a one-byte write.
- Each four-byte store must satisfy the platform’s access and alignment requirements.
- Intervening calls must return to the represented continuation.

No corresponding validation is visible. Ownership, allocation extent, endianness, alignment rules, and fault behavior are not established.

## Behavior

### Widths, arithmetic, and selector interpretation

The word calculations below use a conditional 32-bit machine interpretation: products and left shifts retain their low 32 bits, and address additions are interpreted at that width. Some displayed intermediates are signed; wrapping behavior is therefore an interpretation of the represented machine operations, not an assertion of defined signed-overflow semantics.

A selector read obtains one unsigned byte, ranging from zero through 255. The selected write destination is that value plus 17, so it lies between numerical addresses 17 and 272 inclusive.

This is not a multi-byte pointer load. A selector of zero selects address 17 and does not suppress the write.

Every specified selector read is fresh. Its value cannot be substituted with an earlier read merely because the address expression is unchanged.

For clarity:

- **Primary read address** means the ambient address word logically shifted right by eight, plus one.
- **First packed result** means the retained call result used for the first two nonzero-capable selected-byte stores.
- **Second packed result** means the retained call result used for the fourth-input-derived store and the later selector-address calculation.
- **Secondary read address** means the low 32 bits of 4,096 times the second packed result, plus one.

The primary address ranges from 1 through 16,777,216. Under the stated modular interpretation, the secondary address has its low twelve bits equal to 1.

These are different derivations, not guaranteed different addresses. Both equal 1 when the ambient address word and second packed result are zero.

### Initial calls and four-byte stores

Five indirect calls occur before the first direct store. No explicit arguments are displayed for these calls. The first target depends on the incoming control-address value.

The fifth result is scaled by four. A four-byte store writes the ambient stored word at that scaled value plus 20. A subsequent indirect call receives the scaled value as its sole displayed argument.

Four more indirect calls follow. The last result supplies two related values:

- A base equal to sixteen times that result.
- A payload equal to four times that result.

The payload is stored as four bytes at the base plus 20.

The primary selector-read base is then established from the ambient address word. An indirect call receives three displayed arguments, in order:

1. The base.
2. The same base again.
3. The third explicit input logically shifted right by seven.

Four further indirect calls execute. Before the next call, the operation overwrites the preceding four-byte store location with the base itself. That next call receives four times the most recent result, rather than four times the result originally used to establish the base.

The initial payload at the second store location is therefore exposed to the intervening calls before the overwrite. No non-aliasing or callee-memory-preservation condition is established.

### First packed result and selected-byte stores

Four further indirect calls execute. The last produces the first packed result.

A fresh byte is read at the primary read address. At the destination selected by that byte, the operation writes the low eight bits of four times the first packed result. The next indirect call receives three equal arguments, each containing the 32-bit value of four times that result.

One subsequent indirect call produces a four-byte result that is immediately supplied to the following indirect call as its displayed argument.

Four additional indirect calls follow that forwarding call. Their final result is retained for a later call argument.

The primary selector is then read again. The operation writes a byte selected from the earlier first packed result:

- The high two bits are cleared.
- The remaining value is logically shifted right by 20.
- Only the low eight bits are stored.

The stored byte therefore contains original bits 20 through 27. Original bits 28 and 29 remain in the shifted intermediate but do not reach memory.

The next indirect call receives four times the latest call result. It does not receive the corresponding scaling of the first packed result.

### Fourth-input-derived store and primary clears

Four further indirect calls execute. Their final result becomes the second packed result.

The operation writes the low byte of four times that result at the fourth explicit input logically shifted right by seven, plus 28. This address lies between 28 and 33,554,459.

Immediately afterward, the primary selector is read afresh and zero is written to its selected destination. These two stores are ordered. If their destinations coincide, the clear follows the fourth-input-derived store and replaces its byte before the next call, assuming ordinary memory semantics and no external interference.

An indirect call receives the fourth explicit input logically shifted right by seven.

One further indirect call executes. Another call then receives the second packed result with its high two bits cleared and the remainder logically shifted right by six. This argument ranges from zero through 16,777,215.

Four additional indirect calls execute. After the last returns, the operation rereads the primary selector, clears the newly selected destination, and calls another indirect target with four times the latest result.

### Transition to the loop

Four further indirect calls execute. From the last result, the operation retains original bits 25 through 29 as a five-bit value between zero and 31. This value initially serves as the loop-control word.

The secondary read address is formed from the second packed result. The represented calculation first scales that result by four, then scales the retained value by 1,024 and adds one. Under the conditional 32-bit interpretation, this is the low 32 bits of 4,096 times the second packed result, plus one.

A byte is read at that address, and zero is written to the selected destination.

The primary and secondary read addresses may coincide. When they do, reads through either expression access the same selector byte. Intervening calls, external changes, or overlapping direct stores may therefore affect the selector later observed through either expression. Even when the read addresses differ, the bytes read may be equal and select the same destination.

An indirect call receives twice the five-bit value, an even number between zero and 62. One subsequent indirect call produces the initial first argument for the loop.

### Unconditional loop

Each iteration has the following observable phases:

1. An indirect call receives the retained first argument and the loop-control word logically shifted right by eight.
2. Four further indirect calls execute; the final result is retained.
3. The selector at the secondary read address is reread, and zero is stored at the selected destination.
4. Another indirect call receives four times the retained result.
5. Four further indirect calls execute; the last result establishes the next iteration’s state.

The next first argument is four times that final result. The next loop-control word is that result shifted left by 26, retaining only the low 32 bits.

The secondary read address remains based on the pre-loop second packed result. Later loop results do not replace its derivation. Its contents are not fixed: each iteration performs a fresh read.

For the first iteration, the second argument is zero because the initial loop-control word is at most 31.

For subsequent iterations, the second argument consists of the preceding final result’s low six bits moved into bit positions 18 through 23. Equivalently, it is those six bits multiplied by 262,144.

No local completion test, counter-based exit, or error branch appears.

## Side effects

### Direct memory effects

Before the loop, the represented body performs:

- Three four-byte stores, including an overwrite of the second store’s location.
- Two selected-byte stores derived from the first packed result.
- One byte store derived from the second packed result and fourth explicit input.
- Two selected-byte clears using the primary read address.
- One selected-byte clear using the secondary read address.

Each completed loop iteration performs one additional selected-byte clear using the secondary read address.

A “nonzero-capable” store may still write zero; its value depends on the relevant result bits.

### Ordering and aliasing

Calls and stores occur in the represented order. Calls may observe prior stores or alter memory before later reads and writes. No assumption is made that unknown callees preserve globals or leave selectors unchanged.

The contract imposes no disjointness requirement among:

- The primary and secondary read addresses.
- The destinations selected by any two reads.
- Selector-read addresses and write destinations.
- Byte-store destinations and the four-byte store regions.

Consequently, a direct write may change a later selector when the locations overlap. A call modifying a shared selector affects subsequent reads through either address expression. Separate selector addresses may also select the same destination without themselves aliasing.

No atomic transaction, rollback, synchronization primitive, memory barrier, or concurrency model is established. The stated ordering is that of the supplied representation, not a guarantee about physical bus transactions.

### Ownership and resources

No direct allocation, release, or ownership transfer is identified.

An argument’s numerical use does not establish whether it denotes a pointer, handle, size, or plain integer. Unknown callees may have resource effects, but this evidence does not characterize them or authorize assumptions about retention or release.

## Dependencies

The supplied identity map establishes only F-937dc9beba95. It does not provide opaque identities for the indirect callees. Their identities remain **documented_only**; no additional function IDs are assigned.

The represented dependency interactions include:

- Calls with no recovered explicit arguments.
- Calls with one, two, or three displayed arguments.
- Results used for address calculations, store payloads, later arguments, and loop state.
- One immediate forwarding of a four-byte result to the next call.

Distinct call occurrences are not necessarily distinct functions. Conversely, repeated call patterns do not establish a shared target.

A call with no displayed arguments is not proven to have a zero-argument machine interface. A result not consumed by the represented body does not prove a void return type.

Every effect described after a call is conditional on that call returning to the expected continuation. Callee effects, actual signatures, and preservation obligations are unknown.

## Acceptance examples

These are conditional static consequences, not executed tests. They assume returning callees, valid memory accesses, ordinary memory semantics, and the stated 32-bit arithmetic interpretation.

1. **Low-byte truncation.**  
   If the first packed result is 65, scaling by four produces 260. The first selected-byte store writes 4. The following three-argument call receives 260 in all three positions.

2. **Later packed-field store.**  
   If the first packed result is hexadecimal 11300000, or decimal 288,358,400, the later shifted intermediate is 275. Its stored low byte is 19.

3. **Fresh primary-selector reads.**  
   A selector value of 3 selects address 20. If an intervening callee changes that selector to 9, the next read selects address 26. Reusing the earlier selector would not match the represented behavior.

4. **Fourth-input-derived address.**  
   A fourth input of 1,024 becomes 8 after the right shift, giving destination 36. A second packed result of 63 causes this store to write 252.

5. **Ordered overlapping stores.**  
   In the preceding example, a following primary-selector value of 19 also selects address 36. The subsequent clear leaves zero there before the next call, absent other interference.

6. **Coincident selector-read addresses.**  
   With an ambient address word of zero and a second packed result of zero, both selector-read addresses are 1. A callee changing the byte at address 1 from 3 to 9 changes a subsequent secondary-selected destination from 20 to 26, just as it would for a subsequent primary read.

7. **Different reads, equal destinations.**  
   If the two read addresses differ but both contain 7, both select destination 24. Distinct read addresses do not imply distinct write destinations.

8. **First loop iteration.**  
   Every possible initial five-bit loop-control value produces a second argument of zero after shifting right by eight.

9. **Later loop state.**  
   If an iteration’s final call returns 63, the next first argument is 252. The next second argument is 16,515,072, hexadecimal 00FC0000.

10. **Zero selector and interrupted execution.**  
    A zero selector still causes a store at address 17. If the preceding callee never returns, however, that later read and store are not required to occur; no recovery behavior is established.

## Uncertainties

### Established static evidence

The supplied representation shows the call ordering, explicit argument expressions, direct stores, selector reads, retained-value calculations, and unconditional final loop described above.

It does not establish unequal selector-read addresses. Their possible equality follows from their represented arithmetic, including the zero-input example.

### Inferred intent

The repeated calls and extracted bit fields suggest participation in a larger stateful operation. No particular application purpose—such as decoding, serialization, device control, or memory management—is established.

### Documented-only unknowns

The following remain **documented_only**:

- Accuracy of the function boundary and displayed four-argument interface.
- Actual calling convention and origins of ambient inputs.
- Indirect-callee identities, signatures, side effects, and termination behavior.
- Whether represented retained values correspond faithfully to preserved executable state.
- Executable treatment of signed multiplication and left-shift edge cases.
- Address-space interpretation, endianness, alignment, and access-fault behavior.
- Whether low numerical addresses represent ordinary memory, device mappings, or invalid locations.
- Concurrency and external interference.
- Whether omitted regions or invalid instruction decoding conceal additional paths or exits.
- Whether the apparent nontermination is intentional or a reconstruction artifact.

No conventional pointer, ownership, error, or global-preservation semantics are substituted for these unknowns.

## Evidence

This contract is based solely on the supplied static representation and its accompanying warnings. Those warnings report invalid instruction data and removal of several unreachable regions, limiting confidence in recovered control flow and interfaces.

The address-aliasing conclusion follows directly from the two represented address expressions: setting both relevant source values to zero yields address 1 for both. No inequality precondition is present.

No implementation access, executable inspection, runtime tests, smoke runs, or independent runtime verification have been performed. Acceptance examples express conditional arithmetic and ordering consequences, not observed execution.
