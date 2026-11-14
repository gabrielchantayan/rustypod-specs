<!-- Trusted provenance: F-d4583cafbdc3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d4583cafbdc3

## Purpose

F-d4583cafbdc3 performs a long, ordered sequence of indirect calls, derived-address memory reads, 32-bit stores, and byte stores. Its visible effects include recording the second and third input words, storing values derived from indirect-call results, and repeatedly clearing selected locations.

No reliable application-level purpose is established. In particular, the evidence does not justify describing this function as an allocator, initializer, serializer, device driver, or memory-release routine.

The supplied representation ends at an explicit bad-instruction termination marker. A successful normal return is not established.

This contract describes the visible sequence conditionally: each later operation is reached only if every earlier indirect call returns and every earlier memory access completes. It does not repair questionable control flow or replace low numerical addresses with presumed object pointers.

## Interface

### Inputs

Three explicit arguments are visible:

| Input | Width | Established use |
|---|---:|---|
| First input word | 32 bits | No direct use appears in the supplied body. |
| Second input word | 32 bits | Passed to the first indirect call and subsequently written to a derived address. |
| Third input word | 32 bits, unsigned | Logically shifted right by three and stored unchanged at two later derived addresses. |

Additional implicit inputs affect execution:

- An incoming link-register value determines the first indirect-call target through addition of a fixed displacement.
- Indirect-call results supply numerical values used as addresses, indices, stored data, or later call arguments.
- Several byte and word reads supply further addresses or arguments.
- One later call uses an additional register result whose producing instruction and calling convention are not adequately established.

There is no explicit buffer pointer, buffer length, ownership token, callback table, or status-output argument.

### Output and termination

The recovered interface declares no return value. The visible sequence does not establish normal completion or a meaningful return-register value.

The final visible operation is abnormal termination associated with invalid instruction data. Whether this represents actual executable behavior, an incorrectly recovered boundary, or an unavailable continuation is **documented_only**.

### Width and layout conventions

The following notation describes values, not implementation identifiers:

- **Four-scaled** means multiplication by 4.
- **Sixteen-scaled** means multiplication by 16.
- **Reduced** means clearing the upper two bits of a 32-bit word, then logically shifting right by one.
- **High fragment** means clearing the upper two bits, shifting right by 20, then retaining only the low eight bits for a byte store.

Visible word accesses are 32 bits; visible byte accesses are eight bits. An eight-bit address-selector read is zero-extended before adding an offset. Consequently:

- A byte-selector destination at selector plus `0x11` lies between `0x11` and `0x110`.
- A word destination at selector plus `0x44` lies between `0x44` and `0x143`.

Those are numerical addresses in the supplied representation, not established offsets within an allocated object.

The apparent machine model uses 32-bit words and addresses. Unsigned calculations can be described by their low 32 bits. Several other expressions are represented using signed intermediate values, including left shifts and multiplication. Their exact overflow interpretation is not independently established. Examples below use values that avoid those ambiguities unless truncation is specifically under discussion.

Endianness is unspecified. A 32-bit store establishes a word value, not a particular sequence of four bytes.

## Behavior

### Execution and dependency notation

**F-U** is an opaque designation for the unresolved indirect-call dependency set. It does not assert that all calls reach one function. Individual target identities, target equivalences, and dependency contracts are unknown.

A **four-call acquisition sequence** means four consecutive indirect calls with no explicit arguments in the supplied representation; the fourth result is used by the surrounding operation. Earlier results are not directly consumed there. These calls remain observable and must not be removed merely because their explicit results are unused.

“No explicit arguments” does not establish that a target has no implicit register, global, or memory inputs.

### 1. Record the second input and derive an initial working word

