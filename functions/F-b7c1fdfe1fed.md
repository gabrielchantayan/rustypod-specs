<!-- Trusted provenance: F-b7c1fdfe1fed; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b7c1fdfe1fed

## Purpose

F-b7c1fdfe1fed is a state-mutating routine that forwards one 32-bit input to an initial indirect dependency, performs a long ordered sequence of dependency calls and memory accesses, updates two shared storage locations, and finally attempts to invoke address zero with three zero arguments.

The supplied static representation does not establish a domain-level purpose. In particular, it does not establish that the routine parses a format, constructs an object, performs a supervisor operation, or implements a conventional initialization API. Such interpretations would require additional evidence.

The independently observable contract is therefore an ordered interaction with an external execution environment, not a pure transformation of the explicit input. Successful traversal depends on incoming machine state, dependency-produced values, and writable memory at computed addresses, including low addresses.

No runtime tests have been performed.

## Interface

### Explicit input and result

- **Input:** one opaque 32-bit value. The first dependency invocation receives this value unchanged.
- **Declared result:** no explicit return value.
- **Completion:** an apparent return follows the final invocation of address zero. Ordinary return is possible only if that invocation itself returns and all earlier operations complete. Normal completion is not established.

No pointer, length, enumeration, or ownership interpretation is established for the explicit input.

### Implicit inputs

The routine additionally consumes:

- An incoming control-link value used to compute the first invocation target.
- An incoming 32-bit machine-state value that is written into two early computed memory locations.
- Primary dependency results, sometimes represented as a 64-bit pair with separately consumed lower and upper 32-bit portions.
- Secondary machine-result values used after dependency calls without an independently established public return convention.
- Shared memory, including a byte at address 1 and two distinct global storage locations.
- A stack-relative address supplied in four argument positions during one intermediate invocation.

These are material inputs. The explicit argument alone cannot determine the routine’s behavior.

### Widths and notation

The following notation describes access relationships, not implementation identifiers:

- A **byte read** obtains 8 bits. Where explicitly widened, its value is between 0 and 255.
- A **halfword read or write** accesses 16 bits.
- A **word read or write** accesses 32 bits.
- A **paired result** is represented as 64 bits, split into lower and upper 32-bit words.
- A **selected-byte destination** is obtained by reading one byte from a selector address and then adding 17 to that byte’s numeric value.

Thus, selected-byte destinations lie between addresses 17 and 272 inclusive. They are not pointers loaded from a pointer table.

Arithmetic involving unsigned words is evaluated at 32-bit width. Left shifts discard bits beyond that width; unsigned right shifts introduce zeros. Byte stores retain only the low eight bits.

Several expressions are presented using signed 32-bit intermediates, including multiplication and left shift. Their overflow and negative-input behavior are not independently established. Bit-pattern interpretations of those expressions are conditional on the target machine semantics, not a portable signed-arithmetic guarantee.

### Preconditions

For any particular execution to traverse the entire visible sequence:

1. Each reached invocation target must support the machine-level invocation being attempted.
2. Every reached read and write must be valid for its width and required alignment.
3. Dependency results and secondary result values must be usable in subsequent address calculations.
4. Low-address accesses must be permitted wherever selected-byte destinations or other derived addresses require them.
5. The final invocation of address zero must have defined behavior in the execution environment if return is expected.

The routine contains no visible checks that enforce these conditions.

## Behavior

### Ordering model

The visible body is straight-line: no explicit conditional branch, loop, retry, or early return appears. Its direct operations occur in the displayed order if execution reaches them.

This does not imply unconditional completion. Dependencies may fault, transfer control elsewhere, fail to return, or mutate memory. Memory accesses may also fault.

Dependency calls are observable ordering boundaries. Repeated reads must not be treated as one cached read, and repeated stores must not be removed merely because their values match. An intervening dependency may change the memory being accessed.

### Initial forwarding and word-slot setup

The first invocation forwards the explicit input to a target derived from incoming control-link state.

A following invocation produces a paired result. Its upper word selects a destination of the form:

- four times that word, plus 20 bytes.

The routine writes the incoming implicit 32-bit machine-state value there. It then supplies the paired result’s lower word in the first, second, and fourth argument positions of another invocation, with zero in the third position.

