<!-- Trusted provenance: F-3cebc320ed8a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3cebc320ed8a

## Purpose

F-3cebc320ed8a describes a visible execution prefix that sequences indirect calls, stores calculated 32-bit words, and performs two byte stores through destinations selected by byte reads. The supplied representation ends with an invalid-instruction termination marker rather than an ordinary return.

No application-level purpose is established. The evidence does not identify an allocator, serializer, device interface, or object initializer.

This contract separates:

- **Established static evidence:** calls, accesses, calculations, and sequencing explicitly present in the supplied representation.
- **Conditional arithmetic interpretation:** 32-bit bit-vector behavior used to describe calculations and acceptance examples, without asserting portable signed-language semantics.
- **Inferred intent:** possible explanations that impose no requirements.
- **Unknowns / documented_only:** behavior requiring executable, dependency, ABI, or memory-map evidence that was not supplied.

Warnings about invalid instruction data and truncated control flow limit the contract to the represented prefix. They do not establish that the representation accurately describes executable instructions.

## Interface

### Explicit inputs

| Position | Represented width and type | Visible use |
|---|---|---|
| First | 32-bit uninterpreted value | Forwarded to the first indirect invocation. |
| Second | 32-bit uninterpreted value | No direct reference in the visible body. |
| Third | 32-bit uninterpreted value | No direct reference in the visible body. |
| Fourth | Signed 32-bit integer | Shifted left by 26 positions for a later invocation. |

The absence of direct references to the second and third inputs does not establish that dependencies cannot observe incoming machine state containing them.

### Implicit inputs and preconditions

The prefix also consumes:

- An incoming link-register value used to calculate the first call target.
- An incoming 32-bit register value used by the first word store.
- Results returned by indirect dependencies.
- Memory at calculated word and byte addresses.
- Any additional machine state observed by dependencies.

No initialization or validation of these implicit inputs is visible.

For the entire described prefix to occur, each invoked dependency must return, each target must be executable under the actual calling convention, and every direct memory access must be permitted. The representation provides no checks establishing these conditions.

### Outputs and completion

There is no represented ordinary return value. Observable outputs consist of direct memory changes, dependency invocations, and any effects caused by those dependencies.

If all preceding operations complete, the final visible call is followed by an invalid-instruction termination marker. Whether actual execution traps, faults, or behaves differently because of incorrect decoding is **documented_only**.

### Width and arithmetic interpretation

For the conditional calculations in this contract:

- Words are 32 bits; bytes are 8 bits.
- Arithmetic results and addresses retain their low 32 bits.
- Left shifts discard bits above bit 31.
- Right shifts of unsigned words insert zero bits.
- Byte loads used as selectors produce values from 0 through 255.
- Byte stores retain the low 8 bits.
- Adding 17 to a selector occurs after byte promotion; destinations therefore range from 17 through 272, without 8-bit wraparound.

These conventions describe bit-vector consequences. Several represented signed multiplications and left shifts can overflow or otherwise lack portable signed-language semantics. Their exact executable behavior remains unresolved.

No explicit alignment, bounds, null, or overflow checks are present.

## Behavior

### Notation and ordering

F-D001 through F-D074 are opaque **invocation-site IDs**, not established identities of distinct dependency functions. Different sites may reach the same implementation.

Q0 through Q13 are specification labels for retained dependency results, not source identifiers. Their calculations use the 32-bit interpretation above.

Define two word-access addresses:

- **A(Q):** 20 plus four times Q.
- **B(Q):** 20 plus sixteen times Q.

Each access through A or B is 32 bits wide. These formulas do not establish valid pointers, arrays, allocations, or a larger record layout.

Calls and direct accesses occur in the order described below, except that the relative order of two argument-producing reads is unresolved. Every later operation is conditional on earlier operations completing.

### Initial calls and publication

F-D001 receives the first explicit input. Its target is calculated from the incoming link-register value plus 4,094.

F-D002, F-D003, and F-D004 follow with no explicit arguments represented. F-D005 supplies Q0.

