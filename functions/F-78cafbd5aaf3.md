<!-- Trusted provenance: F-78cafbd5aaf3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-78cafbd5aaf3

## Purpose

F-78cafbd5aaf3 performs an ordered series of dependency interactions and memory accesses. Incoming state, dependency results, and retained values determine the destinations, stored values, and arguments supplied to later dependencies.

The supplied representation establishes an **interaction-and-memory-effects contract**, not an application-level purpose. It does not establish initialization, decoding, device reset, or clearing of a contiguous buffer.

Three evidence classifications apply throughout:

- **Established static evidence:** explicit access widths, value provenance, argument positions, and ordering in the supplied representation.
- **Inferred intent:** a possible interpretation that imposes no acceptance requirement.
- **documented_only:** behavior not resolved by the supplied evidence.

All progression requirements are conditional on reaching the relevant event. Static access sites do not prove that execution completes those accesses.

## Interface

### Inputs

The represented interface has four positional inputs, each 32 bits wide.

| Input position | Established contribution |
|---|---|
| First | A signed arithmetic source for the entry destination and the two explicit arguments to F-U01. |
| Second | No explicit contribution to calculations in the supplied body. |
| Third | No explicit contribution to calculations in the supplied body. |
| Fourth | A signed arithmetic source for the third argument of one four-argument dependency interaction. |

The absence of explicit use of the second and third inputs does not prove whole-execution independence from them. Unknown dependencies may consume incoming machine state beyond their displayed arguments.

Additional environmental inputs include:

- An incoming 8-bit value used repeatedly during the initial write interval.
- Incoming control-flow state that selects F-U01.
- Primary and secondary dependency-result channels.
- A pointer-like secondary result retained for later numeric address calculations and a direct 32-bit write.
- A separately captured 32-bit secondary scalar.
- The contents of two computed 16-bit memory locations.

These environmental channels are not established additional public parameters.

### Outputs and completion

No explicit return value, error status, output length, or returned allocation is established. Outputs are memory effects, a shared-object assignment, and dependency interactions.

The ending obtains an executable result from F-U03 and attempts to invoke it through F-U04. Ordinary return, trapping, termination, and continuation after this sequence are **documented_only**.

### Environmental prerequisites

Each reached memory access requires accessible storage for its full width and any alignment required by the execution environment. Each invocation requires a valid executable target and compatible calling state.

The retained pointer-like result must designate writable four-byte storage when the late word write is reached. Its earlier availability does not prove continued validity.

No explicit local check establishes bounds, nullness, alignment, ownership, lifetime, arithmetic validity, or executable-target validity. Zero, negative, and large positional inputs have no represented rejection branch. These are environmental prerequisites, not validation performed by the function.

## Behavior

### Observable ordering and access inventory

The direct-access inventory contains:

| Access category | Static sites |
|---|---:|
| 8-bit writes | 64 |
| 16-bit reads | 2 |
| 32-bit write through the retained pointer-like result | 1 |
| Shared-object assignment of unavailable width | 1 |

The byte writes occur in six ordered intervals:

| Interval | Byte-write sites | Stored-value source |
|---|---:|---|
| Incoming-value | 22 | Original incoming byte |
| Zero-value | 6 | Constant zero |
| Destination-derived | 1 | Low byte of that event’s numeric destination |
| Reusable-byte | 13 | One captured byte from an unsigned dependency result |
| Separate-selection | 1 | Bits 19–26 of that same unsigned result |
| Three-bit | 21 | Bits 29–31 of a separately retained arithmetic quantity |

These are event counts, not counts of distinct addresses. Destinations may repeat or overlap.

Dependencies intervene between many writes, including interactions whose primary results are unused. The intervals are not bulk-memory operations: matching values or destinations do not authorize removing intervening calls or consolidating writes.

The first read and shared assignment occur during the reusable-byte interval. The second read and word write occur during the three-bit interval. Further dependency interactions occur after the final byte write and before the terminal sequence.

No explicit input-dependent branch, retry, error recovery, or rollback appears in the represented body. This does not constrain control flow inside dependencies.

### Width and arithmetic requirements

Ordinary scalar calculations are represented at 32 bits. Four dependency results are explicitly 64 bits. Byte writes affect eight bits, each read accesses 16 bits, and the retained-pointer write affects 32 bits.

For the explicitly unsigned 32-bit producer of the reusable byte, scaling by eight, sixteen, and 32768 discards overflow beyond bit 31. Byte narrowing then retains the low eight bits.