Further invocations produce indices used for a recurring family of word writes:

- Word slots addressed as four times an index plus 20.
- Larger-stride slots addressed as sixteen times an index plus 20.
- Values equal to four times an index, a previously retained scaled value, the incoming implicit value, or zero.

An early sequence first stores the incoming implicit value in another four-stride slot, then builds and propagates scaled values through sixteen-stride and four-stride slots, and eventually writes zero to a four-stride slot.

These address families do not establish an array or object layout. A scaled address passed to a dependency is often the corresponding base without the additional 20-byte slot displacement.

### Initial selected-byte operations and payload reads

A dependency-produced unsigned word then controls several byte-selector reads.

The first such operation masks that word to its low 30 bits, shifts the result right by 16, adds one, and reads a selector byte. The selected destination receives the low byte of four times the original word.

Later selected destinations receive zero. Their selector addresses use the retained word after masking and shifting by 17 or by 7, as well as a direct read from address 1.

Between these writes, the routine:

- Supplies zero explicitly to one dependency.
- Passes scaled dependency results to later invocations.
- Reads a byte from the address obtained by shifting the retained unsigned word left by three.
- Widens that byte and supplies the same value in four argument positions.
- Supplies a secondary result value together with zero to another invocation.
- Passes two identical scaled values and a value formed by shifting the earlier byte left by 31.

For the last expression, only the byte’s least significant bit can survive in a 32-bit result. It produces either zero or the top-bit-only word.

A later dependency receives a secondary result shifted right by 18. Its primary result is repeated in four argument positions.

### Intermediate word, byte, and halfword reads

Subsequent paired results introduce new retained bases and values. Selected-byte zeroing continues, with selector addresses derived from:

- A secondary result shifted right by nine, plus one.
- A retained lower result plus one.
- That lower result shifted right by nine, plus one.
- Later secondary results plus one or shifted right by nine, plus one.

One direct word read uses a retained lower result as the base and adds an offset formed by shifting that same result right by 19 and then left by 31. At unsigned 32-bit width, the offset is either zero or the top-bit-only word, selected by original bit 19. The word read is repeated in four argument positions.

The next payload stages include:

- A byte read at offset 14 from a secondary-result-derived base.
- A primary result repeated in four argument positions.
- A selected-byte zero write whose selector address uses a retained value multiplied by 4096, plus one.
- A dependency call carrying a secondary result in three positions and its unsigned right shift by 19 in another.
- One invocation receiving the same stack-relative address in all four argument positions.
- A byte read at the sum of a retained base and a secondary result.
- A halfword read at the sum of a later secondary result shifted right by 19 and the previously read byte.

The halfword is supplied in four argument positions. Its signedness is not established by the untyped 16-bit representation.

A following paired result drives a word store to a four-stride slot using the pair’s upper word as index. The stored value is the previously retained right-shifted quantity. Further invocations and a second slot write propagate that quantity.

No bounds, null, or alignment checks precede these reads.

### Direct byte fields and larger-stride storage

A retained lower result is subsequently used as a direct base for two byte stores at offset 13:

1. The low byte of twice the base value.
2. The constant 4.

Dependency calls intervene. The second store does not erase the observability of the first.

During this stage, two word reads use distinct address constructions:

- Four times a retained unsigned word, plus 68.
- The same word masked to 30 bits, shifted right by 20, plus 68.

A later word write uses a destination of 256 times a dependency result plus 20 and stores four times the retained unsigned word. The associated invocation receives the result scaled by 64 in two argument positions, with a third argument derived from a masked right shift followed by a left shift of 31.

Another four-stride slot receives a value masked to 26 bits and shifted right by 13.

Several further groups propagate scaled indices through sixteen-stride and four-stride slots. Some invocations also receive an index shifted left by 24 or 22. These are width-sensitive bit transformations, not established byte-count conversions.

### Linked-base updates and offset-21 bytes

A later sixteen-stride region receives a zero word at displacement 36. A word at displacement 116 from the same scaled region is then read and becomes a new direct base.

After additional invocations:

- A dependency result shifted right by 20 is stored at offset 36 from that base.
- A word at offset 116 is read and passed onward.
- A selected-byte destination receives a truncated masked-and-shifted value.
- A later selected-byte zero store rereads the selector at the direct base plus one.

