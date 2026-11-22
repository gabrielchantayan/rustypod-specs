<!-- Trusted provenance: F-cbbec9cf3027; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cbbec9cf3027

## Purpose

F-cbbec9cf3027 performs an ordered sequence of dependency calls, byte-directed writes, word-field updates, and mixed-width stores. The supplied static representation does not establish an application-level purpose.

The contract is therefore **partial and conditional**: it identifies represented operations, data dependencies, access widths, and sequencing without assigning semantic types to encoded values or assuming that unresolved calls are harmless.

**Established static evidence:** four explicit 32-bit arguments, additional incoming machine state, numerous dependency calls, shared-memory accesses, and termination of the represented path at an invalid-instruction boundary.

**Inferred intent:** repeated scaling, shifts, selector bytes, and field links suggest an encoded state transformation. This interpretation is insufficient to define the function as a particular protocol, allocator, container operation, or dispatcher.

**Unknown:** the intended operation, complete calling convention, valid execution environment, and behavior beyond the represented boundary.

## Interface

### Explicit arguments and result

| Position | Represented width | Established use |
|---|---:|---|
| First | 32 bits | Passed to the first computed dependency call. |
| Second | 32 bits | No explicit use in the supplied body. |
| Third | 32 bits | No explicit use in the supplied body. |
| Fourth | Signed 32 bits | Shifted left by three for an early call’s fourth argument. |

No return value is represented. This does not establish normal return: the visible path ends at invalid instruction data.

The second and third arguments’ apparent nonuse is limited to the supplied representation. Incomplete machine-state and dependency information prevents a stronger claim about the executable interface.

### Additional incoming state

Execution also consumes:

- A continuation-related machine value used to derive the first call target.
- An incoming address used for the initial selector-byte reads.
- An incoming unsigned word supplying the initial byte payloads.
- Secondary results exposed after calls, whose complete result conventions are unavailable.

The four explicit arguments alone do not determine behavior.

Several dependencies have represented 64-bit results that are separated into lower and upper 32-bit components. This establishes component extraction in the representation, not that the actual interface returns a scalar 64-bit integer.

### Memory layout and access widths

No single input object or output buffer is established.

| Access | Established forms |
|---|---|
| 8-bit read | A derived address plus `0x01`; also the byte global at `0x01`. |
| 8-bit routed write | An unsigned selector byte plus `0x11`. |
| Other 8-bit write | A paired result’s upper component plus `0x1c`; a masked-and-shifted derived address. |
| 16-bit write | A zero-derived address, represented as address zero. |
| 32-bit field access | Scaled indices plus `0x14`, derived bases plus `0x34`, and result-relative offsets. |
| Fixed 32-bit state | Reads at `0x28`, publication at `0x7c`, and a store at address zero. |

Result-relative word accesses include offsets `0x04`, `0x08`, `0x0c`, and `0x1c`. Repeated offsets do not establish a common structure type.

Byte order, required alignment, memory mapping, and access atomicity are unknown.

### Preconditions and ownership

Continuation requires every reached call target to be executable and every reached memory access to be supported. This includes low addresses and address zero.

No local null, bounds, allocation, or address-validity checks are visible. Dependencies may perform checks, but none are established here.

No allocation, release, ownership transfer, retention, or lifetime guarantee is identified. A dependency may change ownership or invalidate memory; later accesses are conditional on their continuing validity.

## Behavior

### Sequencing and observation rules

The supplied body presents a long path without an explicit loop or conditional branch. Recovery limitations prevent treating it as a complete control-flow description.

Operations below are ordered across the represented statements, conditional on earlier calls returning and accesses completing. Calls between accesses may change shared state, selectors, destinations, or memory validity.

Three distinctions apply throughout:

1. **Captured values** remain earlier observations even when their originating memory changes.
2. **Separate memory references** are not silently replaced by a shared snapshot.
3. **Multiple references inside one call’s argument list** have no established relative evaluation order.

The representation does not establish volatility, concurrency semantics, or an exact hardware-load count for ordinary global references. The contract preserves the separate source-facing observations without inventing machine-level guarantees.

### Initial calls and byte routing

