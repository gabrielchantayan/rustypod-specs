<!-- Trusted provenance: F-e1b2edcad297; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e1b2edcad297

## Purpose

F-e1b2edcad297 describes a state-mutating operation that interleaves dependency calls, memory transfers, address calculations, and bit-field extraction. The supplied static representation does not establish an application-level purpose such as parsing, allocation, serialization, or device configuration.

This is a **conditional execution contract**: it specifies observable relationships for reached portions of the represented path, assuming the required machine state and accesses are valid. It does not establish that execution completes.

The displayed path ends at an invalid-instruction boundary, without a normal return. Reported decoding and control-flow problems limit whole-function conclusions. They do not justify discarding the displayed distinctions between captured values, later reads, and ordered writes.

## Interface

### Explicit inputs

| Input | Represented width | Established use |
|---|---:|---|
| First explicit argument | 32 bits, uninterpreted | No explicit use appears in the displayed body. |
| Second explicit argument | Signed 32-bit integer | Selects the opening word-store destination and is passed to the first indirect dependency. |

The unused appearance of the first argument does not establish that its incoming machine register is irrelevant to unresolved calls.

### Implicit inputs

Execution also depends on:

- An incoming 32-bit payload for the opening store.
- An incoming address for a word read at offset `0x14`.
- An incoming 32-bit selector-source base, repeatedly accessed at offset `1`.
- An incoming control-transfer value from which the first indirect target is calculated.
- Secondary machine-register values observed after dependency calls.
- Existing memory, including the byte at absolute address `1`, the word at absolute address `0x28`, and an additional late-sampled global word whose address is not supplied.
- Memory addressed through dependency results and values derived from earlier reads.

These are substantive inputs, not established defaults. The operation cannot be treated as context-independent or fully described by its two explicit arguments.

### Outputs and completion

No meaningful return value is established. Visible outputs consist of memory writes and dependency invocations, including their argument values and ordering.

There is no displayed success status, error status, or normal-return postcondition. Behavior at the terminal invalid-instruction boundary is `documented_only`.

### Preconditions

For any represented prefix to execute as described:

- Every reached memory address must permit its displayed access width.
- Every reached dependency target must be executable under the actual calling convention.
- Required incoming machine state must be initialized.
- Dependency-produced address components must permit subsequent accesses.
- Dependencies must return if execution is to continue beyond them.

The body displays no general null, range, capacity, alignment, or executable-target validation. Requiring every address to be nonzero would contradict the represented stores to absolute address zero.

## Behavior

### Widths and arithmetic

The representation distinguishes 8-bit byte accesses, 16-bit halfword stores, 32-bit word accesses and address components, and some 64-bit dependency results.

Unsigned 32-bit calculations retain the low 32 bits. Logical right shifts introduce zero bits. Byte and halfword stores retain only the low 8 or 16 payload bits, respectively.

Some calculations instead pass through signed 32-bit values, including scaling and left shifts used for addresses. Their exact overflow, conversion, and signed-shift behavior is not established as a portable arithmetic contract. Where relevant, the underlying machine semantics remain `documented_only`; an unbounded-integer interpretation is not justified.

Splitting a displayed 64-bit result into low and high 32-bit components establishes a value-level relationship. It does not establish endianness or a public return-register convention.

### Opening transition

The opening word store targets four times the second explicit argument plus `0x14`, using the implicit incoming payload. A word is then read from the incoming word-source address plus `0x14`. Only afterward is the first indirect dependency invoked with the second explicit argument.

The store therefore precedes the read. If their addresses coincide, the read can observe the store; no non-aliasing requirement is displayed.

After intervening calls, a returned integer selects a word destination through multiplication by four and addition of `0x14`. The stored payload is the captured opening word logically shifted right by 11.

A subsequent word read uses the opening capture logically shifted right by 22, plus `0x14`, as its address. There is **no additional multiplication by four** for that read. The resulting word later contributes a left-shifted dependency argument.

### Selector-derived byte writes

A recurring operation reads one unsigned selector byte, zero-extends it, and stores one byte at the numeric selector value plus `0x11`. These destinations range from `0x11` through `0x110`, inclusive.

