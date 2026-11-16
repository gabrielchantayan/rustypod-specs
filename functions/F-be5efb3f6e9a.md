<!-- Trusted provenance: F-be5efb3f6e9a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-be5efb3f6e9a

## Purpose

The established behavior is a linear sequence of indirect calls, byte and 32-bit memory accesses, and arithmetic transformations. The supplied static evidence does not establish an application-level purpose.

Repeated stores at fixed offsets suggest structured storage, and repeated zero stores suggest clearing state. These are **inferred intent**, not established allocation, initialization, serialization, or destruction semantics.

The supplied representation includes warnings about invalid instruction data and overlapping global storage, and ends at an invalid-instruction marker. This contract therefore distinguishes represented operations from a guarantee that an executable instruction stream performs them. Unresolved behavior is identified as **documented_only**.

No runtime tests have been performed.

## Interface

### Explicit inputs

| Input | Represented width | Established use |
|---|---:|---|
| First argument | 32 bits | No explicit use shown. |
| Second argument | 32 bits | Supplied twice to the initial indirect call. |
| Third argument | 32 bits | No explicit use shown. |
| Fourth argument | Unsigned 32 bits | Logically shifted right by three before subsequent use. |

**Reduced input** means the fourth argument divided by eight, discarding the remainder. Its range is 0 through 536,870,911.

The initial dependency receives the second argument, the second argument again, and reduced input, in that order. A later dependency receives a freshly read word followed by reduced input twice.

The absence of explicit uses of the first and third arguments does not establish that the actual calling convention ignores them. Complete interface recovery is **documented_only**.

### Ambient inputs and preconditions

Additional inputs include incoming control state, an ambient 32-bit extraction value, an ambient address, secondary outputs not expressed as ordinary returns, memory contents, and dependency results. The initial indirect target is represented as incoming control state plus 4,094; the validity and interpretation of this computation are unresolved.

Reaching any operation requires all preceding calls to return and preceding accesses to complete. Call targets must support execution, and memory locations must support the represented access widths. No checks enforcing these conditions appear.

### Outputs and termination

No ordinary return value is declared. The visible sequence ends at an invalid-instruction termination marker rather than a normal return.

No success result, recoverable error result, or normal-completion guarantee is established. Actual exception delivery and behavior beyond the endpoint are **documented_only**.

### Widths and address model

A **byte** is eight bits; a **word** is 32 bits. Two represented paired results are 64 bits, separated into low and high 32-bit words.

Byte stores retain the low eight bits. Selector bytes are zero-extended to 0 through 255 before address formation.

Where overflow matters, arithmetic conclusions below are conditional on **32-bit modular arithmetic**. The representation mixes signed-looking and unsigned-looking operations; it does not establish portable signed-overflow behavior, address wrapping, alignment requirements, or byte order.

Addresses below describe byte-addressed locations. A word access occupies four consecutive bytes under the ordinary flat-address model. These numerical layouts do not establish valid mapped storage.

## Behavior

### Ordering and dependency boundaries

No conditional branch, loop, explicit bounds check, null check, or error branch appears in the visible region.

In this contract, **F-unknown** denotes an unresolved dependency occurrence. Separate occurrences are separate call boundaries; the notation does not assert a shared target. Calls described as having “no displayed arguments” are not established as having no hidden inputs.

The following phases preserve direct-access order and significant result provenance. Intervening calls remain observable boundaries even where their arguments and results are not described. They cannot be assumed to preserve memory or global state.

### Initial dispatch and byte writes

After the initial three-argument call and further dependencies, a returned scalar is obtained. Before passing four times that scalar to another dependency, the function reads a byte at the ambient address plus 1 and stores a byte at the selector plus 17.

The stored byte comes from the ambient extraction value logically shifted right by 22, retaining bits 22 through 29. The selector-selected destination ranges from address 17 through 272.

After further calls, four times a newly returned scalar establishes **first base**. A word is read at first base plus 116; its low byte is stored at first base plus 1. The word at first base plus 116 is then read again. The next relevant dependency receives this second read, reduced input, and reduced input.