1. Compute the unsigned third input shifted right by three. Retain this value for a later address calculation.
2. Invoke F-U using the link-register-derived target, passing the second input.
3. Perform a four-call acquisition sequence. At four times its final result plus `0x14`, store the second input as a 32-bit word.
4. Invoke F-U with four times that result.
5. Perform another four-call acquisition sequence. Form an **initial working word** equal to four times its final result. At sixteen times that result plus `0x14`, store the initial working word.
6. Invoke F-U with the initial working word, invoke another F-U call, and pass that latter call’s result to the following F-U call.
7. Perform another four-call acquisition sequence. At four times its final result plus `0x14`, store the reduced initial working word.
8. Invoke F-U with four times that result.

The second input’s store precedes all later word and byte stores. No validation of the second input or these derived destinations is visible.

### 2. Propagate another scaled result

1. Perform a four-call acquisition sequence. Retain four times its final result as an **intermediate scaled word**.
2. At sixteen times that result plus `0x14`, store the intermediate scaled word.
3. Read a 32-bit word at the intermediate scaled word plus `0x14`, logically shift the read value right by three, and pass it to F-U.
4. Perform another four-call acquisition sequence. At four times its final result plus `0x14`, store the retained intermediate scaled word.
5. Invoke F-U with four times that result.

The read in this phase is not necessarily from the preceding store: their address formulas differ.

### 3. Record the third input and establish the principal derived state

1. Perform a four-call acquisition sequence. Retain its final 32-bit result as the **principal result**.
2. Store the third input at the address formed by multiplying its previously retained right-shifted value by `0x80000`, then adding `0x24`.
3. Retain sixteen times the principal result as the **principal base**, and retain the reduced principal result as the **reduced principal value**.
4. Invoke F-U with the principal base and four times the principal result.
5. Make four further indirect calls with no explicit arguments.
6. Store the third input at principal base plus `0x24`.
7. Read a 32-bit word at principal base plus `0x74` and pass it to F-U.
8. Perform a four-call acquisition sequence. Retain its final result as the **shift-source result**.
9. Read one byte at principal base plus one. At that byte’s numerical value plus `0x11`, store the high fragment of the principal result.
10. Invoke F-U with four times the shift-source result.

The two third-input stores use independently calculated addresses. Their equality is not established.

### 4. Perform selector-based byte clears

1. Invoke F-U without explicit arguments, then pass its result to another F-U call.
2. Perform a four-call acquisition sequence.
3. Re-read the byte at principal base plus one. Store zero at the newly read selector plus `0x11`.
4. Invoke F-U with four times the acquisition result.
5. Make four more indirect calls without explicit arguments.
6. Store one zero byte at reduced principal value plus `0x0f`.
7. Retain reduced principal value multiplied by `0x400` as the **secondary base**.
8. Read one byte at secondary base plus one and store zero at that selector plus `0x11`.
9. Invoke F-U with four explicit arguments: three copies of the shift-source result shifted left by 23, followed by the shift-source result shifted left by 25.

The selector in step 3 is read again after intervening calls. It cannot be assumed to equal the selector used in the preceding phase.

### 5. Clear selected words and absolute location `0x24`

1. Invoke F-U without explicit arguments, then pass its result to another F-U call.
2. Perform a four-call acquisition sequence.
3. Read the byte at secondary base plus one and retain that selector.
4. Store zero at retained selector plus `0x11`.
5. Invoke F-U with four times the acquisition result.
6. Make four calls without explicit arguments.
7. Store a 32-bit zero at retained selector plus `0x44`.
8. Store a 32-bit zero at absolute address `0x24`.
9. Invoke F-U with the explicit arguments zero, `0x98`, secondary base, and `0x98`.

The selector retained in step 3 is reused in step 7 without another selector read. Intervening callees could modify the selector’s source byte, but that would not change the already retained numerical selector.

### 6. Continue word clearing using call results

