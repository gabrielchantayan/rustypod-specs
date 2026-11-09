<!-- Trusted provenance: F-9e4db05ff58b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9e4db05ff58b

## Purpose

The supplied static representation describes an ordered sequence of byte accesses and indirect calls followed by an unconditional loop. Two explicit input words, two incoming execution-context values, and selected dependency results determine the visible addresses, stored data, and call arguments.

No application-level purpose is established. The evidence does not identify a parser, allocator, device operation, or protocol.

This contract describes the represented behavior, conditional on the static representation accurately reflecting execution. Diagnostics report invalid instruction data and removed unreachable regions. The reconstructed interface and control flow therefore remain subject to those limitations.

## Interface

### Explicit inputs

The represented interface contains four 32-bit input slots.

| Input | Representation and visible use |
|---|---|
| First word | Unsigned 32-bit value. Supplies the initial stored byte, an unsigned right-shifted dependency argument, and a repeatedly used selector-read address. |
| Second word | 32-bit value with no explicit use shown. Signedness and semantic type are not established. |
| Third word | 32-bit value with no explicit use shown. Signedness and semantic type are not established. |
| Fourth word | Unsigned 32-bit value. Its unsigned right shift supplies both the initial destination-address component and the first dependency argument. |

No explicit use of the second and third words does not establish their irrelevance under the actual calling convention. Hidden argument use by dependencies is unresolved.

### Incoming execution context

Two additional values are consumed without visible initialization:

- A base value used to locate a selector byte one byte beyond that base.
- A control-context value used to construct the first indirect-call target by adding `0xFEC`.

These are effective hidden inputs. Their origins, widths under the actual ABI, validity, and relationships to the explicit inputs are not established independently.

### Output and completion

No return value or normal return path is represented.

If all accesses succeed and all dependencies return normally, execution continues indefinitely through the loop. A fault, exception, nonreturning dependency, or other external control transfer may interrupt that progression. No local recovery path is shown.

### Preconditions

For an execution prefix to complete:

- Every reached read address must permit a one-byte read.
- Every reached write address must permit a one-byte write.
- Each reached indirect target must be executable under the actual ABI.
- Dependencies must return for execution to advance beyond their calls.
- Address calculations and shifts must have an applicable execution-platform interpretation.

The function does not visibly check these conditions. No object size, buffer length, ownership token, or address-range validation is supplied.

## Behavior

### Widths and address interpretation

All directly represented memory accesses are one byte wide.

A selector read produces an unsigned value from 0 through 255. Adding `0x11` selects a zero-write destination from `0x11` through `0x110`, inclusive. The representation supplies no additional object base for these destinations. They must not be reinterpreted as offsets into an assumed array or structure.

Unsigned operations on the first and fourth inputs use 32-bit values. Results retained from F-D07, F-D14, F-D19, and F-D24 are represented as signed integer values; their later left shifts require particular care. F-D12’s result is retained as an unsigned 32-bit value.

The formulas below distinguish established unsigned operations from conditional 32-bit interpretations of signed shifts.

### Entry memory effects

Before any dependency call, the following events occur in order:

1. The first input’s low eight bits are written to the address obtained by unsigned-shifting the fourth input right by seven and adding `0x1C`.
2. A selector byte is read from the incoming hidden base plus one.
3. Zero is written to the address equal to that selector value plus `0x11`.

The initial destination ranges from `0x1C` through `0x0200001B`. Its calculation does not overflow an unsigned 32-bit word.

The selector is read **after** the initial data write. Its entry-time contents are not necessarily the contents observed by that read. If the initial destination equals the hidden-base-plus-one location, the initial write replaces the selector before it is consumed.

The two writes may also target the same byte. In that case, the later write replaces the initial byte with zero. These are ordered write events, not guarantees about final memory contents.

### Initial call sequence

After the entry memory effects, dependencies are called in this order:

| Order | Dependency | Explicit argument or retained result |
|---|---|---|
| 1 | F-D01 | Receives the fourth input unsigned-shifted right by seven. |
| 2 | F-D02 | No explicit argument shown. |
| 3 | F-D03 | Receives the first input unsigned-shifted right by eight. |
| 4 | F-D04 | No explicit argument shown. |
| 5 | F-D05 | No explicit argument shown. |
| 6 | F-D06 | No explicit argument shown. |
| 7 | F-D07 | Its result is retained for F-D08. |

Immediately after F-D07 returns, the function rereads the selector at the hidden base plus one, then writes zero to the newly observed selector value plus `0x11`.

This is a fresh read, not reuse of the first selector. The initial writes and any preceding dependency effects can change the byte observed. The first and second hidden-base-selected destinations need not match.

Execution then continues:

| Order | Dependency | Explicit argument or retained result |
|---|---|---|
| 8 | F-D08 | Receives the F-D07 result shifted left by two. |
| 9 | F-D09 | No explicit argument shown. |
| 10 | F-D10 | No explicit argument shown. |
| 11 | F-D11 | No explicit argument shown. |
| 12 | F-D12 | Its result is retained as an unsigned 32-bit word. |

