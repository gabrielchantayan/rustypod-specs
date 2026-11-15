<!-- Trusted provenance: F-c4d5c4118cac; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c4d5c4118cac

## Purpose

The supplied static representation establishes an ordered execution prefix containing indirect calls, selector-byte reads, byte and 32-bit word writes, and addresses derived from retained values. It does not establish an application-level purpose.

A recurring operation reads a byte and treats its zero-extended numeric value as an address base, rather than as an offset within the object containing that byte. Other operations copy a word’s low byte, store scaled results, or split a represented 64-bit result into two 32-bit halves. These patterns do not establish an object model, protocol, allocation scheme, or serialization format.

The visible body ends at an invalid-instruction boundary. This contract covers the established prefix, conditional on each preceding access and dependency call completing. It does not promise normal return or successful application-level completion.

## Interface

### Inputs

The displayed interface contains three explicit arguments:

| Argument | Represented width | Established use |
|---|---:|---|
| First | 32 bits | No direct use in the visible body. |
| Second | 32 bits | No direct use in the visible body. |
| Third | Signed 32 bits | Supplied unchanged to calls and shifted left by three bits. |

Absence of direct use does not establish that the first two arguments are irrelevant to the actual entry convention.

Implicit inputs include:

- An incoming 32-bit **context**, used both numerically and as an address.
- An incoming link-register value used in selecting the first indirect target.
- Incompletely represented auxiliary register outputs used later as arguments or address operands.
- Memory reached by the accesses described below.

Accessible memory and callable targets are preconditions for reaching the complete visible prefix. The representation supplies no validation of these conditions.

### Outputs and numeric interpretation

No explicit return value or normal return is established. Observable outputs are direct writes, dependency invocations, and dependency-produced effects.

Unless stated otherwise, words, retained results, and address operands are 32 bits. A **pair result** is represented as 64 bits, with low and high 32-bit halves.

Selector reads are unsigned byte reads. Byte stores retain the low eight bits; word accesses cover 32 bits. Right shifts identified as logical introduce zero bits.

Unsigned 32-bit arithmetic retains the low 32 bits. Where the representation uses signed multiplication, signed addition, or signed left shifts, overflow-sensitive interpretations remain conditional on 32-bit bit-vector behavior; machine-level semantics have not been independently established. Address wrapping, alignment, and byte order are not otherwise inferred.

## Behavior

### Ordering and dependency notation

Every stage is conditional on completion of the preceding prefix. No validation, recovery, rollback, or success branch is visible.

All unresolved callees are designated **F-unknown**. This denotes a collection of unresolved targets, not a claim that every call reaches one function.

An **argumentless call** has no displayed arguments; hidden machine inputs remain possible. A **four-call run** contains exactly four consecutive argumentless calls. When a run supplies a retained result, that result comes from its fourth call.

The sequence below preserves call placement and direct memory-access order. Values described as retained remain distinct from later memory rereads. Dependencies may alter memory and unspecified state; no global or memory preservation is assumed.

### Entry and context-derived accesses

**1. Initial selector.** A byte is read at context plus `0x01`. Zero is written at the byte’s numeric value plus `0x11`. A call receives four arguments: two identical copies of the third explicit argument shifted left three bits, the unchanged third argument, and zero. A four-call run supplies a retained result.

**2. Selector reread.** The byte at context plus `0x01` is read again and retained as the **initial selector**. Its selected byte at offset `0x11` is cleared. A call receives the preceding retained result shifted left two bits, followed by a four-call run.

**3. Initial word copy and pair.** The word at initial selector plus `0x74` is read, and its low byte is written at initial selector plus `0x01`. A call receives twice context, the third explicit argument, context logically shifted right two bits, and initial selector. One argumentless call supplies a pair. The next call receives its low half, high half, repeated low half, and `(context & 0x7fffffff) >> 1`. A four-call run supplies another retained result.

**4. Scaled temporary.** Four times that result forms a temporary base. The word at its offset `0x74` is read and its low byte written at offset `0x01`. A call receives the temporary base, followed by a four-call run.

### Doubled context and loaded addresses