1. Invoke F-U without explicit arguments; pass its result to another F-U call.
2. Invoke F-U without explicit arguments again; pass that result to another F-U call.
3. Perform a four-call acquisition sequence.
4. Store a 32-bit zero at four times its final result plus `0x44`, then store a 32-bit zero at absolute address `0x24`.
5. Invoke F-U with four times that result.
6. Perform a four-call acquisition sequence whose final result is represented as 64 bits.
7. Pass four times the low 32-bit half to F-U.
8. Store a 32-bit zero at twice the high 32-bit half plus `0x44`, then store a 32-bit zero at absolute address `0x24`.
9. Invoke F-U with zero, an additional register-derived value, and four times that register-derived value.
10. Invoke F-U without explicit arguments.
11. Invoke F-U with the low 22 bits of the reduced principal value shifted left by seven.
12. Perform another four-call acquisition sequence.
13. Store a 32-bit zero at four times its final result plus `0x44`, then store a 32-bit zero at absolute address `0x24`.
14. Invoke F-U with four times that result.

The 64-bit result is consumed as separate low and high halves. It must not be collapsed into a single 32-bit pointer.

### 7. Clear and rebuild additional derived words

1. Perform a four-call acquisition sequence, then pass four times its result to F-U.
2. Use the result of that additional call to address a 32-bit zero store at four times the result plus `0x14`.
3. Invoke F-U with three copies of that result and, as the fourth argument, the low 22 bits of the reduced principal value shifted left by seven.
4. Perform a four-call acquisition sequence.
5. Store a 32-bit zero at four times its result plus `0x14`; invoke F-U with four times that result.
6. Perform another four-call acquisition sequence.
7. Retain four times its result as a **later scaled word**. Store it at sixteen times the result plus `0x14`; invoke F-U with the later scaled word.
8. Perform another four-call acquisition sequence.
9. Store the retained later scaled word at four times its result plus `0x14`; invoke F-U with four times that result.
10. Perform another four-call acquisition sequence.
11. Store four times its result at sixteen times that result plus `0x14`.
12. Invoke F-U with four times that result and sixteen times that result.
13. Invoke F-U without explicit arguments.
14. Pass that call’s result and the low 22 bits of the reduced principal value shifted left by four to F-U.
15. Perform another four-call acquisition sequence.
16. Store the low 22 bits of the reduced principal value multiplied by `0x80` at four times its result plus `0x14`.
17. Invoke F-U with four times that result.

The seven-bit-shifted quantity is at most `0x1fffff80`. The four-bit-shifted quantity is at most `0x03fffff0`. Neither requires 32-bit overflow.

### 8. Perform final byte updates

1. Perform a four-call acquisition sequence. Retain its final result as the **first final result**.
2. Form the **final selector base** by multiplying the reduced principal value by `0x4000000`.
3. Read one byte at final selector base plus one. At the selector plus `0x11`, store the low byte of four times the first final result.
4. Invoke F-U with four times the first final result.
5. Invoke F-U without explicit arguments and pass its result to another F-U call.
6. Perform a four-call acquisition sequence.
7. Re-read the byte at final selector base plus one. Store the high fragment of the first final result at this selector plus `0x11`.
8. Invoke F-U with four times the acquisition result.
9. Perform another four-call acquisition sequence. Retain its final result as the **second final result**.
10. Re-read and retain the byte at final selector base plus one. Store the low byte of four times the second final result at retained selector plus `0x11`.
11. Invoke F-U with the retained selector, zero-extended.
12. Invoke F-U without explicit arguments and pass its result to another F-U call.
13. Perform another four-call acquisition sequence.
14. Read a byte at eight times the second final result plus one. Store the high fragment of the second final result at this selector plus `0x11`.
15. Invoke F-U with four times the last acquisition result.
16. Make four further indirect calls without explicit arguments.
17. Reach the invalid-instruction termination marker.

The final byte-store address depends on the second final result, not the final selector base. The last acquisition result instead supplies a subsequent call argument.

## Side effects

The established direct side effects are the ordered reads and writes above. There is no visible conditional branch that skips a store based on input validity, call status, or an error flag.

Important constraints:

- Stores may overlap each other or selector-read locations.
- Absolute address `0x24` receives four separate 32-bit zero stores. Intervening calls prevent treating them as one operation.
- Repeated selector reads must remain distinct because intervening callees may modify memory.
- A zero written earlier is not guaranteed to remain zero after subsequent indirect calls.
- There is no visible rollback, transactional boundary, synchronization, or error recovery.

No allocation, release, ownership transfer, reference-count change, or resource lifetime can be attributed to F-U without further evidence. The current function has no visible direct allocation or release operation.

## Dependencies

The only supplied authoritative function identifier is **F-d4583cafbdc3**. No authoritative identities or contracts are supplied for its indirect targets.

**F-U** therefore identifies unresolved dependencies only. Required but unknown properties include:

- Executable target resolution and calling conventions.
- Whether each invocation returns.
- Whether explicit arguments exhaust the target’s inputs.
- Result widths and signedness where recovery is ambiguous.
- Memory, global-state, register, allocation, and ownership effects.
- Whether the apparent 64-bit result is one logical value or a recovered register pair.

The first target additionally depends on incoming link-register state. Later targets use various fixed displacements from embedded address-like values. Their differing forms must not be normalized into one verified helper.

## Acceptance examples

These are conditional static acceptance cases, not executed tests.

1. **Initial input recording.** If the first acquisition result is 3, the second input must be stored as a 32-bit word at `0x20`, followed by a call whose explicit argument is 12. Earlier calls still occur.

2. **Third-input address calculation.** For third input 15, the retained right-shifted value is 1, making the first third-input destination `0x80024`. For third input 7, that destination is `0x24`. The stored word is the original input, not its shifted value.

3. **Reduced-word calculation.** An initial working word of `0x40000004` produces a reduced stored value of 2. Shifting without first clearing the upper two bits would not satisfy the visible behavior.

4. **Independent selector observations.** If the principal-base selector is 5 before the high-fragment store and becomes 9 before the later clear, the writes target `0x16` and `0x1a`, respectively. Reusing the first selector would be incorrect.

5. **Byte truncation.** A first final result of 65 produces byte value 4 for its four-scaled byte store. The associated call receives 260, not 4, assuming ordinary 32-bit arithmetic.

6. **Split-width result.** If the represented 64-bit result has low half 3 and high half 5, the next call receives 12, and the subsequent zero-word destination is `0x4e`. The following absolute `0x24` clear remains a separate write.

7. **Interrupted execution.** If an indirect call does not return, no later direct access in this sequence is established to occur. There is no visible fallback path.

## Uncertainties

The following are **documented_only**, not guessed:

- Application purpose and object-layout interpretation.
- Actual instruction-set semantics and complete calling convention.
- Correctness and completeness of the recovered control flow.
- Executability of indirect targets.
- Validity, alignment, and permissions of derived addresses.
- Overflow behavior of signed-looking intermediate expressions.
- Meaning and origin of the additional register-derived argument.
- Callee preservation of memory, globals, registers, or resources.
- Whether low addresses denote ordinary memory, mapped hardware, or recovery artifacts.
- Whether the invalid-instruction marker corresponds to actual runtime termination.

A usable execution environment must make every reached target and memory access valid according to its machine rules. The function visibly performs no checks establishing those preconditions.

## Evidence

**Established static evidence:** the supplied representation contains three 32-bit explicit inputs; the shifts, masks, scaling factors, offsets, access widths, call ordering, and direct stores described above; a split 64-bit call result; repeated absolute-address clears; and an invalid-instruction endpoint. It also carries warnings about unreliable instruction recovery and overlapping global symbols.

**Inferred intent:** repeated zeroing and value propagation may participate in state setup or reset. This is not strong enough to define an application-level contract.

**Unknown:** dependency identities, runtime validity, ownership semantics, and normal completion.

No runtime tests have been performed. The acceptance examples are arithmetic and ordering consequences of the supplied static representation, not observations of an executing implementation.
