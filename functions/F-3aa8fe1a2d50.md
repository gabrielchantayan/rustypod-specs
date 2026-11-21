<!-- Trusted provenance: F-3aa8fe1a2d50; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3aa8fe1a2d50

## Purpose

F-3aa8fe1a2d50 performs a state-dependent sequence of dependency invocations, arithmetic-derived memory accesses, field extraction, and publication of captured values. Its application-level purpose is not established.

The supplied static representation shows propagation of 32-bit quantities through selected word locations and repeated byte-mediated writes: a byte read from one address determines the absolute destination of a subsequent byte store. Other effects include structured word reads, repeated stores to the same field, and overlapping word and halfword stores at absolute address zero.

Packed-data processing is inferred intent, not an established format or protocol. No allocator, serializer, decoder, object model, or device interface is identified.

This contract describes conditional consequences of the supplied representation. Invalid instruction data, omitted unreachable material, and overlapping global symbols limit conclusions about executable fidelity. Normal return is not established.

## Interface

### Explicit inputs

The displayed interface has two 32-bit inputs.

| Input | Established use |
|---|---|
| First input | No explicit use appears in the displayed body. |
| Second input | Passed to the first indirect invocation and later stored intact in a dependency-selected word location. |

Their signedness, ownership interpretation, and valid-value ranges are unspecified. Apparent nonuse of the first input does not establish its irrelevance to the underlying executable, whose complete machine-state interface is unavailable.

### Implicit inputs

Behavior also depends on:

- Incoming machine state determining the initial indirect target.
- An incoming unsigned 32-bit quantity used in early selector-byte addresses and a separate call argument.
- Primary and secondary dependency results.
- Memory reached through those results and arithmetic transformations.
- The byte at absolute address `0x01`.
- The shared word at absolute address `0x28`.
- A later shared quantity whose declaration and absolute address are unavailable.

The word at `0x28` has three separate represented reads in one phase. The unlocated shared quantity has two separate represented reads in a later phase. Neither group may be treated as a single captured observation.

### Outputs, errors, and termination

No explicit return value, successful status, or completion flag is established. Observable outputs consist of direct memory effects, dependency interactions, and any effects produced by dependencies.

The displayed endpoint follows additional indirect invocations and is marked as invalid instruction data. Whether execution traps, continues elsewhere, or performs another terminal transfer is `documented_only`.

No validation failure, exception translation, rollback, or recovery path is visible. Every later effect is conditional on earlier accesses and dependencies permitting continuation.

### Preconditions

Execution of a reached action requires its target to be callable or its memory access to be permitted, as applicable. Continuing through the sequence requires reached dependencies to return sufficiently for subsequent actions.

The representation accesses low absolute addresses and potentially unaligned derived addresses. It establishes no bounds, alignment, non-nullness, access-right, or lifetime checks. An environment prohibiting those accesses cannot perform the displayed sequence unchanged.

## Behavior

### Widths, arithmetic, and memory layout

The representation distinguishes 8-bit bytes, 16-bit halfwords, 32-bit words, and 64-bit composite results. Composite results are separated into low and high 32-bit halves; this numerical division does not establish memory byte order.

Unsigned right shifts are logical. Unsigned 32-bit arithmetic retains the low 32 bits. Byte and halfword publications retain the low eight and sixteen bits, respectively.

Some expressions instead pass through signed 32-bit intermediates. Signed overflow, signed left shifts, and conversion of calculated integers into addresses do not establish portable arithmetic behavior. Their executable interpretation remains `documented_only`; universal wrapping or unbounded arithmetic must not be assumed.

Principal access forms include:

| Address form | Established access |
|---|---|
| `0x14` plus four times a selector | Word reads and writes. |
| `0x14` plus sixteen times a selector | Word writes, distinct from four-times scaling. |
| Captured unsigned word shifted right by 22, plus `0x14` | A word read, without additional four-times scaling. |
| A transformed quantity plus one | Selector-byte read in multiple phases. |
| Zero-extended selector byte plus `0x11` | Byte store at an absolute address from `0x11` through `0x110`. |
| Derived base plus `0x04`, `0x08`, or `0x0c` | Word reads. |
| Derived base plus `0x1c` | A byte store in one phase; separate word reads in another. |
| Derived base plus `0x34` | Word stores, including consecutive pairs. |

A selector’s value determines the byte-store destination; the destination is not generally adjacent to the selector’s storage.