These are two distinct reads. Under ordinary flat-address arithmetic, the intervening byte store does not overlap the word.

A separate no-displayed-argument dependency produces a word forwarded to another dependency. After further calls, four times another scalar establishes **second base**. A word at second base plus 116 supplies the low byte stored at second base plus 1. The next dependency receives second base.

### High-fragment propagation

Define:

- **High fragment**: first base logically shifted right by 25; range 0 through 127.
- **High placement**: high fragment shifted left by 23.
- **Wide scale**: high placement logically shifted right by three, equal to high fragment multiplied by 1,048,576.

Following dependencies, the function stores high fragment multiplied by 524,288 as a word at four times a represented secondary output plus 20. A dependency then receives that secondary output.

A later dependency supplies a 64-bit result. Another receives its low word, high word, and low word again.

After further calls, a scalar selects a word destination at four times that scalar plus 20. The stored value is the then-represented secondary output shifted left by three. A dependency receives four times the scalar. Preservation of the earlier secondary-output value across calls is not established.

Next, wide scale multiplied by 524,288 is stored at wide scale multiplied by 2,097,152 plus 20. A dependency receives the stored arithmetic value three times.

Under 32-bit modular arithmetic, both products are zero. This operation therefore stores a zero word at address 20 and passes three zero arguments. After further calls, wide scale itself is stored at four times a new scalar plus 20, followed by a dependency receiving four times that scalar.

### Retained word and saved selector

A later result becomes **retained word**; **retained scale** is four times retained word.

A byte is read at wide scale plus 1. The low byte of retained scale is stored at that selector plus 17. The next dependency receives the selector logically shifted right by three, retained scale, and four times wide scale.

After intervening calls, including another scalar-producing call, the selector location is read again. This fresh byte becomes **saved selector**. At saved selector plus 17, the function stores the low byte of retained word masked to 30 bits and logically shifted right by 20. A dependency then receives four times the newly produced scalar.

After further calls, retained scale is stored as a word at four times saved selector plus 20. The next relevant dependency receives saved selector three times, followed by saved selector logically shifted right by one.

A subsequent scalar selects another word at four times that scalar plus 20. It receives retained scale before a dependency receives four times the scalar.

### Intermediate scaled-value propagation

The next group obtains a scalar, stores four times it at sixteen times it plus 20, and passes the four-times value to a dependency. **Small scale** is established as saved selector logically shifted right by three, ranging from 0 through 31.

After intervening calls, the retained four-times value from this group is stored at four times another scalar plus 20. A dependency receives four times this destination-selecting scalar.

Another group obtains a scalar and establishes **propagated scale** as four times it. Propagated scale is stored at sixteen times its originating scalar plus 20 and passed to a dependency. A separate following no-displayed-argument call produces a word forwarded to another dependency.

After further calls, propagated scale masked to 30 bits and logically shifted right by one is stored at four times a new scalar plus 20. A dependency receives four times that new scalar.

A further group stores four times its scalar at sixteen times that scalar plus 20, then reads a word at four times the scalar plus 20. The read word, logically shifted right by three, is passed to a dependency. The read address is not generally the preceding store address.

After additional calls, the retained four-times value from this last group is stored at four times another scalar plus 20, followed by a dependency receiving four times the new scalar.

### Later selector stores

A new result becomes **later word**. Saved selector is stored as a word at small scale multiplied by 524,288 plus 36.

Define **later base** as sixteen times later word, and **half fragment** as later word masked to 30 bits and logically shifted right by one. Half fragment ranges from 0 through 536,870,911.

A dependency receives later base and four times later word. Following intervening calls, saved selector is stored as a word at later base plus 36. A word is then read at later base plus 116 and passed to a dependency.

After another call sequence, a returned scalar becomes **shift source**. A byte is read at later base plus 1. At that selector plus 17, the function stores the low byte of later word masked to 30 bits and logically shifted right by 20. A dependency receives four times shift source.