The routine then:

- Stores the incoming implicit 32-bit register value at A(Q0).
- Invokes F-D006 with four times Q0.

No visible validation separates Q0’s production from its use as an address component.

### Word operations and associated calls

Each preparation group in the following table contains four invocations in ascending ID order. No explicit arguments are represented for those calls. The final invocation in each group supplies the indicated result.

Direct effects within a row precede its following invocation. That invocation precedes the next preparation group.

| Preparation group | Result | Direct memory effects | Following invocation |
|---|---|---|---|
| F-D007–F-D010 | Q1 | Store four times Q1 at B(Q1); perform two represented word reads at A(Q1). | F-D011 receives one read shifted right by 12, four times Q1, and the other read shifted left by 2. |
| F-D012–F-D015 | Q2 | Store four times Q1 at A(Q2). | F-D016 receives four times Q2. |
| F-D017–F-D020 | Q3 | Store four times Q3 at B(Q3). | F-D021 receives four times Q3 twice, then the fourth explicit input shifted left by 26. |
| F-D022–F-D025 | Q4 | Store four times Q3 at A(Q4). | F-D026 receives four times Q4. |
| F-D027–F-D030 | Q5 | Store four times Q5 at B(Q5). | F-D031 receives four times Q5. |
| F-D032–F-D035 | Q6 | Store four times Q5 at A(Q6). | F-D036 receives four times Q6. |
| F-D037–F-D040 | Q7 | Store four times Q7 at B(Q7), then read and retain the word at A(Q7). | F-D041 receives four times Q7. |
| F-D042–F-D045 | Q8 | Store the retained word logically shifted right by 12 at A(Q8). | F-D046 receives four times Q8. |
| F-D047–F-D050 | Q9 | Store four times Q9 at B(Q9). | F-D051 receives the three arguments specified below. |
| F-D052–F-D055 | Q10 | Store four times Q9 at A(Q10). | F-D056 receives four times Q10. |

F-D051 receives, in positional order:

1. The low six bits of four times Q8, shifted left by 14.
2. Four times Q9.
3. Q8 shifted left by 30.

Under the conditional bit-vector interpretation, the first argument equals Q8’s low four bits multiplied by 65,536. Its possible values are therefore 0 through 983,040 in increments of 65,536. The third argument depends only on Q8’s low two bits. Neither argument is derived from Q9 or the retained word.

### Word-read ordering and retention

F-D011’s first and third arguments contain separately represented word reads from A(Q1). Both reads follow the store at B(Q1) and precede the invocation. Their relative order is not established.

The contract does not require the two reads to observe the same value. Nor does the representation establish whether actual executable accesses remain separate or are merged. An implementation-level atomic snapshot is not supported.

The read associated with Q7 is different: its result is explicitly retained before F-D041. The later store at A(Q8) uses that retained result shifted right by 12, not a fresh read from A(Q7). Intervening dependencies may alter memory without changing which represented value supplies this store. This does not establish any broader register-preservation or calling-convention guarantee.

Calculated word addresses are not guaranteed distinct. In particular, B(Q) and A(Q) can coincide under 32-bit modular arithmetic. Where they coincide, the represented store precedes the associated read or reads; other causes of memory change remain unconstrained.

### First selector read and byte store

F-D057 through F-D060 execute next without explicit arguments represented. F-D060 supplies unsigned 32-bit Q11.

Define the **selector address** as:

**S = 1 + Q8 × 268,435,456**, retaining the low 32 bits.

The routine reads an unsigned byte at S. It adds 17 to that byte’s value and stores the low byte of Q11 shifted left by 2 at the resulting destination.

Consequences under the stated interpretation:

- S depends only on Q8’s low four bits.
- S has the form 1 + k × 268,435,456, where k is from 0 through 15.
- The destination is from 17 through 272.
- The stored byte’s low two bits are zero.
- The selector is used as a numeric address selector, not as an offset from an established object base.