### Sequencing rules

The displayed body has no ordinary conditional branch, loop, validation exit, or early-success path. It presents a straight-line sequence, subject to the control-flow uncertainties described above.

Separate statements establish relative ordering. Multiple argument expressions within one invocation do not establish a relative evaluation order among their reads. Each argument read supplies its specified argument position, but no left-to-right or right-to-left chronology is promised.

A captured value remains distinct from a later direct read of its original location. Invocations with no displayed arguments or unused results remain consequential: their memory effects, failure behavior, and ability to return are unknown.

These are access-level obligations of the representation, not guarantees of volatile qualification, atomicity, or a particular physical bus trace.

### Initial input publication and word extraction

The first indirect invocation receives the second explicit input. After further invocations, a returned selector determines a word destination at `0x14` plus four times that selector. The complete second input is stored there.

A later returned selector determines two distinct locations:

- Four times the selector is stored as a word at `0x14` plus sixteen times the selector.
- A word is then captured from `0x14` plus four times the selector.

Those locations can alias for particular values, but are not generally interchangeable.

After additional invocations, the captured unsigned word shifted right by 11 is published into another selected word location. A separate word is then read at the captured word shifted right by 22, plus `0x14`.

That right-by-22 operation forms a **word-read address**, not a selector-byte address. No additional multiplication by four appears in that address.

### Early selector-byte updates

An early dependency-produced unsigned quantity supplies two byte-store values:

- Its left shift by two, narrowed to a byte.
- Its low 29 bits shifted right by 19, narrowed to a byte.

For both updates, the selector-byte address is the incoming implicit unsigned quantity shifted right by 11, plus one. Intervening dependencies separate the reads; the second update reads its selector again.

The incoming implicit quantity also supplies a separate right-by-22 call argument. That argument is not a selector-byte address.

### Repeated byte-mediated propagation

Much of the remaining sequence alternates dependency chains with selector-byte reads and byte stores. Store values include zero, low bytes of scaled results, and masked or shifted fields.

Selector-address transformations include scaling by 2, 4, 8, 16, 64, 1024, and 4096; right shifts by 9, 11, 12, and 21; and further masked or larger power-of-two transformations. Mask placement relative to scaling and shifting matters.

Right-by-22 shifts are not included in this selector-address classification. The displayed address-forming right-by-22 shift selects a 32-bit word at the shifted value plus `0x14`. Other displayed right-by-22 shifts contribute byte-store values or call arguments.

Several intermediates are explicitly zeroed, causing selector reads at absolute address `0x01`. Their resulting stores and intervening dependency interactions remain part of the represented sequence; zero arithmetic does not remove them.

F-23325cd5e50c receives a captured 32-bit dependency result. The retained result later contributes scaling, masking, and subtraction from a zero-extended selector byte. Where both subtraction operands are unsigned 32-bit quantities, subtraction wraps modulo \(2^{32}\).

A later dependency result is stored as a word at absolute address `0x7c` before being supplied twice to another invocation. Those arguments reuse the capture; they are not rereads of `0x7c`. Subsequent dependencies may alter that storage.

### Retained word and three observations at `0x28`

A word read from a secondary-result-derived address plus `0x14` is retained across intervening calls and byte updates. It later supplies a left shift by 11 for publication into a selected word location.

Separately, the shared word at `0x28` participates in three reads:

1. An initial read captures the value used by a later store.
2. A fresh read supplies the first argument of a subsequent invocation.
3. Another fresh read supplies its second argument.

The capture precedes both argument reads. The relative order of the argument reads is unspecified.

After that invocation and a further call, the initial capture is written to `0x14` plus four times a secondary-result selector. Neither argument is represented as reuse of that capture. All three observations may differ where the environment permits changing storage.

### Structured reads and composite-result propagation

One composite result supplies a high-half-derived base. A word at that base plus `0x08` is captured first, followed by a word at base plus `0x0c`, before the next invocation.

The captured `0x0c` value later determines a selector-byte address. The captured `0x08` value supplies the associated byte-store value. A separate word read at a retained low-half-derived base plus `0x04` later supplies another byte-store value.

The mapped dependencies occur in this relative order, with other interactions between them:

1. F-23325cd5e50c.
2. F-bbb54b5f036d.
3. F-f80b3ac065fc.
4. F-de15789688a8.
5. F-ea83f9a17404.
6. F-674e75a953e8.