A separate following producer supplies a word forwarded to another dependency. After additional calls and another scalar result, the byte at later base plus 1 is freshly read. A zero byte is stored at the new selector plus 17, followed by a dependency receiving four times the new scalar.

### Additional byte clearing

Following further calls, the function stores a zero byte at half fragment plus 15.

**Clear base** is half fragment multiplied by 1,024. A byte is read at clear base plus 1, and a zero byte is stored at that selector plus 17.

A dependency receives shift source shifted left by 23 three times, followed by shift source shifted left by 25. A separate following producer supplies a word forwarded to another dependency.

After further calls, including another scalar result, the byte at clear base plus 1 is read again and retained as **clear selector**. A zero byte is stored at clear selector plus 17. A dependency receives four times the new scalar.

Following additional calls, a zero word is stored at clear selector plus 68, then a zero word is stored at absolute address 36.

### Offset-68 clearing and result provenance

The next relevant dependency receives zero, 152, clear base, and 152. Its return value is not explicitly consumed.

A separate no-displayed-argument dependency then produces a word forwarded to another dependency. A later no-displayed-argument dependency produces a second word forwarded onward. Neither forwarded word is established as the result of the earlier four-argument call.

After further calls, a scalar selects a zero-word store at four times that scalar plus 68. A zero word is then stored at address 36, followed by a dependency receiving four times the scalar.

Later, a dependency supplies a 64-bit paired result. Another dependency receives four times its low word. After that call, zero words are stored first at twice the saved high word plus 68, then at address 36.

The next dependency receives zero, a represented secondary output, and four times that secondary output.

**Masked half fragment** is half fragment masked to 22 bits. After an intervening call, a dependency receives masked half fragment shifted left by seven.

Additional calls lead to another zero-word store at four times a new scalar plus 68, then another zero-word store at address 36, followed by a dependency receiving four times the scalar.

There are four distinct explicit zero-word stores at address 36 across this clearing sequence.

### Final offset-20 propagation

A scalar-producing dependency is followed by a dependency receiving four times that scalar. This second call itself produces the scalar used to select a zero-word store at four times its value plus 20.

The next relevant dependency receives this latest scalar three times, followed by masked half fragment shifted left by seven.

After more calls, a zero word is stored at four times a new scalar plus 20, followed by a dependency receiving four times the scalar.

The next propagation group stores four times a scalar at sixteen times its originating scalar plus 20. After intervening calls, that retained value is stored at four times another scalar plus 20. Each store is followed by a dependency receiving the relevant four-times value.

A further group stores four times its scalar at sixteen times that scalar plus 20. A dependency receives both four times and sixteen times the scalar; its return value is not explicitly consumed.

A separate following producer supplies a word. Another dependency receives that word and masked half fragment shifted left by four.

After more calls, masked half fragment multiplied by 128 is stored at four times a new scalar plus 20. A dependency receives four times the scalar.

### Final byte-selected writes and endpoint

A returned word becomes **final word**. **Final selector base** is half fragment multiplied by 67,108,864.

A byte is read at final selector base plus 1. The low byte of four times final word is stored at that selector plus 17. A dependency receives four times final word; its return value is not explicitly consumed. A separate following producer supplies a word forwarded to another dependency.

After further calls, the selector location is read again. The low byte of final word masked to 30 bits and logically shifted right by 20 is stored at the newly selected address plus 17. A dependency receives four times a newly returned scalar.

After more calls, another result becomes **last word**. The selector location is read again, and the low byte of four times last word is stored at that selector plus 17. The selector is passed to a dependency. A separate following producer supplies a word forwarded onward.

Following further calls, a byte is read at eight times last word plus 1. At that selector plus 17, the function stores the low byte of last word masked to 30 bits and logically shifted right by 20.

A dependency receives four times another returned scalar. Further no-displayed-argument calls precede the invalid-instruction endpoint.

## Side effects

Direct effects comprise the described byte and word reads and writes. Byte-selected destinations range from 17 through 272. The clear-selector word destination ranges from 68 through 323 and need not be aligned.