The first computed dependency receives the first explicit argument. A second computed call follows.

Execution then reads a selector byte at the incoming address plus one and writes the incoming payload word’s low byte at the selector value plus `0x11`.

An early secondary result is captured and passed three times to another call. That call’s fourth argument is the fourth explicit argument shifted left by three.

After further calls, execution reads the initial selector location again and routes the incoming payload’s bits 22–29. This second selector observation need not equal the first.

Subsequent calls forward generated results and scaled values, including repeated arguments. No result or shared-state preservation is assumed beyond the explicit captures shown.

### First derived record

The early captured unsigned result, shifted right by ten, supplies a record address. Execution reads a 32-bit word at that address, then another at offset `0x08`.

A paired result supplies the lower and upper components to a dependency that also receives the record address plus `0x14` and another copy of the lower component.

The first record word determines a selector location by multiplication by `0x400`, followed by addition of one. The second word supplies two payloads:

- Its low byte.
- Its bits 22–29.

Calls separate the two routed writes. Each write uses its own selector observation.

### Early word-field updates

The next field transformations include, in represented order:

- Clearing a word at four times a dependency-provided index plus `0x14`.
- Storing that index into another dependency-selected field of the same address form.
- Storing four times a later index at sixteen times that index plus `0x14`.
- Reading the word at four times that later index plus `0x14`.
- After calls, storing the captured word shifted right by eleven into another selected field.
- Reading a word at the captured word shifted right by 22, plus `0x14`.

The last address has no additional multiplication by four. Treating it as an ordinary word-array index would change the represented address.

### Recurring byte-routing transformations

Later phases interleave calls with selector reads and routed writes. Selector locations use combinations of scaling, masking, and right shifts. Represented scales include four, eight, sixteen, `0x40`, `0x400`, `0x1000`, and larger powers of two. Some selectors are read at fixed address `0x01`.

Payloads include zero, low bytes of scaled results, and extracted upper bit fields.

Important narrowing rules include:

- A word masked with `0x3fffffff`, shifted right by 20, and stored as a byte contributes original bits 20–27.
- A word masked with `0x1fffffff`, shifted right by 19, and stored as a byte contributes bits 19–26.
- Masking with `0x3fffff`, shifting right by twelve, and narrowing retains only bits 12–19 of the value being masked.

Some values have already undergone earlier shifts. Later extraction applies to that transformed value, not to the original word with discarded bits restored.

Selectors captured for subsequent call arguments remain captures. Other selector expressions are evaluated afresh after calls.

### First mapped call and fixed-state publication

F-23325cd5e50c receives one generated word. That captured word remains an input to later scaling, masking, and subtraction operations. No ownership consequence follows from the call shape.

Later, a generated word is written to fixed address `0x7c` before being passed twice to an unresolved dependency. Both arguments use the captured result; they are not separate reads of `0x7c`.

Another dependency-derived address supplies a word at offset `0x14`. That captured word participates in subsequent calls, shifts, and routed writes.

### Distinct references to fixed address `0x28`

The representation distinguishes three word references at `0x28`:

- An initial read captured for a later field store.
- A separate reference supplying one argument of an intervening call.
- Another separate reference supplying that call’s other argument.

The capture precedes the argument references. The relative evaluation order of the two argument references is not established. Their identical address expressions do not make them copies of the captured value or prove equal argument values under an environment permitting changing observations.

After that call and another call, the original captured word is stored at four times a dependency-provided index plus `0x14`.

Thus the later store is tied to the first observation. Changes during either intervening dependency do not replace its payload with the current contents of `0x28`.

### Paired results and mapped-call order

A later paired result provides an upper component used as a base for consecutive word reads at offsets `0x08` and `0x0c`. The lower component is captured separately. A subsequent word read at that captured lower component plus `0x04` supplies another routed payload.

The mapped dependencies occur in this order:

1. F-23325cd5e50c.
2. F-bbb54b5f036d.
3. F-f80b3ac065fc.
4. F-de15789688a8.
5. F-ea83f9a17404.
6. F-674e75a953e8.

Numerous unresolved calls intervene. The mapped dependencies are not the complete set of externally significant calls.