The F-D07 shift is represented as a signed left shift. Portable behavior for negative operands or overflowing positive operands is not established.

### Five-bit extraction and third selector-based write

From the F-D12 result, the function retains bits 25 through 29, inclusive. The extracted value is between 0 and 31. Bits 30 and 31 do not contribute.

Next, it reads a selector byte using an address derived from the original first input:

\[
A=\bigl((\text{first input}\times 1024)\bmod 2^{32}\bigr)+1.
\]

It then writes zero to that selector value plus `0x11`.

The multiplication uses unsigned 32-bit arithmetic. The product is a multiple of 1024, so adding one cannot overflow. The possible selector-read addresses range from `0x00000001` through `0xFFFFFC01`.

Only the first input’s low 22 bits affect this address. Inputs differing by a multiple of \(2^{22}\) select the same address, although their other effects can differ.

After this zero write:

1. F-D13 receives twice the extracted five-bit value, an unsigned argument from 0 through 62.
2. F-D14 is called, and its result becomes the first argument of the first loop-entry call.

### Loop behavior

Each completed iteration contains ten dependency calls and one selector read followed by one zero-byte write.

The entry dependency changes after the first iteration:

- The first iteration calls F-D15.
- Every later iteration calls F-D25.

The entry call has two explicitly represented arguments.

For the first iteration:

- The first argument is the F-D14 result.
- The second argument is the extracted five-bit value shifted right by eight, which is necessarily zero.

After the entry call, each iteration performs this sequence:

1. Call F-D16, F-D17, and F-D18, in that order.
2. Call F-D19 and retain its result.
3. Read a fresh selector byte at address \(A\).
4. Write zero to that selector value plus `0x11`.
5. Call F-D20 with the F-D19 result shifted left by two.
6. Call F-D21, F-D22, and F-D23, in that order.
7. Call F-D24 and retain its result.
8. Derive the next iteration’s two entry arguments from that result.

No explicit arguments are shown for F-D16 through F-D19 or F-D21 through F-D24.

The F-D19 shift passed to F-D20 is represented as a signed left shift, with the same unresolved negative-value and overflow behavior as the F-D07 shift.

For the next iteration, the F-D24 result is shifted left by two to form the first argument. Separately, that signed result is shifted left by 26, assigned to an unsigned 32-bit quantity, and subsequently shifted right by eight to form the second argument.

Assignment to an unsigned destination does not, by itself, resolve the preceding signed-shift semantics.

Under a **conditional 32-bit wrapping bit-vector interpretation**, an F-D24 result with bit pattern \(r\) produces:

\[
\text{next first argument}=(r\times4)\bmod 2^{32},
\]

\[
\text{next second argument}=(r\mathbin{\&}63)\times2^{18}.
\]

The second argument then ranges from zero through `0x00FC0000`, in increments of `0x00040000`. These formulas are not an unconditional claim about signed overflow or negative signed operands.

There is no represented exit condition.

## Side effects

### Direct memory effects

Before loop entry, the direct effects are:

- One byte write containing the first input’s low eight bits.
- Two reads from the hidden-base selector location, each followed by a zero-byte write.
- One read from the first-input-derived selector location, followed by a zero-byte write.

Each completed loop iteration adds:

- One read from the first-input-derived selector location.
- One zero-byte write selected by that read.

These counts exclude dependency effects and apply only to execution that reaches the corresponding events.

### Aliasing and observable order

The initial data write can overlap a selector location or a later zero-write destination. A zero write can also overlap a selector location.

Consequently:

- An entry-time selector value does not guarantee the first observed selector value.
- A selector-based write may change the selector used by a later read.
- Dependencies may change selectors or other shared state before subsequent accesses.
- A byte written earlier need not retain that value later.

Each selector read precedes its associated zero write. Selector values are not visibly cached across separate write events.

No synchronization, transaction, rollback, or cross-call atomicity guarantee is represented. Concurrent modification of selector memory is not specified.

### Ownership and resources

The body shows no explicit allocation, release, retention, or transfer of an identified memory object. It forms numeric addresses and accesses bytes directly.

This does not establish the absence of resource activity in dependencies. Their memory ownership, lifetime requirements, retention behavior, and release behavior remain unknown.

### Errors and interrupted execution

No error status, bounds check, null check, or recovery branch is represented.

A faulting access or unsuccessful control transfer may stop execution after earlier effects have occurred. No restoration of prior writes is shown. Dependency-specific failure behavior is not established.

## Dependencies

The following opaque IDs identify call sites and target roles within this contract. They do not establish external function identities or independent dependency contracts.