The selector is an address component, not a pointer-sized load. Selector sources include:

- The incoming selector base plus `1`.
- Absolute address `1`.
- Dependency-derived values scaled by four, sixteen, or other powers of two, plus `1`.
- Masked and shifted values plus `1`.
- Word-loaded address components plus `1`.

Payloads include zero, low bytes of scaled values, low bytes of captured words, and extracted fields.

| Transformation before byte truncation | Original bits retained by the byte store |
|---|---|
| Logical right shift by 22 | Bits 22–29 |
| Mask with `0x3fffffff`, then logical right shift by 20 | Bits 20–27 |
| Mask with `0x1fffffff`, then logical right shift by 19 | Bits 19–26 |
| Mask with `0x3fffff`, then logical right shift by 12 | Bits 12–19 |
| Unsigned multiplication by four | Low eight bits of the product |

A separate payload comes from a 64-bit result logically shifted right by 54; the byte store retains original bits 54–61.

Selector locations are often reread after calls. Capturing one selector does not authorize replacing another displayed read with that capture. Calls may change memory, and external mutation is not excluded.

### Absolute-address `0x28` sampling

This region contains **three displayed word reads** from absolute address `0x28`, with distinct roles.

The first read is captured before an indirect call. That call then receives two additional displayed reads of `0x28` in its two argument positions. The captured first value is **not** the displayed source of either argument.

The relative evaluation order of the two argument reads is unspecified. Both are separate from the earlier capture. Equality among all three samples requires an additional memory-stability assumption.

After that call and a further dependency invocation, the operation stores the original capture at a destination formed from a secondary-register-derived integer multiplied by four, plus `0x14`. This store does not refresh its payload from `0x28`.

Thus, mutation of `0x28` may affect the call arguments without changing the later stored capture.

### Absolute-address `1` as dependency input

A later indirect call supplies the byte at absolute address `1` in both argument positions through **two displayed reads**. This is not an established single capture reused twice.

The reads have unspecified relative argument-evaluation order. No dependency contract establishes equal samples or excludes external mutation during their evaluation.

After this call, another dependency is invoked before a selector-derived zero-byte store. That store reads its own selector from an address calculated as a previously retained low result multiplied by `0x2000000`, plus `1`. It is not established to reuse either argument byte from absolute address `1`.

Subsequent calls precede another read of that computed selector source and a field-byte write. Those selector reads also remain distinct.

### Captured words and dependency-derived layouts

Several regions preserve a direct capture across calls:

- A word read at a secondary-register-derived base plus `0x14` is supplied twice to a dependency. Its retained value later contributes shifted word and byte payloads.
- A dependency result is stored as a word at absolute address `0x7c` before the same result is supplied in both argument positions to a following dependency.
- A 64-bit result supplies a high component used for ordered word reads at offsets `8` and `0x0c`. The first word supplies a later byte payload. The second supplies an address component for a later selector read.
- The low component of that earlier result supplies a base for a separate word read at offset `4`. Its low byte is written later through the selector mechanism.

The ordered reads at offsets `8` and `0x0c` occur before intervening dependency calls. Their later uses consume captured values, not newly established reads of those fields.

Another 64-bit result supplies a low-byte payload and a high-component destination base; the byte destination is that base plus `0x1c`.

Elsewhere, a byte is stored directly at an address derived by masking a retained low component with `0x3fffffff` and logically shifting right by 7. A displayed zero-valued contribution does not add an offset there.

These uses establish access widths and offsets, not structure names, allocation extents, or ownership.

### Repeated word stores and distinct scaling layouts

Word destinations include:

- Four times a selected integer plus `0x14`.
- Sixteen times a selected integer plus `0x14`.
- A derived base plus `0x34`.
- Absolute locations.

These layouts are not interchangeable.

Two consecutive-store pairs are explicit:

- At a base derived by logical right shift by 9, plus `0x34`, a zero-extended selector is stored as a word, followed by twice that selector at the same address.
- At another base plus `0x34`, the same integer shifted left by two is stored twice.

Neither pair contains an intervening dependency call. Both writes belong to the represented access sequence, even when ordinary memory ultimately retains only the second value.