F-f80b3ac065fc receives a composite result’s low half, its high half, and two further copies of the low half. The other mapped dependencies after the first receive a captured quantity repeated in two argument positions.

Additional invocations combine composite-result halves, shifted fields, repeated values, and secondary results. Repetition does not establish ownership, reference counting, or preservation of reachable memory.

### Separate byte argument reads and zero-address stores

Before the zero-address word publication, one invocation obtains its two arguments through separate direct reads of the byte at `0x01`. Each supplies a value from 0 through 255. They do not reuse an earlier selector, and their relative evaluation order is unspecified.

Further calls and byte-mediated updates intervene before the following stores:

- An explicitly zeroed unsigned intermediate determines a word destination by multiplication by eight. A dependency result is stored at absolute address zero.
- After more calls and byte accesses, masking that same zero intermediate and shifting right by 11 determines a halfword destination, also zero. A later result is narrowed to sixteen bits and stored there.

If both stores complete, the halfword store overwrites byte addresses `0x00` and `0x01`, overlapping the earlier word store covering `0x00` through `0x03`. Byte order determines the significance and encoding of those bytes, not which addresses overlap. Intervening dependencies may also modify them.

A subsequent byte store has a different destination: an earlier composite result’s low half, masked to 30 bits and shifted right by seven, with an additional zero contribution. Its value comes from a later composite result. It is not necessarily a byte-mediated destination.

### Repeated field publications

A later phase writes a captured selector-byte value as a 32-bit word at a derived address plus `0x34`, then immediately writes that value shifted left by one to the same location.

Another phase writes the same call-derived value shifted left by two twice to a secondary-result-derived address plus `0x34`.

No dependency call separates either pair. Both represented stores matter; final contents alone do not describe the access sequence. The second pair includes signed-shift uncertainty where the represented intermediate is signed.

### Two reads of the unlocated shared quantity

After publishing a dependency-produced quantity into a selected word location, the operation reads the unlocated shared quantity in two successive statements.

The first observation is captured as unsigned 32-bit data. Later, its low 30 bits shifted right by ten supply an invocation argument.

The second observation supplies multiplication by four. That result is retained separately and used for selector reads at the scaled quantity plus one. Subsequent transformations of this second-read-derived quantity contribute further addresses.

The first read precedes the second. They are not required to agree. The shared quantity’s declaration, absolute address, and access mechanism remain `documented_only`.

### Paired word reads at offset `0x1c`

Later, the retained quantity derived from the second shared read undergoes a left shift by 26. An invocation obtains both arguments through separate 32-bit reads at the transformed address plus `0x1c`.

These are independent read expressions from the same derived location, not two copies of one capture. Their relative evaluation order is unspecified, and differing observations are permitted where storage can change.

A further multiplication by 1024 contributes to subsequent selector addresses. Signed intermediates prevent unconditional machine-width simplification of this chain without an explicit arithmetic interpretation.

### Late slot publications and final propagation

Late phases publish values, secondary results, shifted captures, and zero into selected word locations. One unsigned phase stores a composite result’s low half multiplied by \(2^{26}\) at `0x14` plus that low half multiplied by \(2^{28}\), with unsigned 32-bit arithmetic.

Another returned selector causes four times the selector to be stored at `0x14` plus sixteen times the selector. The following invocation receives:

- A fresh unsigned word read at `0x14` plus four times the selector, shifted right by 12.
- Four times the selector.
- A separate unsigned word read from that same location, shifted left by two.

The first and third arguments do not share a capture. Their read order is unspecified.

Further phases continue distinguishing sixteen-times-scaled publication destinations from four-times-scaled slots. In a different phase, a word is captured once from a four-times-scaled slot and later published after a right shift by 12.

The final byte-mediated update narrows a call-derived unsigned quantity masked to 30 bits and shifted right by 20. Further invocations precede the invalid-instruction endpoint.

## Side effects

Direct effects include arithmetic-derived byte and word accesses, publication at `0x7c`, three reads at `0x28`, repeated reads at `0x01`, overlapping mixed-width stores at zero, and repeated stores at offset `0x34`.

No rollback or transaction boundary is established.

No allocation, release, ownership transfer, reference-count obligation, or lifetime boundary is established by the caller’s direct actions. An address-like result does not prove ownership. Unknown dependencies may allocate, release, mutate, or invalidate memory, including globals and previously accessed storage.

