<!-- Trusted provenance: F-2af62ce18af3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2af62ce18af3

## Purpose

F-2af62ce18af3 represents an ordered sequence of calls, integer transformations, selector-directed byte writes, and indexed 32-bit writes. One direct dependency is identified as F-674e75a953e8; the remaining calls have unresolved identities.

No application-level purpose is established. Repeated offsets and scaled addresses suggest manipulation of structured state, but describing the operation as initialization, serialization, allocation, decoding, or device configuration would exceed the evidence.

The supplied static representation includes warnings about invalid instruction data and overlapping global storage. Its displayed path ends at an invalid-instruction boundary, not an ordinary return. This contract therefore distinguishes:

- **Established static evidence:** operations and data relationships explicitly represented.
- **Conditional consequences:** results obtained when those operations are interpreted using 32-bit bitvector arithmetic.
- **documented_only unknowns:** execution properties and dependency contracts not established by the supplied material.

These distinctions apply throughout. The contract does not assert that the displayed path is reachable or executes successfully.

## Interface

### Explicit arguments

The displayed interface has four 32-bit arguments and no ordinary return value.

| Argument | Represented use |
|---|---|
| First | Passed to the first indirect call. |
| Second | No explicit body use is shown. |
| Third | Shifted left by 26 bits and passed as the second argument of an indirect call. |
| Fourth | Shifted left by 10 bits and passed as the third argument of another indirect call. |

The apparent lack of use of the second argument does not establish its irrelevance to the actual binary interface. Interface recovery is uncertain.

### Additional inputs

Behavior also depends on execution state not supplied by the four arguments:

- An incoming link-related value used to calculate the first call target.
- An incoming address-like value used for the first byte read.
- An incoming 32-bit value supplying the first stored byte.
- Auxiliary register outputs associated with several calls.
- Two reads of unresolved, overlapping global storage.
- Ordinary results from indirect calls, including one represented 64-bit result.
- Contents and accessibility of every calculated memory address.

The auxiliary outputs have no established calling-convention contract. References below to a retained value describe the displayed data flow, not a guarantee that an unknown callee preserves a physical register.

### Widths and arithmetic

Byte accesses are 8 bits. Word accesses, explicit arguments, and most arithmetic are 32 bits. One call result is represented as 64 bits and separated into upper and lower 32-bit halves.

Conditional arithmetic consequences use modulo-\(2^{32}\) addition, multiplication, and left shifts. High bits are discarded; calculations are not unbounded integer arithmetic. This model does not assert that every signed expression in a higher-level language has defined wrapping behavior.

Right shifts of values represented as unsigned are logical. Byte writes retain only the lowest eight bits of their supplied value. Read selectors are zero-extended before address formation.

Endianness, address mapping, alignment requirements, and memory permissions are unresolved.

### Completion

No successful return value, status code, or recovery interface is represented. The displayed path reaches a terminal invalid-instruction marker after its final indirect call.

That marker does not identify a platform-specific exception, signal, or termination mechanism. Earlier calls or memory accesses may also prevent execution from reaching it.

## Behavior

### Ordering and interpretation

The displayed body is linear, with no explicit loop or conditional branch. Its order of calls and memory effects is significant.

Unknown calls are not inert separators. They may modify memory or globals, expose auxiliary outputs, transfer control elsewhere, or fail to return. Every later effect is conditional on preceding execution reaching it.

Repeated reads remain distinct accesses. When two memory reads occur as arguments to the same call, the representation establishes both read expressions but does not establish their relative evaluation order.

### Entry and first selector-directed write

The first indirect target is formed from the incoming link-related value plus 0xFFE. The call receives the first explicit argument. Further indirect calls precede the first explicit memory access.

A byte is then read at the incoming address-like value plus 1. Its zero-extended value selects an absolute byte-write address by adding 0x11. The original read base is not added to the write address.

The byte written is the low eight bits of the incoming data value shifted logically right by 22. Consequently, every selector-directed destination in this contract lies numerically between 0x11 and 0x110 inclusive.