F-D061 follows the byte store. Its first and second arguments are Q11 shifted left by 2; its third argument is Q11 shifted left by 4.

F-D061 uses a represented target displacement of 4,076 rather than the recurring displacement of 4,094. The evidence does not justify treating this difference as an error or normalizing it away.

### Intermediate forwarding

F-D062 is invoked without explicit arguments represented. Its 32-bit result is passed directly to F-D063.

No direct memory access appears between production and forwarding of that result. It is not established to be a pointer, owned object, allocation, or resource handle.

F-D064 through F-D067 then execute without explicit arguments represented. F-D067 supplies Q12.

### Second selector read and byte store

The routine performs a fresh unsigned byte read at S. It does not reuse the first selector value.

The new selector plus 17 determines the second byte-store destination. The stored value is the low byte of Q11 after masking to its low 30 bits and logically shifting right by 20. Thus the stored byte contains Q11’s original bits 20 through 27. Original bits 28 and 29 are discarded by byte truncation; bits 30 and 31 are removed by the mask.

F-D068 then receives Q12 shifted left by 2.

The second selector may differ because intervening dependencies can modify memory. No preservation of selector memory is established.

The first byte store **cannot directly overwrite S through calculated-address equality** under the stated interpretation. If k is zero, S is 1, below every byte-store destination. If k is nonzero, S is at least 268,435,457, above every destination. The same argument excludes equality between S and the second byte-store destination.

Aliasing through distinct mapped addresses, device side effects, or other external memory behavior is **documented_only**. None is established by these address calculations.

### Final calls

F-D069 through F-D072 execute without explicit arguments represented. F-D072 supplies Q13.

F-D073 receives:

1. Q13 shifted left by 2.
2. The zero-extended selector value obtained by the second selector read.

The second argument is retained across intervening calls; no additional selector read is represented.

F-D074 follows without explicit arguments represented. The visible prefix then ends at the invalid-instruction termination marker.

## Side effects

If execution reaches the represented endpoint, the visible body performs:

- Eleven 32-bit stores.
- Three represented 32-bit loads: two for F-D011 and one retained-word load.
- Two unsigned byte loads.
- Two byte stores.
- Seventy-four indirect invocations.

These counts exclude all operations inside dependencies. The load count describes represented accesses, not verified machine-level transactions.

Direct writes use calculated addresses without visible validation. The representation establishes neither their allocation boundaries nor their relationship to any object. Endianness is unspecified; the word-store descriptions concern 32-bit values, not an established byte layout.

No direct allocation, release, ownership transfer, synchronization, or cleanup is visible. This absence imposes no such restrictions on dependencies. A dependency may allocate or release resources, mutate global state or memory, perform I/O, or fail to return.

There is no visible rollback, error-result check, retry, exception handler, or recovery branch. A faulting access or non-returning dependency prevents the later prefix from being required; earlier effects have no represented rollback mechanism.

Reentrancy, thread safety, idempotence, and purity are not established.

## Dependencies

The dependency IDs identify sites solely within this contract. No independently verified dependency definitions or ABI declarations were supplied.

| IDs | Represented role |
|---|---|
| F-D001 | Forward the first input through an incoming-state-dependent target. |
| F-D002–F-D005 | Initial preparation; F-D005 supplies Q0. |
| F-D006, F-D016, F-D026, F-D036, F-D046, F-D056 | Consume four-scaled results after stores at A addresses. |
| F-D011, F-D021, F-D031, F-D041, F-D051 | Consume arguments after stores at B addresses and associated calculations. |
| Four-call preparation groups F-D007–F-D060 | Supply Q1 through Q11 at the endpoints identified in Behavior. |
| F-D061 | Consume three shifted Q11 arguments after the first byte store. |
| F-D062–F-D063 | Produce and forward a 32-bit result. |
| F-D064–F-D067 | Supply Q12. |
| F-D068 | Consume shifted Q12 after the second byte store. |
| F-D069–F-D072 | Supply Q13. |
| F-D073 | Consume shifted Q13 and the retained second selector. |
| F-D074 | Final visible invocation. |