**5. Address-result capture.** A byte read at twice context plus `0x01` selects a byte at offset `0x11`, which is cleared. A call receives an incompletely represented auxiliary value. One argumentless call supplies the **address result**. A call receives that result, followed by a four-call run supplying a separate result.

**6. Doubled-context byte.** The selector at twice context plus `0x01` is reread. Its selected byte at offset `0x11` receives the high byte of the 32-bit doubled-context value, obtained by logical right shift by 24 bits. A call receives the preceding run’s result shifted left two bits. A four-call run follows; four times its result becomes the **scaled base**.

**7. Loaded pointer.** The word at address result plus `0x04` is retained as the **loaded pointer**. A byte read at loaded pointer plus `0x01` selects a location whose offset `0x11` is cleared. A call receives scaled base three times, followed by a four-call run supplying a result.

**8. Small base.** A byte read at `(loaded pointer >> 2) + 0x01` becomes the **small base**. Small base plus `0x11` is cleared. A call receives the preceding run’s result shifted left two bits, followed by a four-call run supplying a result.

### Extraction and word stores

**9. Packed and linked words.** The word at scaled base plus `0x04` is captured as the **packed word**. The word at small base plus `0x74` becomes the **linked word**, whose low byte is then written at small base plus `0x01`. A call receives linked word, two copies of scaled base shifted left 19 bits, and the preceding run’s result shifted left two bits. One argumentless call supplies a **bridge result**. Packed word logically shifted right 22 bits defines the **extracted value**.

**10. Bridge forwarding.** A call receives bridge result and linked word logically shifted right two bits. A four-call run supplies a result.

**11. Copied word.** Four times that result forms another temporary base. The word at offset `0x74` is retained as the **copied word**; its low byte is written at offset `0x01`. A call receives this temporary base, followed by a four-call run.

**12. Auxiliary-address store.** Copied word is stored at four times an incompletely represented auxiliary value plus `0x14`. A call receives that auxiliary value. One argumentless call supplies a result. The next call receives this result and two copies of `(extracted value & 0x1ff) << 21`. A four-call run supplies a result.

**13. Extracted payload.** Extracted value shifted left 23 bits is stored at four times the preceding result plus `0x14`. A call receives four times that result. A four-call run supplies the **scaling result**.

### Masked state and the seven-call overwrite interval

**14. Scaling store.** Four times scaling result is stored at sixteen times scaling result plus `0x14`. The retained **masked scaling result** is scaling result masked with `0x3fffffff`; the **quarter-masked value** is that masked result logically shifted right two bits. A call receives masked scaling result. A four-call run supplies the **index**.

**15. Indexed store.** Masked scaling result is stored at four times index plus `0x14`. A call receives four times index. A four-call run supplies the **second word**.

**16. First paired-location write.** The low byte of eight times index is written at linked word plus `0x01`. Second word shifted left two bits is stored at eight times index plus `0x34`.

A call receives that shifted second word. One argumentless call supplies a result, which is forwarded to another call. A four-call run follows; its fourth call supplies a pair result. The **masked second word** is second word masked with `0x3fffffff`.

**17. Overwrite.** The low byte of eight times index is written at masked second word plus `0x01`. The pair’s high half is then stored at eight times index plus `0x34`, overwriting the location targeted in stage 16.

**Exactly seven calls occur between those two word writes:** the call receiving shifted second word, one argumentless call, the forwarding call, and all four calls of the following run. The result-producing fourth call is included. The byte write at masked second word plus `0x01` also precedes the second word write. Dependencies may modify the location during this interval; preservation of the first payload is not assumed.

A call then receives the pair’s low half shifted left two bits, followed by a four-call run supplying a result.

### Low-address writes and large-base state

**18. Derived low address.** The low byte of sixteen times index is written at `(masked second word >> 20) + 0x11`. The low byte of four times scaling result is written at sixteen times index plus `0x0d`. A call receives two copies of the preceding run’s result shifted left two bits. One argumentless call supplies a result. The next call receives that result unchanged and logically shifted right three bits. A four-call run supplies a result.

**19. Absolute-address write.** Absolute byte address `0x11` receives the low byte of sixteen times index. The byte write at sixteen times index plus `0x0d` is repeated with the low byte of four times scaling result. A call receives the preceding run’s result shifted left two bits. A four-call run supplies the **large-base seed**.