A previously obtained call result, shifted left by two, is passed onward. After intervening calls, another call receives the captured selector and the third explicit argument shifted left by 26.

### Initial word and global samples

A later indirect call supplies a 32-bit word. Another call receives that word twice, followed by the fourth explicit argument shifted left by 10.

After an intervening call, the word is stored at four times an auxiliary output plus 0x14. The word remains the source of a later write.

The unresolved global location is then read twice:

- The first sample is retained without scaling.
- The second sample is multiplied by four to form a retained base.

These are distinct reads, not an established atomic snapshot. No intervening call is shown between them, but their equality and concurrency semantics are not guaranteed by the supplied evidence.

Following further calls, a returned index selects a destination at four times that index plus 0x14. The earlier word shifted logically right by 12 is stored there. A subsequent call receives four times that index.

### Byte writes using the global-derived base

Another indirect call supplies a new 32-bit word.

A selector is read at the retained global-derived base plus 1. The selected destination receives the low byte of the new word shifted left by two. The following call receives the first global sample masked with 0x3FFFFFFF and shifted logically right by 10.

After intervening calls, the selector address is read again. The newly selected destination receives the low byte of the new word masked with 0x3FFFFFFF and shifted logically right by 20.

Later calls consume a returned value shifted left by two and, subsequently, another shifted result together with the second selector.

Neither a shared selector nor a shared destination may be assumed for the two writes.

### Collapsed base and auxiliary-output writes

The retained global-derived base is shifted left by 26. Two represented 32-bit reads at that shifted base plus 0x1C supply the two arguments of an indirect call. Their relative evaluation order and equality are not established.

Multiplying the shifted base by 0x400 produces a further base. Under the 32-bit model, that base is necessarily zero: shifting left by 26 and then multiplying by \(2^{10}\) discards all bits, irrespective of the earlier scaling.

After intervening calls, the next selector read is therefore at address 1, and the selected byte destination receives zero. The following call receives zero, an auxiliary output, and the zero-valued base.

After further calls, address 1 is read again. The selected destination receives the low byte of the auxiliary output shifted logically right by 22.

Subsequent calls consume a shifted return value and later receive the zero-valued base shifted left by 26 together with another return value shifted left by two.

### Direct dependency and paired word writes

An indirect result is passed twice to F-674e75a953e8. The direct dependency’s ordinary return value is not explicitly consumed.

After another indirect call, the supplied value is written at four times a new auxiliary output plus 0x14. The next call receives that auxiliary output and its masked form, using mask 0x3FFFFFFF followed by a logical right shift of 10.

Following more calls, the auxiliary output itself is stored at four times a returned index plus 0x14. A later call receives four times that index.

Nothing establishes whether F-674e75a953e8 preserves globals, changes memory, acquires resources, or releases resources.

### Further byte updates

A later call supplies a new word. A selector read at eight times the value supplied to F-674e75a953e8, plus 1, selects a destination for the low byte of four times the new word. The following call also receives four times that word.

After intervening calls, a selector is read at 64 times the new word plus 1. Its destination receives the low byte of the word masked with 0x3FFFFFFF and shifted logically right by 20. These two selector addresses use different source values and different multipliers.

Later calls consume shifted results and the second selector. Another call receives the earlier direct-dependency argument shifted left by 10 in both positions.

A separate value explicitly set to zero forms the next selector-read address. That read is at address 1, and the selected destination receives zero.

A subsequent call receives another auxiliary output twice, followed by 16 times that output. After more calls, a selector is read at 16 times the auxiliary output plus 1. Its destination receives the low byte of that output shifted logically right by 22.

Later calls consume shifted results and this selector. A separately zero-initialized value, shifted left by 10, is then passed twice to another call.

### Split 64-bit result

The next call is represented as returning 64 bits. The following effects use its upper and lower 32-bit halves:

| Order | Represented effect |
|---|---|
| First | Store zero at four times the upper half plus 0x14. |
| Next | Pass the upper half to an indirect call. |
| Later | Store the upper half at four times a returned index plus 0x14; pass four times that index onward. |
| Later | Store the lower half multiplied by 0x4000000 at the lower half multiplied by 0x10000000, plus 0x14. |
| Next | Pass the lower half multiplied by 0x4000000 to a call. |
| Later | Store the lower half masked with 0x1FFFFFFF and shifted logically right by 9 at four times another returned index plus 0x14; pass four times that index onward. |