### Byte-global arguments and mixed-width stores

One paired result causes a byte write at its upper component plus `0x1c`, using the lower component’s low byte.

Later, an unresolved call has two arguments that separately reference the byte global at address `0x01`. These are not uses of a single saved selector. Their relative evaluation order is unspecified, and their equality is not established solely by the repeated address expression.

Following further calls and routed writes, a zero-valued index produces:

1. A 32-bit store at eight times that index, hence address zero.
2. A later 16-bit store at the masked index shifted right by eleven, also address zero.

The second payload is the low sixteen bits of a dependency result. Intervening calls and writes remain significant; the two stores do not establish a final combined word independently of byte order and intervening effects.

Another paired result produces a byte store at a masked lower component shifted right by seven. The zero index contributes no additional displacement.

### Repeated fields and separate late global observations

Later updates include two consecutive word writes to a derived base plus `0x34`: first a captured selector, then twice that selector. Another location at offset `0x34` receives two consecutive identical word stores.

These remain distinct represented writes. They are not contracted into a single final-value assignment.

Additional phases link scaled indices through fields at offsets `0x14`, store dependency results, and store captured values shifted right by twelve.

A late shared word global, whose address is unavailable, is referenced separately for:

- A saved unsigned value.
- A four-times-value calculation used for addressing.

The saved observation precedes the scaling observation in the representation. There is no intervening dependency call, but this does not establish a single snapshot, atomicity, or an absence of asynchronous changes. The saved value later supplies a masked-and-shifted call argument; the scaling observation supplies selector addresses and the late address transformations.

### Late intermediate address versus zero-derived selectors

Under the conditional interpretation that the relevant operations wrap to 32 bits, the four-times-global calculation followed by a left shift of 26 can produce a **nonzero intermediate**.

That intermediate has a different role from its subsequent multiplication by `0x400`:

- The word references use the intermediate plus `0x1c`.
- Multiplication of the intermediate by `0x400` produces zero modulo \(2^{32}\).
- Subsequent selector addressing uses that zero product plus one, hence address `0x01`.

The word location is therefore not unconditionally `0x1c`.

For a scaling observation of one, the four-times value is four, the shifted intermediate is `0x10000000`, and the word address is `0x1000001c`. Only the later product becomes zero.

The call at this point contains two separate word references at the intermediate plus `0x1c`. Neither their relative evaluation order nor a shared captured value is established.

After that call, execution performs a zero-payload routed write using a selector at `0x01`. Following more calls, it reads the selector again and routes a secondary result’s bits 22–29. The zero-derived value also contributes later call arguments before F-674e75a953e8.

### Tail and termination

After F-674e75a953e8, the represented path continues with field links, routed bytes, zero-derived arguments, and a zero word store through a field address derived from a paired result’s upper component.

Further operations include:

- A scaled lower component stored at another scaled address plus `0x14`.
- A masked lower component shifted right by nine stored in a selected field.
- Repeated four-times-index links through sixteen-times-index fields.
- A call with two separate references to one word field, one shifted right by twelve and the other shifted left by two; argument-reference order and equality are not established.
- A later, genuinely captured field value shifted right by twelve for a subsequent store.
- A six-bit masked argument shifted left by fourteen, alongside other scaled arguments.
- Final routed payloads from a generated result, with calls between selector observations.

The visible path ends at invalid instruction data. Normal return, cleanup, recovery, or continuation beyond that boundary is not established.

### Arithmetic constraints

Unsigned 32-bit operations retain the low 32 bits. Unsigned right shifts are logical. Byte and halfword stores retain the low eight and sixteen bits.

Some scaling and left shifts use represented signed 32-bit intermediates. Overflow or left-shifting negative values does not establish portable arithmetic behavior. Wrapping interpretations, including the late zero-product deduction, are conditional.

For the signed fourth argument, shifting left by three has an unambiguous nonnegative, nonoverflowing range of zero through `0x0fffffff`.

No explicit saturation, overflow error, or arithmetic-recovery path is visible.

## Side effects

The function can modify shared and dependency-derived memory through 8-, 16-, and 32-bit stores.