**20. Seed preparation.** Four times scaling result is stored at masked second word plus `0x14`. The low byte of twice masked second word is written at masked second word plus `0x0d`.

The **large base** is 32 times large-base seed; the **large payload** is seed shifted left seven bits; the **seed fragment** is `(seed & 0x07ffffff) >> 17`. Exactly three argumentless calls follow; the third supplies a **fresh result**.

**21. Large-base stores.** Large payload is stored at large base plus `0x14`. The low byte of fresh result shifted left two bits is written at large base plus `0x0d`. A call receives fresh result shifted left two bits, followed by a four-call run.

**22. Fragment and selector stores.** The low byte of twice quarter-masked value is written at seed fragment plus `0x07`. A byte read at large base plus `0x01` selects a byte at offset `0x11`, which receives the low byte of seed fragment. A call receives fresh result shifted left 21 bits three times. One argumentless call supplies the **saved result**. A call receives saved result, followed by a four-call run supplying a result.

### Captured selector and linked updates

**23. Selector capture.** The byte at large base plus `0x01` is captured as the **captured selector**. Captured selector plus `0x11` is cleared. A call receives the preceding run’s result shifted left two bits, followed by a four-call run.

**24. Saved byte and absolute state.** The **saved byte** is the low byte of saved result logically shifted right three bits. It is written at large base plus `0x0d`. Captured selector is then written to absolute byte address `0x0d`.

A call receives twice captured selector, `0x98`, and `0x260`. One argumentless call follows. The next call receives an incompletely represented auxiliary value logically shifted right three bits. A four-call run supplies a result.

**25. Intermediate result.** Another unresolved call receives that result shifted left two bits and supplies a retained result. Saved byte is written at twice captured selector plus `0x0d`. Absolute byte address `0x0d` is then overwritten with `0xc0`.

**26. Link-update result.** A call receives the retained result from stage 25 shifted left two bits. A four-call run supplies the **link-update result**.

**27. First linked update.** Link-update result shifted left two bits is stored at twice captured selector plus `0x24`. The word at twice captured selector plus `0x74` becomes the **final linked base**. A call receives final linked base, twice final linked base, link-update result shifted left five bits, and `(final linked base & 0x7fffffff) >> 2`. A four-call run follows.

**28. Second linked update.** Link-update result shifted left five bits is stored at final linked base plus `0x24`. The word at final linked base plus `0x74` is read and supplied to a call. A four-call run supplies the **first final pair**.

**29. Full saved payload.** Saved result logically shifted right three bits—not merely saved byte—is stored at four times the first final pair’s high half plus `0x14`.

A call receives three copies of the first final pair’s low half shifted left two bits and its high half logically shifted right three bits. One argumentless call supplies the **second final pair**. The next call receives that pair’s low half, high half, repeated low half, and high half logically shifted right three bits. A four-call run supplies a result.

**30. Earlier-pair payload.** The first final pair’s low half shifted left two bits is stored at four times the preceding run’s result plus `0x14`. This payload does not come from the second final pair. A call receives four times the preceding result. A four-call run supplies the **terminal word**.

### Terminal selector accesses

**31. First terminal selector write.** A byte read at final linked base plus `0x01` selects a byte at offset `0x11`, which receives the low byte of terminal word shifted left two bits. Exactly three argumentless calls follow; the third supplies a retained result.

**32. Terminal selector reread.** The selector at final linked base plus `0x01` is reread. Its selected byte receives the low byte of `(terminal word & 0x3fffffff) >> 20`. A call receives the preceding retained result shifted left two bits. A four-call run supplies a result.

**33. Scaled terminal selector.** A byte is read at `(terminal word << 2) × 0x400 + 0x01`. Its selected byte at offset `0x11` is cleared. A call receives terminal word shifted left five bits twice, terminal word shifted left two bits, and the preceding run’s result shifted left two bits. The five-bit-shift description is conditional on 32-bit bit-vector equivalence to the represented successive shifts. A four-call run supplies a result.