Intervening call sequences remain part of the ordering. The large multipliers retain only limited low-order information under the 32-bit model.

### Five scaled-value stores and their distinct transitions

The later body contains five instances of the same address/value relationship: a call supplies a source value, four times that value is retained as its scaled value, and the scaled value is written at 16 times the source value plus 0x14.

The transitions are **not all equivalent**:

| Instance | Behavior before the next instance |
|---|---|
| First | A call consumes two separate word reads from the scaled value plus 0x14, respectively shifted right by 12 and left by two, with the scaled value as the middle argument. After further calls, the first scaled value is stored at four times a returned index plus 0x14. |
| Second | A call receives the second scaled value twice and the earlier 64-bit result’s lower half shifted left by 26. After further calls, the second scaled value is stored at four times a returned index plus 0x14. |
| Third | A call receives the third scaled value. After further calls, the third scaled value is stored at four times a returned index plus 0x14. |
| Fourth | A word is captured from the fourth scaled value plus 0x14 before the following call. After intervening calls, the captured word shifted logically right by 12 is stored at four times a returned index plus 0x14. **No intervening indexed store of the fourth scaled value is represented.** |
| Fifth | A call receives three derived arguments described below. After further calls, the fifth scaled value is stored at four times a newly returned index plus 0x14. |

Each intervening indexed write is followed by a call receiving four times its destination index.

The fourth-to-fifth transition retains the index used for the captured-word write and its four-times-index base. Following the fifth scaled-value store, a call receives:

- The low six bits of that retained base, shifted left by 14.
- The fifth scaled value.
- The retained captured-word destination index shifted left by 30.

The later indexed store of the fifth scaled value is distinct from the captured-word write and remains part of the represented behavior.

For every instance, the scaled-value store address is 16 times the source plus 0x14, whereas the additional word-read address, where present, is four times the source plus 0x14. These are not generally the same address.

### Final byte pair and boundary

After the indexed store of the fifth scaled value and further calls, another call supplies a word.

A selector is read at the retained captured-word destination index multiplied by 0x10000000, plus 1. Its destination receives the low byte of the new word shifted left by two.

The next call receives that shifted word twice and the new word shifted left by four. A following call result is forwarded to another call.

After more calls, the selector address is read again. The selected destination receives the low byte of the new word masked with 0x3FFFFFFF and shifted logically right by 20.

Further calls consume shifted results and the second selector. A final indirect call precedes the invalid-instruction boundary. No normal completion follows in the supplied representation.

## Side effects

### Memory layout and aliasing

Explicit access families include:

| Access family | Width | Address relationship |
|---|---:|---|
| Selector read | 8 bits | Derived base plus 1 |
| Selector-directed write | 8 bits | Zero-extended selector plus 0x11 |
| Indexed word write | 32 bits | Four times an index plus 0x14 |
| Scaled-value word write | 32 bits | 16 times a source value plus 0x14 |
| Shifted-base word reads | 32 bits | Shifted base plus 0x1C |
| Scaled-base word reads | 32 bits | Four times a source value plus 0x14 |
| Global samples | Represented as 32 bits | Unresolved overlapping storage |

These relationships do not establish an array, record, independent object, or valid allocation.

Aliasing is unrestricted by the evidence. Selector-directed destinations may overlap other accessed storage. Calls may modify locations later read. Repeated reads cannot be collapsed into one cached read without additional justification.

### Ownership and state

No allocation, deallocation, ownership transfer, reference-count protocol, or lifetime guarantee is established. Dependencies may perform such operations; their behavior is unknown.

Completed writes are explicit state changes. No rollback, transaction, or cleanup mechanism is represented.

### Preconditions and errors

The path contains no explicit bounds, null, permission, alignment, or indirect-target validation.