Every unsigned byte-selected destination is between `0x11` and `0x110`, inclusive. This range includes the fixed state locations `0x28` and `0x7c`; aliasing cannot be excluded.

The explicit 32-bit store at address zero may also overlap the byte global at `0x01`, depending on the represented memory mapping. Its byte-level effect requires the execution environment’s byte order.

No transaction, rollback, locking, atomicity, or reentrancy guarantee is established. Earlier effects may remain observable if a later call does not return or an access faults.

Dependencies may mutate shared state or alter memory ownership. No preservation or release behavior is assigned without a dependency contract.

## Dependencies

| Opaque ID | Established arguments |
|---|---|
| F-23325cd5e50c | One captured generated 32-bit word. |
| F-bbb54b5f036d | Two copies of one captured generated word. |
| F-f80b3ac065fc | Lower paired component, upper paired component, and two further copies of the lower component. |
| F-de15789688a8 | Two copies of one captured generated word. |
| F-ea83f9a17404 | Two copies of one captured generated word. |
| F-674e75a953e8 | Two copies of one captured generated word. |

Their purposes, result conventions, effects, ownership behavior, and failure behavior are **documented_only**.

Additional dependencies include computed targets and two direct targets without supplied opaque mappings. Their identities remain **documented_only**; no substitute IDs or semantic names are invented.

## Acceptance examples

These are conditional static criteria, not executed tests.

1. **Fresh selector.** If the initial selector is `0x20`, the destination is `0x31`. If intervening calls change it to `0x30`, the later routed write uses `0x41`.

2. **Upper-bit payload.** An unsigned payload of `0xabc00000`, shifted right by 22 and narrowed to a byte, produces `0xaf`, not `0xab`.

3. **Mask followed by narrowing.** Masking `0x3ff00000` with `0x3fffffff`, shifting right by 20, and storing a byte produces `0xff`.

4. **Captured fixed state.** If the initial `0x28` observation is `0x12345678`, the later field store uses that value even if the intervening calls change `0x28`.

5. **Separate fixed-state arguments.** If an admissible observation model gives the two argument references at `0x28` different values, the contract preserves them separately. It does not prescribe which argument was evaluated first.

6. **Separate byte-global arguments.** The two byte-global references used as call arguments are not required to reuse an earlier selector or each other’s observation.

7. **Separate late global values.** If the saved late-global observation is two and the scaling observation is one, the saved-value calculation uses two, while the wrapping-interpreted word address is `0x1000001c`.

8. **Nonzero intermediate.** With scaling observation one, both late word argument references address `0x1000001c`; subsequent zero-derived selectors address `0x01`. The word references do not address `0x1c`.

9. **Repeated writes.** A write-observing model retains both consecutive stores to an offset-`0x34` field, including the pair with identical payloads.

10. **Mixed-width zero stores.** If address zero supports the accesses, the represented word store precedes the halfword store. No environment-independent final word value follows.

11. **Failure and termination.** An access fault does not require rollback or a returned error. Reaching the invalid-instruction boundary does not demonstrate successful return.

## Uncertainties

The following remain **documented_only**:

- Application purpose, semantic argument types, and complete ABI.
- Meaning and validity of additional incoming machine state.
- Completeness of recovered control flow.
- Dependency effects, results, errors, ownership, and nonreturn behavior.
- Whether paired components represent integers, pointers, or an aggregate.
- Address mapping, byte order, alignment, volatility, and atomicity.
- Runtime load counts and within-call argument evaluation order.
- Signed-overflow and signed-shift interpretation.
- Concurrency, lifetime, and reentrancy guarantees.
- Behavior beyond the invalid-instruction boundary.

No runtime tests have been performed.

## Evidence

The evidence is the supplied static representation and opaque dependency mapping. Prior prose and reviewer feedback are not independent behavioral evidence.

The representation establishes the described access widths, offsets, captures, separate global and field references, mapped-call order, and represented write ordering. Arithmetic examples are deductions under the explicitly stated width and wrapping conditions.

The supplied material also reports invalid instruction data, removal of an unreachable region, and overlapping global symbols. These limit whole-program conclusions.

No implementation inspection, dependency inspection, binary validation, or runtime observation was performed.