The shifted result is at most 4095. Shifting it right by a further 21 therefore supplies zero to the relevant invocations, regardless of its original value.

Three payload bytes are read from offset 21 of separately obtained secondary-result bases. Dependencies intervene between these reads. Several selected-byte zero writes use a selector address based on the retained right-shifted result multiplied by 1024, plus one.

The third payload byte is widened and written to a four-stride word slot. A secondary result then propagates through another slot.

A separate unsigned value is masked to 29 bits, shifted right by 18, and later shifted left by 13. That transformed value is passed to a dependency and stored in a word slot.

### Zero-derived state and low-address reads

Later stages explicitly retain zero-valued quantities while continuing to invoke dependencies.

One selected-byte write obtains its selector from address 1 because its local selector expression is zero plus one. This is an actual byte read at that point, not a guarantee that the selector equals an earlier read from address 1.

The routine then:

- Performs additional result-forwarding invocations.
- Writes a returned word into a secondary-result-indexed slot.
- Propagates scaled results through further slots.
- Reads a word from a four-stride slot, shifts it right by 21, and passes it onward.
- Writes another scaled secondary-result-derived value into a slot.
- Sets the first shared global storage location to zero.

After further calls and slot updates, a retained zero causes another zero-valued argument and additional selector reads from address 1.

An explicitly located dependency receives the same primary result twice. Later payload reads include a byte at a result-derived base plus one, followed by a byte at the earlier byte’s numeric value plus one.

The latter address is between 1 and 256 inclusive. It is not a second dereference through a full-width pointer.

Several local quantities derived from retained zero remain zero throughout these arithmetic transformations. One later unsigned quantity is masked to 30 bits and shifted right by 19, then shifted right by 22; that final value is necessarily zero. The routine still performs the associated invocation and word store.

### Late linked-state propagation

The late portion continues alternating dependency invocations with:

- Four-stride and sixteen-stride word writes.
- Selector reads followed by byte writes.
- Shifted-result arguments.
- Reads at offsets 20, 52, and 116.
- Byte writes at offsets 1 and 9.

One direct byte destination is a scaled result multiplied by 8192, plus 9. It receives zero. A later direct base plus 9 receives the low byte of that base.

A paired result supplies a scaled lower word and an upper word to a dependency, together with a word read at offset 52 from a retained base.

Several linked word reads follow: a word at offset 20 of a secondary-result-derived base; a word at offset 20 of that word shifted right by 22; and another word at offset 20 from a subsequently retained value. These are separate reads with intervening stores and calls.

The first shared global is reread well after its earlier zero assignment. Its value is not guaranteed still to be zero: intervening dependencies may have modified it.

For one local value, two successive unsigned right shifts by 23 force the second result to zero. This remains true even when the reread global was nonzero.

### Second global update and terminal sequence

A late primary result is assigned to the second shared global before being passed to a dependency. Further invocations occur afterward, so that assignment is not a guaranteed final global value.

The routine repeatedly reads words at offset 116 and writes their low bytes to offset 1 of retained or scaled bases. Repeated reads remain distinct; dependencies intervene.

Additional calls receive arguments formed by left shifts of 11, 12, 16, or 18. Later word writes propagate retained values, scaled results, and zero. One destination uses a retained unsigned word with its low two bits cleared, then adds 20.

The first shared global is read again. A word at offset 20 from that read value is obtained after intervening invocations. Bytes derived from this word are written through selector-based destinations.

The final visible sequence is:

1. Obtain a primary result and pass it to a dependency.
2. Invoke another dependency and retain an 8-bit result.
3. Write zero through a secondary-result pointer.
4. Assign the second shared global a 32-bit composition whose low byte is that 8-bit result and whose upper 24 bits come from separate shared storage.
5. Write a zero halfword at the address formed by masking a retained unsigned result to 30 bits and shifting right by seven.
6. Attempt to invoke address zero with three zero arguments.
7. Reach the apparent return only if that invocation returns.

The source does not establish that address zero denotes a valid dependency, intentional trap, or runtime service.

## Side effects

### Memory and shared state

The routine directly writes bytes, halfwords, and words to computed addresses. It also writes both shared globals and later rereads the first.