Low numerical addresses do not establish valid ordinary RAM. Permissions, memory-mapped behavior, access faults, and the significance of overlapping global storage remain unresolved.

Dependencies may observe preceding stores, modify memory or globals, fail, or not return. No dependency is assumed to preserve global state. Fresh reads remain distinct observations; repeated stores separated by calls cannot be merged on the assumption that intervening calls are inert.

No allocation, release, ownership transfer, reference-counting, synchronization, atomicity, or rollback contract is established. An access failure or non-returning dependency may prevent every later effect. No compensating action is shown.

## Dependencies

**F-be5efb3f6e9a** is the only supplied resolved opaque function identity.

**F-unknown** is a contract-local opaque placeholder for independently unresolved dependency occurrences, not an assertion that they identify one function. Whether different occurrences share a target is **documented_only**.

Displayed arguments and consumed results establish operand flow only. Complete signatures, hidden inputs, secondary-output conventions, memory effects, ownership behavior, and termination guarantees are unresolved.

A dependency whose return is not explicitly consumed must not be treated as the producer for a separate following call.

## Acceptance examples

These examples are conditional static consequences, not observed executions.

1. **Input reduction.** Fourth arguments 0 through 7 produce reduced input 0; 8 through 15 produce 1; 4,294,967,295 produces 536,870,911.

2. **Initial extraction.** If the ambient extraction value has only bit 22 set, the first stored byte is 1. If only bit 30 is set, it is 0.

3. **Separate word reads.** With first base 4, both early word reads address 120, while the intervening byte store addresses 5. The dependency receives the second read. Stable, ordinary-memory contents are sufficient for equal read values; equality does not establish stability. The intervening byte store does not overlap the word under the stated address model.

4. **Equal observations despite changes.** If the word location contains 7 at each read but changes to 9 and back between them, both reads can return 7. Their equality does not exclude intervening mutation.

5. **Fresh selectors.** If a dependency changes a selector byte from 5 to 9 before a subsequent fresh read, that read selects destination 26 rather than 22.

6. **Byte truncation.** A scaled value of 256 supplies byte 0 to a byte store, while a word store of that same value retains 256.

7. **Modular collapse.** Under 32-bit modular arithmetic, the wide-scale product store writes zero at address 20 for every possible high fragment.

8. **Clearing-call provenance.** If the four-argument clearing dependency leaves an incidental result of 7 but the separate following producer returns 11, the next forwarding consumer receives 11. A later producer returning 19 supplies 19 to its consumer.

9. **Final-call provenance.** If the dependency receiving four times final word leaves an incidental result of 23 and the separate following producer returns 29, the forwarding consumer receives 29.

10. **Distinct clears.** The four explicit stores to address 36 remain separate effects. Dependencies between them may observe or change that storage.

11. **Endpoint.** An ordinary successful return is not an established acceptance expectation.

## Uncertainties

The following remain **documented_only**:

- Application purpose and data-structure meanings.
- Executability and completeness of the represented instruction sequence.
- Actual calling convention, hidden inputs, and secondary-output provenance.
- Dependency identities, effects, memory ownership, and termination.
- Address validity, alignment, byte order, aliasing, and device-memory effects.
- Arithmetic behavior outside the conditional 32-bit modular model.
- Concurrency, reentrancy, and external mutation.
- Fault handling and continuation beyond the endpoint.

Equal read values establish neither stability nor absence of intervening writes. Conversely, different read values are not explained by self-overlap of the early byte store under the stated address model.

## Evidence

Established static evidence consists of the supplied representation and its diagnostic warnings. It supports the represented widths, masks, shifts, constants, access order, explicit call operands, consumed results, and invalid-instruction endpoint.

Arithmetic simplifications are derived from those operations under the expressly stated width model. Structured-storage and clearing interpretations are inferred intent only.

Prior prose and reviewer feedback are not independent behavioral evidence. The read-equality distinction follows from the two separate observations and their nonoverlapping intervening byte store; no memory-stability assumption is established.

No runtime tests, external dependency verification, or independent executable inspection have been performed.