No synchronization, atomicity, reentrancy, or thread-safety guarantee is established. Access width does not prove atomicity, and repeated read expressions do not prove volatile qualification.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-23325cd5e50c | Receives one captured 32-bit result. |
| F-bbb54b5f036d | Receives a captured 32-bit quantity twice. |
| F-f80b3ac065fc | Receives low and high composite-result halves, followed by two more copies of the low half. |
| F-de15789688a8 | Receives a captured 32-bit quantity twice. |
| F-ea83f9a17404 | Receives a captured 32-bit quantity twice. |
| F-674e75a953e8 | Receives a captured 32-bit quantity twice late in the sequence. |

Their semantics, error behavior, state preservation, memory effects, and ownership conventions are unknown.

Numerous other indirect and direct dependencies have no supplied opaque IDs. Their identities and complete interfaces remain `documented_only`; they must not be conflated with mapped dependencies.

## Acceptance examples

These examples are conditional static obligations, not executed tests. Each assumes the relevant actions are reached and permitted.

### Full-width input publication

A second explicit input of `0x89abcdef` is published as that complete 32-bit pattern into the initial selected slot. Later retention is not promised.

### Word-address shift versus selector addressing

If the early captured unsigned word is `0x01000000`, shifting it right by 22 gives four. The associated word-read address is therefore `0x18`, not `0x24`.

This calculation selects a 32-bit word. It does not itself read a selector byte or choose a byte-store destination.

### Byte-selected destination and narrowing

A selector byte of `0xff` selects absolute destination `0x110`. A byte-store value of `0x123` publishes `0x23`.

### Three independent shared-word observations

If the capture at `0x28` yields `0x11111111`, while the reads supplying the first and second arguments yield `0x22222222` and `0x33333333`, the invocation receives the latter two values. The later selected-slot store uses `0x11111111`.

Values here are assigned by argument position, not chronological read order.

### Distinct late shared reads

If the first unlocated shared observation is `0x00000400` and the second is three, the first capture’s low-30-bit, right-by-ten argument is one. The second observation’s multiplication by four gives twelve, making the initial associated selector-byte address `0x0d`.

### Independent argument reads

Two direct byte reads at `0x01` may supply four and nine to their respective argument positions. Reusing one capture would not preserve that behavior.

Likewise, the paired word reads at derived offset `0x1c` remain independent.

For the late selected-slot invocation, a first-argument read of `0x00012000` produces `0x12` after shifting right by 12. A separate third-argument read of three produces `0x0c` after shifting left by two.

### Repeated writes and overlap

A captured selector byte of seven produces consecutive word publications of seven and fourteen in the doubled-publication phase.

An unsigned quantity of `0xffffffff`, masked to 30 bits and shifted right by 20, produces `0x3ff`; narrowing publishes byte `0xff`.

If both zero-address stores complete, the later halfword store overwrites addresses zero and one, but does not directly write addresses two and three. No successful completion status is promised if a reached low-address access fails.

## Uncertainties

The following remain `documented_only`:

- Application purpose, encoding, and object layout.
- Complete calling convention and secondary-result channels.
- Dependency semantics and indirect-target validity.
- Executable control-flow fidelity and terminal transfer.
- Signed arithmetic, address conversion, alignment, and byte order.
- Declaration and address of the unlocated shared quantity.
- Volatile qualification, concurrent mutation, and physical access realization.
- Memory-map meaning, aliasing, extents, ownership, and lifetimes.
- Error propagation, synchronization, atomicity, and reentrancy.

Repeated masks suggest packed fields; byte-selected destinations suggest indirection. Neither inference resolves these unknowns.

## Evidence

Established static evidence comprises the displayed input widths, access widths, arithmetic expressions, statement ordering, distinct argument-read expressions, captured-value reuse, mapped dependency interactions, repeated stores, and terminal marker.

In particular, the address-forming right-by-22 expression accesses a word at the shifted quantity plus `0x14`; the selector-byte address transformations are separate. Other right-by-22 uses contribute stored byte values or invocation arguments.

The supplied representation reports invalid instruction data, omitted unreachable material, and overlapping global symbols. These reports limit conclusions about executable behavior and storage identity.

No runtime tests, execution traces, dependency implementations, independent instruction validation, or memory-map verification have been performed. Acceptance examples state conditional consequences of static evidence, not observed execution.