That unsigned rule does not establish wrapping semantics for signed calculations. Signed multiplication overflow, invalid signed left shifts, conversion of numeric values to addresses, and unsigned-to-signed interpretation remain **documented_only** where applicable. Scaling descriptions below identify value relationships; they do not silently supply missing execution-environment rules.

Bit numbering is numeric, with bit zero least significant. Numeric bit selection and division of a wide result into portions do not establish memory endianness or a serialized layout.

### Entry and incoming-value interval

The first observable direct access writes the incoming byte to the numeric address twice the first positional input. F-U01 follows, receiving two equal explicit arguments corresponding to that input scaled by 4096.

F-U01’s target is selected using incoming control-flow state with a displacement of 4094. The meaning and validity of that state are **documented_only**.

All 22 writes in this interval retain the original incoming byte as their value source. Intermediate dependency results do not replace it.

Most subsequent destinations are eight times an associated primary result. Their immediate consumers normally receive two copies of that result scaled by 16384. The established exceptions are:

- The second write is followed by the first 64-bit-result producer, receiving a single argument equal to four times the destination’s producer.
- The sixth write uses twice its producer as the destination, followed by duplicated arguments scaled by 4096.
- The seventh write is followed by a four-argument interaction. Arguments one, two, and four equal the destination producer scaled by 16384. Argument three independently equals the fourth positional input scaled by 1024.
- The twenty-second write retains the ordinary eight-times destination but is followed by duplicated arguments scaled by four.

Immediately after the four-argument interaction, another dependency receives a secondary-channel quantity scaled by eight.

The destination producer for the eighteenth write, scaled by 16384, is retained beyond this interval. This **older retained quantity** is distinct from newer dependency results and remains the source of later arithmetic.

Intervening F-U02 interactions include calls with no explicit arguments, unchanged result forwarding, four-times-scaled arguments, duplicated results, and forwarding of results returned by preceding consumers. An unused primary result does not make the interaction unobservable.

### Wide-result argument layouts

Each of the four explicit 64-bit results contributes numeric lower and upper 32-bit portions to a subsequent consumer.

| Wide-result occurrence | Consumer argument order |
|---|---|
| First | Lower, upper, lower |
| Second | Lower, upper, upper |
| Third | Lower, upper, upper |
| Fourth | Lower, upper |

The first occurrence follows the second incoming-value write. The second and third occur between later incoming-value writes. The fourth follows the width-sized-shift checkpoint described below.

These requirements concern positional values. Physical result channels, register placement, and the signed interpretation of individual portions are **documented_only**.

### Zero-value and destination-derived intervals

Each of the six zero writes targets eight times its associated primary result. Its immediate consumer receives two copies of that result scaled by 16384.

No contiguous range, unique destination, or lasting zeroed state is established.

The following destination-derived write also targets eight times a primary result. It stores the low eight bits of its own numeric destination. Its consumer receives two copies of the producer scaled by 16384, followed by the numeric destination as the third argument.

This stored byte may equal a byte written elsewhere, but its provenance and consumer arguments remain independent requirements.

### Reusable byte and retained arithmetic

One unsigned 32-bit dependency result supplies:

- A reusable byte: the low eight bits after scaling by eight.
- The first destination in this interval: the result scaled by sixteen.
- The first consumer’s two equal arguments: the result scaled by 32768.

The captured byte supplies all 13 writes in this interval and the shared-object assignment. Neither newer primary results nor changes to shared storage refresh it.

The remaining destination relationships are:

- The second write uses the older retained quantity scaled by 65536; its consumer receives that older quantity scaled by \(2^{27}\), twice.
- The third through fifth writes use their respective current primary results scaled by 8192; their consumers receive duplicated arguments scaled by \(2^{24}\).
- The sixth write uses a **retained arithmetic quantity**, formed from the older retained quantity by a further factor of 32768. Its destination scales that retained quantity by 4096; its consumer arguments scale it by \(2^{23}\).
- The seventh write is the first read checkpoint.
- The eighth through twelfth writes use current primary results scaled by eight, followed by duplicated arguments scaled by 16384.
- The thirteenth write uses a current result scaled by eight, followed by duplicated arguments scaled by four.

The retained arithmetic quantity is not a later dependency result. Its high bits determine every byte in the final interval.

After the thirteenth reusable-byte write, one forwarding interaction uses twice its primary-result source rather than the frequent factor of four.

### First read and shared assignment