| ID | Role |
|---|---|
| F-D01 | Initial indirect target formed from incoming control context plus `0xFEC`. |
| F-D02–F-D06 | Remaining calls before F-D07. |
| F-D07 | Supplies the value later shifted for F-D08. |
| F-D08 | Consumes the shifted F-D07 result. |
| F-D09–F-D11 | Calls preceding F-D12. |
| F-D12 | Supplies the unsigned five-bit extraction source. |
| F-D13 | Receives twice the extracted value. |
| F-D14 | Supplies the first loop iteration’s first argument. |
| F-D15 | Entry target for the first loop iteration. |
| F-D16–F-D18 | Calls preceding F-D19 in each iteration. |
| F-D19 | Supplies the value later shifted for F-D20. |
| F-D20 | Consumes the shifted F-D19 result. |
| F-D21–F-D23 | Calls preceding F-D24 in each iteration. |
| F-D24 | Supplies the next iteration’s argument state. |
| F-D25 | Entry target for every iteration after the first. |

Except for F-D01, targets are formed from literal-derived values rather than an input-indexed table. Their executable validity and actual identities remain unverified. F-D01’s target may also coincide with another target for some incoming context; distinct labels do not prove distinct external functions.

“No explicit argument shown” does not establish a true zero-argument interface. Dependencies may consume hidden register or shared-state inputs.

Nothing here guarantees that dependencies preserve globals, preserve memory, manage ownership in a particular way, or return normally.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Each assumes successful accesses and returning dependencies through the stated observation. Selector examples also require the stated value at the read, without an intervening external modification.

### Initial byte truncation

For first input `0x123456AB` and fourth input zero, the initial write stores byte `0xAB` at address `0x1C`.

This specifies the write event, not the byte’s eventual value.

### Fourth-input granularity

Fourth inputs zero and 127 both produce initial destination `0x1C` and explicit F-D01 argument zero.

Fourth input 128 produces initial destination `0x1D` and F-D01 argument one.

### Repeated selector reads

Suppose the hidden-base selector contains three **when the first selector read occurs**, after the initial data write. The first selector-based zero write targets `0x14`.

If the same selector contains nine when F-D07 returns, the next selector read observes nine and the second hidden-base-selected zero write targets `0x1A`.

An entry-time value of three alone is insufficient for the first conclusion. Likewise, an earlier write of nine is insufficient for the second conclusion if another effect changes the selector before F-D07 returns.

### Initial write changes the first selector

For first input four, fourth input zero, and hidden base `0x1B`:

- The initial data destination is `0x1C`.
- The hidden-base selector location is also `0x1C`.
- The initial write stores four there.
- The following selector read observes four.
- The first selector-based zero write therefore targets `0x15`.

This holds even if the selector contained three on entry.

### Five-bit extraction

If F-D12 returns `0xFFFFFFFF`, the extracted value is 31 and F-D13 receives 62.

If F-D12 returns `0xC0000000`, the extracted value is zero because neither set bit lies in positions 25 through 29.

In either case, F-D15’s second explicit argument is zero.

### Unsigned address wrap

For first input `0x00400000`, multiplication by 1024 wraps to zero under unsigned 32-bit arithmetic, so address \(A\) is one.

This does not make the invocation equivalent to first input zero: the explicit F-D03 argument differs.

### Loop target and argument transition

If F-D14 returns seven, the first loop-entry call is F-D15 with arguments seven and zero.

If the first iteration’s F-D24 returns one, the next entry call is F-D25 with arguments four and `0x00040000`. Both relevant left shifts are representable for this example, avoiding the signed-overflow uncertainty.

### Partial effects before failure

If the initial data write succeeds but the following hidden-base selector read faults, the initial write has already occurred. No dependency call or selector-based zero write has yet occurred. No rollback is represented.

## Uncertainties

The following remain **documented_only**:

- Reliability of reconstructed control flow given the invalid-instruction and unreachable-region diagnostics.
- Actual ABI, hidden arguments, register preservation, and executable target interpretation.
- Origins, widths, and validity of the incoming context values.
- Machine-level interpretation of address calculations involving the hidden context.
- Whether the low addresses denote ordinary memory, mapped devices, or reconstruction artifacts.
- Dependency identities, side effects, ownership behavior, and failure modes.
- Exact behavior of negative or overflowing signed left shifts.
- Effects of concurrent changes to selector memory.
- Whether external control transfers or dependency behavior terminate execution.
- Application-level meaning of the stored bytes, extracted bits, and recurring call sequence.

**Inferred intent:** the selector-based zero writes can be described as clearing selected bytes. No broader purpose is established.

## Evidence

Established static evidence includes the byte-width accesses, ordered entry write and selector read, repeated fresh selector reads, unsigned input arithmetic, five-bit extraction, dependency ordering, first-to-later-iteration target change, and loop without a visible normal exit.

The aliasing examples follow from that ordering: the initial write precedes the first selector read and can replace its value.

The wrapping formulas for signed dependency-result shifts are conditional interpretations, not established portable semantics. Dependency behavior and application intent remain unresolved.

This contract is based solely on the supplied static representation and diagnostics. No runtime tests have been performed.