The shared storage has an overlap warning in the supplied material. Distinct symbolic storage references must therefore not be assumed disjoint from smaller accesses.

Selected-byte stores can alias each other or other low-address storage. Direct address calculations may also alias. The contract does not grant non-aliasing.

For assignments that read and write within one expression, the required source value is obtained before the destination is overwritten. No broader ordering between independent argument-expression reads is established here.

### Ownership and lifetime

No allocation, release, retain, or ownership-transfer contract is established for any dependency.

Passing an address does not establish transfer of ownership. In particular, the four repeated stack-relative arguments do not establish four allocations or four independently owned objects. Whether the receiving dependency retains that address is documented_only.

### Errors and atomicity

There is no explicit error result, validation failure path, rollback, or cleanup path.

If a call or access fails, earlier writes may already be visible. The routine is not established to be transactional, reentrant, thread-safe, or safe under concurrent mutation.

## Dependencies

Only F-b7c1fdfe1fed has a supplied resolved opaque identity. Other dependency identities are **documented_only**.

For discussion, the following opaque labels classify unresolved invocation sites; they do not claim shared target identity:

| Opaque label | Invocation class | Established interaction |
|---|---|---|
| D-001 | Incoming-state-derived initial site | Receives the explicit 32-bit input. |
| D-002 | Repeated computed sites | Receive zero to four visible arguments; some produce primary, paired, or secondary consumed results. |
| D-003 | Explicitly located intermediate site | Receives the same primary result in two positions. |
| D-004 | Terminal zero-address site | Receives three zeros; return behavior unknown. |

No dependency is assumed to preserve globals, preserve memory, release memory, or follow a conventional allocator or object-management protocol.

Repeated similar invocation forms are not evidence that they call the same function.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests.

1. **Input forwarding.** Given an explicit input of 0x89ABCDEF and a valid initial target, the first invocation receives that exact 32-bit pattern.

2. **Early paired result.** If the initial paired result has upper word 3, the first direct word store targets address 32 and stores the incoming implicit word, assuming that address is writable.

3. **Byte-selected destination.** If a selector read returns 255, its associated byte store targets address 272. It does not target an address loaded from a word-sized table entry.

4. **Truncated scaled byte.** If the retained unsigned word in the first selector stage is 0x41, four times that value is 0x104. The stored byte is 0x04.

5. **High-bit-only offset.** A retained word with bit 19 clear contributes zero to the described word-read offset. With bit 19 set, it contributes 0x80000000 at 32-bit width. Both cases still require the resulting address to be readable.

6. **Intervening selector mutation.** If a dependency changes address 1 between two selector reads, the corresponding selected-byte stores may target different addresses. An implementation that caches the earlier selector is not equivalent.

7. **Intermediate offset-13 state.** An observer within an intervening dependency may see the first byte written at offset 13 before the later constant 4 is stored.

8. **Global mutation.** If a dependency modifies the first shared global after its zero assignment, later reads use the modified value. Treating the earlier zero as permanent is not equivalent.

9. **Terminal failure.** If address zero is not callable, reaching the final invocation does not produce an ordinary successful return. Earlier writes remain part of the observed execution.

## Uncertainties

The following are documented_only:

- Domain purpose and input interpretation.
- Actual machine ABI and secondary-result conventions.
- Dependency identities and behavior beyond visible argument use.
- Endianness, alignment rules, memory-map permissions, and fault delivery.
- Signed-overflow and negative signed-shift behavior.
- Exact global storage extents and overlap relationships.
- Ownership, allocation, retention, and release behavior.
- Stack-address retention by dependencies.
- Meaning and validity of the final zero-address invocation.
- Whether the static representation faithfully recovers every invocation boundary.

Possible initialization or state-construction intent is inference only. It is not an acceptance requirement.

## Evidence

This contract is based exclusively on the supplied static representation.

Established evidence includes the explicit 32-bit input, absence of an explicit result, visible access widths, argument relationships, straight-line ordering, mask-and-shift arithmetic, repeated shared-state accesses, and terminal invocation of address zero.

Arithmetic conclusions about bounded unsigned shifts follow from the displayed widths. They do not establish that the resulting addresses are valid.

No implementation inspection, dependency-body inspection, runtime execution, or behavioral testing has been performed.