At the seventh reusable-byte write, the same primary result determines both the byte destination, scaled by 8192, and a read base, scaled by 4096.

The byte write precedes a 16-bit read at the read base plus 56 bytes. No dependency invocation intervenes. The next consumer receives the undisplaced read base first and the read value second.

The read accesses two bytes. Its signedness and extension into the argument channel are **documented_only**.

After further dependency interactions, the shared object is assigned from the captured reusable byte. A dependency invocation with two explicit zero arguments immediately follows. Six reusable-byte writes remain after this assignment.

The shared object’s type and width are unavailable. A byte-sized value source therefore does not prove a one-byte transaction or establish the complete converted object representation. No reset meaning or final shared-state value is established.

### Separate selection and final three-bit value

The separate-selection write stores bits 19–26 of the original unsigned reusable-byte producer, with bit 19 becoming the stored byte’s least significant bit. Bits 27 and 28 do not survive byte narrowing.

Its destination is independently derived from a current primary result scaled by eight. Its consumer receives two copies of that current result scaled by four.

Every one of the final 21 byte writes stores bits 29–31 of the retained arithmetic quantity. The stored numeric value is between zero and seven. Later uses preserve this same selection; they do not repeatedly reduce an already reduced value.

Eighteen final-interval destinations use current primary results scaled by eight. Three differ:

- The second uses twice the outcome of the width-sized shift.
- The fourth uses the numeric retained pointer-like value scaled by 256.
- The eleventh uses the captured secondary scalar scaled by 2048.

At the second write, the representation applies a right shift by 32 to a signed 32-bit quantity previously scaled by four. No unconditional numeric outcome is established. Zero, sign propagation, and shift-count masking are not interchangeable permitted assumptions. Its consumer receives two copies of the unresolved outcome scaled by 4096.

At the fourth write, the pointer’s numeric value determines the destination; no word is read through the pointer to obtain that address. The following consumer receives two copies of the pointer’s numeric value scaled by 524288, using the represented 32-bit signed interpretation.

The secondary scalar is captured during the interaction sequence following this fourth write. It remains the source for the eleventh write and its duplicated consumer arguments, which scale it by \(2^{22}\). Four designated consumers before the twelfth through fifteenth writes also receive this same scalar in argument position two; their first arguments independently scale current primary results by four.

For ordinary destinations, immediate consumers normally receive duplicated arguments scaled by 16384. The ninth, sixteenth, and twentieth writes instead have following interactions without explicit arguments. The read and word-write checkpoints impose the additional exceptions below.

Other late dependencies receive distinct secondary-channel quantities as sole explicit arguments. Those channels are not established reads of the shared object or pointer-denoted memory.

### Second read, word write, and ending

The fifteenth three-bit write precedes a 16-bit read without an intervening dependency invocation. The read address is 128 times the numeric retained pointer-like value, plus 32 bytes. Its consumer receives only the read value explicitly.

The displacement is byte-based. Typed-pointer indexing, a structure layout, read signedness, and argument extension are not established.

At the seventeenth three-bit write, one primary result supplies an eight-times byte destination and a word value equal to four times that result. The byte write precedes a 32-bit write through the retained pointer, with no intervening dependency invocation.

The next consumer receives three equal copies of the word value. A later two-argument consumer receives that retained word value in position two, independently of newer primary results.

After further writes and interactions, F-U03 receives zero and a fixed site-identifying token, and its executable result is invoked through F-U04 without explicit arguments. Neither the word write nor this ending is an established success indication.

## Side effects

### Aliasing and persistence

No non-aliasing guarantee exists among byte destinations, read ranges, shared storage, and retained-pointer storage.

Each read observes memory when reached, including effects of earlier overlapping writes and dependency mutations. The late word write may overlap and overwrite the immediately preceding byte write. Its byte-level consequences depend on endianness.

Completed writes do not promise persistent final contents. Dependencies may mutate the same storage afterward. Atomicity, synchronization, barriers, concurrency safety, and reentrancy are **documented_only**.

### Ownership and lifetime

No direct allocation, release, ownership transfer, or cleanup operation is established.

A pointer-like result confers no demonstrated ownership, extent, or lifetime guarantee. Numerically derived destinations do not identify allocation boundaries.

Unknown dependencies must not be assumed to preserve globals, pointed-to contents, or storage lifetime. Every subsequent direct access independently requires valid storage when reached.

### Errors and interruption

No local interpretation of dependency error statuses, error return, recovery, or rollback is represented.