Later groups propagate scaled integers and extracted fields through additional `0x14` destinations. Some store four times an integer at sixteen times that integer plus `0x14`, then propagate the scaled value to a separately selected four-times-integer destination.

Another region stores a low result multiplied by `0x4000000` at that result multiplied by `0x10000000`, plus `0x14`. These unsigned products are 32-bit products, not widened address calculations.

### Late-sampled global word

After a word store through a secondary-register-derived destination scaled by four and offset by `0x14`, the representation performs **two separate reads of an additional global word before the next dependency call**.

The samples have different propagation paths:

- The **first sample**, retained as an unsigned 32-bit value, later supplies a dependency argument after masking with `0x3fffffff` and logically shifting right by 10.
- The **second sample** is multiplied by four and retained through a signed 32-bit intermediate. That scaled value supplies selector-source addresses and later shifted address calculations.

Between sampling and those later uses, dependencies are invoked and another word is stored through a result-selected `0x14` destination. The first sample remains the source of its masked argument; it is not reread immediately before that call.

The scaled second sample selects a byte source at offset `1`. After further calls, the same source expression is read again for another selector-derived write. The first write uses the low byte of a dependency result multiplied by four; the later write uses that result masked with `0x3fffffff` and logically shifted right by 20.

The retained scaled second sample is subsequently shifted left by 26 through a signed intermediate. Two displayed word reads at this shifted base plus `0x1c` become the two arguments of another indirect dependency. Their relative evaluation order is unspecified; neither is established as a captured word reused twice.

The shifted base is also multiplied by `0x400`, producing a later selector-source base and dependency inputs. Further left shifts by 26 occur downstream.

These calculations must not be widened silently or collapsed using portable signed-arithmetic assumptions. Under an additional low-32-bit machine-arithmetic interpretation, some compounded shifts and products discard all bits. The supplied representation alone does not resolve every signed operation sufficiently to make that simplification unconditional.

The two original global samples must not be merged. Equality requires memory stability between the reads. Later dependency effects do not retroactively change a capture, but neither do captures prove that the global remains unchanged.

### Literal zero-address stores

A displayed intermediate is set to zero and has no represented reassignment before a word store through that value multiplied by eight. The destination is therefore absolute address zero.

After intervening calls and selector writes, a halfword destination is formed from the same zero-valued intermediate by masking and logical right shifting. This also targets address zero.

If reached, the word store precedes the halfword store. Their payloads are distinct dependency-produced values; the halfword payload is truncated to 16 bits. Endianness and intervening effects prevent asserting a final combined word value.

### Final transfer regions

A late dependency receives two separate reads of one word location: one read is logically shifted right by 12, while the other is shifted left by two. Their relative evaluation order is unspecified. Replacing them with one sample requires additional memory-stability evidence.

A different later word is explicitly captured before calls and eventually stored, logically shifted right by 12, through another selected destination. This capture must remain distinct from the two-read argument case.

The final visible region interleaves selector reads, field-byte stores, and dependencies. It publishes no final status before the invalid-instruction boundary.

## Side effects

Direct effects include 8-bit and 32-bit reads, 8-bit and 32-bit writes, and a 16-bit write. Some accesses target low absolute addresses; others depend on incoming state, sampled memory, dependency results, or secondary registers.

Dependencies occur between many direct accesses. Their memory, register, control-flow, allocation, and release effects are unknown. In particular, no dependency is assumed to preserve selectors, globals, or memory later accessed through retained addresses.

No rollback, transaction, synchronization, atomicity, or exclusion of external observers is established. Consecutive writes cannot be reduced to final-state equivalence for write-sensitive memory without additional environmental assumptions.

No ownership transfer is established. Repeating a value across arguments does not prove copying, reference counting, consumption, retention, or release. Allocation extent, borrowed lifetime, and validity of addresses after calls are `documented_only`.

The displayed path does not test dependency results for failure or return a documented error code. Invalid accesses, invalid call targets, or dependency failures may interrupt execution; reporting and recovery behavior are unknown.

## Dependencies

Mapped dependencies occur in this relative order, with additional calls between them:

| Opaque function ID | Established argument relationship |
|---|---|
| F-23325cd5e50c | Receives one captured 32-bit result. |
| F-bbb54b5f036d | Receives one captured 32-bit result twice. |
| F-f80b3ac065fc | Receives a low 32-bit result, its associated high component, and two further copies of the low result. |
| F-de15789688a8 | Receives one captured 32-bit result twice. |
| F-ea83f9a17404 | Receives one captured 32-bit result twice. |
| F-674e75a953e8 | Receives one captured integer twice. |

Their application semantics and preservation, ownership, and completion guarantees are unavailable.

Unmapped dependencies include the calls consuming the two additional `0x28` reads, the two absolute-address-`1` byte reads, the first late-global sample’s masked field, and the two shifted-base-plus-`0x1c` word reads.

Many targets are computed. Two additional fixed-target calls have no supplied opaque IDs. An argument-shape resemblance does not establish identity with a mapped dependency.

Secondary-register outputs remain unresolved interface elements, not established conventional return values.

## Acceptance examples

These examples are conditional static criteria, not executed tests.

### Opening alias

If the opening destination equals the opening read address, ordinary coherent memory without external interference permits the read to capture the just-stored payload. Moving the read before the store would not preserve this case.

### Selector boundaries and truncation

A selector of zero yields destination `0x11`; `0xff` yields `0x110`.

For a payload of `0xffffffff`, masking with `0x3fffffff` and shifting right by 20 produces `0x3ff`; the byte store writes `0xff`.

### Three samples at `0x28`

Suppose the original capture is `0x11111111`, while the later argument reads observe `0x22222222` and `0x33333333`. Either assignment of those later samples to argument positions is compatible with the unspecified evaluation order.

The later word-store payload remains `0x11111111`, assuming the represented capture survives and execution reaches the store. Passing that capture twice instead would not preserve the displayed reads.

### Two absolute-address-`1` inputs

If the two argument reads observe `2` and `3`, the dependency may receive those values in either positional order. They cannot unconditionally be replaced by two copies of one sample.

The subsequent selector-derived zero store still performs its own selector read after the intervening call.

### Independent late-global samples

If the first late-global sample is `0x00000400` and the second is zero, the first sample’s masked-right-shift-by-10 argument is `1`. The second sample’s multiplication by four yields zero, making its initial selector-source address `1`.

Merging the samples would lose this combination. A later global change does not refresh either retained sample.

### Paired offset-`0x1c` reads

If memory at the shifted base plus `0x1c` changes between the two argument reads, differing argument values are compatible with the representation. Capturing once and duplicating that value requires an additional stability guarantee.

### Repeated stores and address zero

For a selector of `7`, the first repeated-word pair writes `7`, then `14`, to the same address.

If the zero-address region is reached and permits both widths, a word store at zero precedes a later halfword store there. No endian-specific final word is asserted.

## Uncertainties

The following remain `documented_only`:

- Application purpose, true entry boundary, and complete control flow.
- Actual calling convention and required incoming register state.
- Unmapped dependency identities and all unavailable dependency contracts.
- Declaration and address of the late-sampled global.
- Signed arithmetic, address conversion, endianness, alignment, faults, and atomicity.
- Whether low addresses denote ordinary memory, device state, or representation artifacts.
- Concurrency, external mutation, ownership, allocation extent, and lifetime.
- Reachability and actual behavior of the terminal invalid-instruction boundary.
- Any successful-completion postcondition.

Repeated scaling and extraction may suggest encoded-state manipulation. That is inferred intent, not an acceptance requirement.

## Evidence

Established static evidence consists of displayed access widths, masks, shifts, offsets, dependency argument relationships, captures, distinct read expressions, consecutive stores, and the absence of a normal return.

In particular, the representation distinguishes three reads at `0x28`, two absolute-address-`1` dependency inputs, two late-global samples with different propagation paths, and two subsequent word-read arguments at shifted-base offset `0x1c`.

The supplied representation reports invalid instruction data, removed unreachable control flow, overlapping global symbols, and terminal truncation. These are limitations of the supplied evidence, not independently validated machine behavior.

No implementation access, independent instruction validation, or runtime tests have been performed. Acceptance examples are derived conditions, not observed results.