To continue through a reached operation, the relevant call target must be executable and the accessed memory must support the required operation. In particular, reads at address 1 are arithmetic consequences, not represented null checks.

Faults, dependency failures, non-returning calls, and the terminal invalid instruction have no established reporting or recovery contract.

## Dependencies

- **F-674e75a953e8:** Called once with the same 32-bit value in both argument positions. Its ordinary return value is not explicitly used. Its effects, preservation rules, ownership behavior, and completion behavior are documented_only.
- **Unresolved indirect dependencies:** Targets are computed using incoming execution state or represented address values. Most target calculations add 0xFFE; exceptional calculations add 0x670 or 0xFEC. These offsets do not establish dependency identities or semantics.

No additional opaque function IDs are supplied. Unknown targets are therefore left unidentified rather than assigned invented contracts.

## Acceptance examples

These are conditional static consequences, not runtime test results. Each assumes execution reaches the operation and the required memory is accessible.

1. **Selector addressing.** A selector value of 0x20 selects address 0x31. It does not select the read base plus 0x31.

2. **Byte truncation.** An incoming first-write value of 0xABCDEF01 shifted logically right by 22 yields 0x2AF. The stored byte is 0xAF.

3. **Argument-width loss.** A third explicit argument of 0x40 becomes zero when shifted left by 26 under 32-bit arithmetic. A value of 1 becomes 0x04000000.

4. **Independent global samples.** If the first global read yields 3 and the second yields 5, the retained unscaled sample is 3 and the scaled base is 20. Substituting one shared sample changes the represented data flow.

5. **Collapsed base.** For every second global sample, shifting its four-times-scaled base left by 26 and multiplying by 0x400 yields zero. The corresponding selector reads therefore use address 1.

6. **Split-result stores.** If the 64-bit result has upper half 7 and lower half 3, its first associated write stores zero at 0x30. Its later lower-half-derived write stores 0x0C000000 at 0x30000014.

7. **Fourth transition.** Suppose the fourth scaled-store source is 2. It produces scaled value 8 and writes 8 at 0x34. The subsequent word read is at 0x1C. If that read captures 0x12345000 and the later destination index is 7, the transition writes 0x00012345 at 0x30—not 8. No additional indexed write of the fourth scaled value is required by the representation.

8. **Fifth forwarding write.** If the fifth source is 3, its scaled value is 12 and its scaled-store destination is 0x44. If its later forwarding destination index is 9, the subsequent indexed store writes 12 at 0x38.

9. **Selector reread.** If a selector changes from 0x20 to 0x40 between paired reads, the corresponding writes select 0x31 and 0x51. A common destination is not guaranteed.

10. **Completion limitation.** Reaching the terminal boundary does not satisfy a successful-return requirement. No such return is represented.

## Uncertainties

The following remain **documented_only**:

- Validity of the recovered instruction stream, function boundary, and interface.
- Reachability of the displayed linear path.
- Identities and contracts of indirect dependencies.
- Origins and stability of auxiliary register outputs.
- Actual identity, width, ownership, and aliasing of overlapping global storage.
- Whether low numeric addresses are mapped memory or artifacts of unreliable recovery.
- Endianness, alignment, volatility, atomicity, and concurrency behavior.
- Relative evaluation order of the two reads supplied to a single call.
- Platform-visible behavior at the terminal invalid instruction.
- Application-level purpose, ownership protocol, and postcondition.

The first three scaled-store transitions forward their preceding scaled values. The fourth instead writes a shifted captured word. This distinction is established by the displayed data flow, not left unresolved.

## Evidence

This contract is based solely on the supplied static representation and dependency mapping. The prior draft and reviewer feedback are not independent execution evidence.

Static evidence supports the displayed argument uses, memory-access widths and relationships, call/access ordering, repeated reads, direct dependency, split 64-bit result, five scaled-value stores, distinct fourth transition, fifth forwarding write, and terminal invalid-instruction boundary.

The invalid-instruction and overlapping-global warnings limit conclusions about actual execution. Arithmetic examples are conditional on the stated 32-bit bitvector model.

No dependency implementation inspection, runtime execution, or runtime tests have been performed.