Invalid memory, unsupported arithmetic, invalid executable targets, faults, or nonreturning dependencies may prevent later events. Earlier effects have no represented local undo mechanism. Fault translation, exception handling, and interruption cleanup are **documented_only**.

## Dependencies

Only F-78cafbd5aaf3 has a supplied verified opaque function identity. Other references are document-local opaque IDs:

| ID | Role |
|---|---|
| F-U01 | Entry interaction selected from incoming control-flow state |
| F-U02 | Collection of intervening dependency interactions |
| F-U03 | Terminal exceptional-operation interaction yielding an executable result |
| F-U04 | Invocation of that executable result |

F-U02 does not identify one shared implementation. Equal arguments or similar interaction shapes do not establish equal targets.

Complete signatures, hidden inputs, mutations, preservation obligations, ownership effects, and return guarantees are **documented_only**. “Without explicit arguments” does not mean “without inputs or side effects.”

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Each assumes the relevant prefix is reached, required storage is accessible, dependencies permit continuation, and the stated arithmetic has a supported interpretation.

### Input and value provenance

- First input 3 and incoming byte 165 require the entry write at numeric address 6, followed by F-U01 receiving 12288 in both explicit positions.
- Fourth input 5 requires 5120 in argument position three of the four-argument interaction. The other positions remain destination-producer-derived.
- Changing only positional input two or three changes no directly represented input-based calculation. Whole-execution equivalence is not required.
- Unsigned reusable-byte producer 35 requires byte 24, first destination 560, and duplicated arguments 1146880.
- Destination-derived producer 35 requires destination 280 and byte 24. The matching byte does not permit exchanging these events.

### Width boundaries

- Unsigned reusable-byte producer 32 produces byte zero, not saturation to 255.
- Unsigned producer 268435457 requires first destination 16, byte 8, and duplicated arguments 32768 after 32-bit truncation.
- A separate-selection producer with only bit 26 set yields stored bits representing 128. With only bit 27 set, it yields zero. A signed character interpretation does not turn this into a wider write.
- Conditional on a supported retained arithmetic quantity with highest numeric byte 224, all final-interval writes store seven. This example does not establish that such a quantity is reachable under every signed-arithmetic interpretation.
- Wide results with lower portion 17 and upper portion 34 require consumer layouts 17/34/17, 17/34/34, 17/34/34, and 17/34.

### Ordering, retention, and aliasing

- First-read producer 2 requires byte destination 16384, then read address 8248, then consumer arguments 8192 and the read value.
- Retained pointer numeric value 64 requires the second read at 8224, not at 64.
- Captured secondary scalar 3 requires exceptional destination 6144 and duplicated arguments 12582912. The four designated later consumers retain 3 in argument position two.
- Late word-checkpoint producer 3 requires byte destination 24 and word value 12. If retained-pointer storage begins at address 24, the byte write still precedes the four-byte write. The next consumer receives 12 three times; persistence of the earlier byte is not required.
- If a dependency does not return, obligations for the unreached suffix do not arise. No unconditional destination is required for the width-sized-shift checkpoint, and ordinary return after F-U04 is not required.

## Uncertainties

The following remain **documented_only**:

- Application purpose, complete calling convention, and valid input domain.
- Dependency identities, hidden inputs, mutations, ownership behavior, and continuation.
- Meaning of incoming control-flow state and secondary result channels.
- Signed-arithmetic boundaries, the right shift by 32, and numeric-address conversion.
- Endianness, alignment, and 16-bit read extension.
- Signed interpretation of character-sized narrowed values.
- Shared-object width, conversion behavior, meaning, and final contents.
- Pointer ownership, extent, accessibility, and lifetime.
- Ordinary-memory versus externally observable mapping semantics.
- Concurrency, reentrancy, fault handling, and terminal continuation.

**Inferred intent:** repeated interaction shapes may belong to a larger structured operation. This supplies no additional behavioral requirement.

## Evidence

Established claims derive from static inspection of the supplied representation’s explicit access widths, retained-value relationships, narrowing boundaries, argument positions, and relative event order.

The contract distinguishes static sites from completed transactions, numeric bit layouts from memory layouts, and established unsigned truncation from unresolved signed operations. It does not promote comments, suggestive names, previous prose, or reviewer feedback into behavioral authority.

No dependency implementations or verified execution-environment rules were supplied. The supplied representation alone does not establish behavior for unsupported arithmetic or reconstruct omitted behavior.

No runtime execution or tests have been performed.