**34. Final reread.** The selector at the same derived address is reread and its selected byte at offset `0x11` is cleared again. A call receives the preceding result shifted left two bits. A final four-call run follows. The representation then ends at the invalid-instruction boundary.

## Side effects

The explicit relative access offsets are:

| Width | Offsets |
|---|---|
| Byte | `0x01`, `0x07`, `0x0d`, `0x11` |
| 32-bit word | `0x04`, `0x14`, `0x24`, `0x34`, `0x74` |

These offsets do not establish one structure layout. The two offset-`0x04` reads use different retained bases and supply different values.

A selector-derived offset-`0x11` destination lies numerically between `0x11` and `0x110`, inclusive. Separate direct byte writes target absolute addresses `0x0d` and `0x11`. Other derived addresses may alias these locations or each other. No non-aliasing guarantee is established.

No direct allocation, release, ownership transfer, synchronization, or persistent-storage operation is established. Dependency effects remain unknown. Earlier completed writes may remain observable if a later call does not return or an access fails; no transaction boundary or rollback is shown.

## Dependencies

All calls are represented by the unresolved dependency designation **F-unknown**. Target expressions use the incoming link-register value or embedded address-like values. The supplied dependency map does not establish callable identities, equivalence between targets, or complete signatures.

The displayed argument lists and consumed results are established static evidence. Hidden inputs, auxiliary-output provenance, memory effects, ownership effects, exceptions, and termination behavior are **documented_only**.

A call without displayed arguments is not necessarily input-free. An unused displayed result does not imply an effect-free call. No dependency is assumed to preserve globals or to own, retain, or release memory.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Numeric selector addressing.** If the first selector is `0x20`, the first clear targets byte address `0x31`, not context plus `0x31`.

2. **Reread sensitivity.** If the initial four-call run changes the context selector from `0x20` to `0x21`, the second clear targets `0x32`. Later operations using a captured selector do not substitute a new memory read.

3. **Byte truncation.** A word of `0x123456ab` copied into a byte destination contributes `0xab`. This numeric conversion does not establish memory byte order.

4. **Seven-call overwrite interval.** Suppose index is two and second word is three, with no overlapping direct destinations. The first stage-16 word write places `0x0000000c` at `0x44`. Exactly seven calls occur before the stage-17 overwrite at `0x44`. If the seventh call supplies high half `0x12345678`, that is the second payload. Any intervening dependency writes remain independently observable.

5. **Extraction boundary.** Packed word `0xffffffff` yields extracted value `0x3ff`. The duplicated masked arguments are `0x3fe00000`; the later shifted payload is `0xff800000` under 32-bit arithmetic.

6. **Full word versus saved byte.** Saved result `0x12345678` produces saved byte `0xcf`. The later stage-29 word payload is `0x02468acf`, not `0x000000cf`.

7. **Captured-selector address.** Captured selector `0xff` makes the stage-25 saved-byte destination `0x20b`, even if a dependency subsequently changes the original selector location.

8. **Partial prefix.** If a dependency never returns, subsequent described accesses are not reached. Earlier completed effects are not rolled back by any visible mechanism.

## Uncertainties

The following remain **documented_only**:

- Application purpose and validity of the represented body.
- Complete entry convention and hidden inputs.
- Exact machine semantics of overflow-sensitive signed arithmetic.
- Byte order, alignment, address mapping, permissions, and device-memory behavior.
- Dependency identities, hidden effects, and auxiliary-output provenance.
- Ownership and aliasing relationships.
- Whether the terminal boundary corresponds to an actual trap, malformed input, decoding failure, or unavailable continuation.

No error code, exception contract, normal return, or application-level success criterion is established.

## Evidence

Established static evidence comprises the supplied representation’s access widths, offsets, constants, argument transformations, retained-value uses, pair splitting, call ordering, and invalid-instruction termination warning.

The seven-call interval follows directly from the call receiving shifted second word, the subsequent argumentless call, the forwarding call, and all four calls of the following run. Both offset-`0x34` writes bracket that complete interval.

Terms such as “context,” “selector,” and “linked base” are descriptive labels, not recovered types. An interpretation as state manipulation is inferred intent only.

No implementation access, dependency inspection, instruction-level validation, or runtime tests have been performed.