“No explicit arguments represented” is not a verified zero-argument calling convention. Hidden register, stack, memory, and global-state inputs remain unknown.

No dependency is assumed to preserve globals, preserve memory, own a returned value, or release resources.

## Acceptance examples

These are conditional static examples, not executed tests. They assume returning dependencies, permitted accesses, and the stated 32-bit bit-vector interpretation unless an interruption is the condition being illustrated.

1. **Initial store.** With Q0 equal to 3 and the incoming implicit word equal to 0xA1B2C3D4, the first store writes that word at address 32. F-D006 receives 12.

2. **Different scaling.** With Q1 equal to 2, the store at B(Q1) writes 8 at address 52. Both represented F-D011 source reads use address 28.

3. **Separate observations.** If F-D011’s right-shift source observes 0x12345000 and its left-shift source observes 3, its arguments are 0x00012345, four times Q1, and 12. This does not prescribe which source read occurs first.

4. **Retained-word use.** If the Q7-associated read yields 0xABCDEF01, the later store at A(Q8) writes 0x000ABCDE, even if intervening dependencies change the original memory location.

5. **Fourth-input width.** A fourth-input bit pattern of 64 produces zero after a 26-position left shift at 32-bit width. A bit pattern of 1 produces 0x04000000. These are conditional bit-vector results, not assertions of portable signed-shift semantics.

6. **Q8-derived arguments.** With Q8 equal to 5, F-D051’s first argument is 0x00050000 and its third is 0x40000000. Its second argument remains four times Q9.

7. **Selector-address separation.** With Q8’s low four bits equal to zero, S is 1; even the lowest byte-store destination is 17. With those bits equal to one, S is 268,435,457; even the highest destination is only 272. No permitted k makes these calculated addresses equal.

8. **Changed selector.** If the first selector is 7 and an intervening dependency changes selector memory so the second read yields 9, the stores target addresses 24 and 26. F-D073 receives 9 as its second argument.

9. **Stable selector without external changes.** If both selector reads observe unchanged ordinary memory, both stores target the same destination. The first store cannot itself change S through equal calculated addresses. This example assumes no distinct-address mapping alias or device side effect.

10. **Byte extraction.** With Q11 equal to 0x0AB00003, the first byte store writes 0x0C and the second writes 0xAB. F-D061 receives 0x2AC0000C twice, followed by 0xAB000030.

11. **Interrupted prefix.** If F-D041 does not return, no later direct access or invocation is required. Earlier stores have already occurred, with no visible rollback.

## Uncertainties

The following remain **documented_only**:

- Whether the representation accurately corresponds to executable instructions.
- Actual dependency identities, signatures, hidden arguments, and calling conventions.
- Validity of the first target and the incoming implicit register value.
- Valid address regions, access permissions, alignment, endianness, and memory-map semantics.
- Physical aliasing through distinct calculated addresses and device-induced side effects.
- Dependency effects on globals, memory, registers, resources, and ownership.
- Exact executable results of represented signed overflow and signed left shifts.
- Whether the two F-D011 source loads remain separate machine accesses.
- Actual behavior at the truncated endpoint.
- Application-level success, failure, ownership, or resource-lifetime requirements.

The repeated pattern may suggest mechanically generated sequencing. That is inferred intent only, not an additional behavioral requirement.

## Evidence

This contract is based solely on the supplied static representation. No implementation access, independent executable inspection, runtime observations, or runtime tests were available.

Call ordering, retained-value relationships, access widths, and direct-effect counts follow the represented operations. The relative evaluation order of F-D011’s source reads remains unresolved.

The selector-address separation follows from the declared 32-bit interpretation: S is either 1 or at least 268,435,457, while each byte-store destination lies between 17 and 272. This establishes non-equality of calculated addresses, not a complete external memory-map contract.

Invalid-instruction and truncated-control-flow warnings prevent treating the represented endpoint as proof of complete runtime behavior.
