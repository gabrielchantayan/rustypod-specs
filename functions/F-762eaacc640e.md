<!-- Trusted provenance: F-762eaacc640e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-762eaacc640e

## Purpose

The supplied static representation describes a procedure that executes an ordered sequence of indirect calls, transforms selected results using shifts and masks, performs thirteen coprocessor-directed operations, and attempts a 16-bit memory write before an invalid-instruction termination marker.

No higher-level purpose is established. The evidence does not identify a protocol, object model, device, initialization task, or useful completed result.

This contract is **documented_only**. It distinguishes:

- **Established static evidence:** the represented interface, operation order, argument relationships, and final store.
- **Conditional arithmetic deductions:** consequences if the represented operations have 32-bit truncating machine semantics, with the indicated signed or unsigned right shifts.
- **Unknown behavior:** dependency effects, instruction validity, calling conventions, architectural exceptions, and memory mappings.

The supplied representation reports invalid instruction data and truncated control flow. Consequently, the presence of an operation does not establish that an executable path reaches it. No runtime tests have been performed.

## Interface

### Explicit inputs

The represented interface has three arguments and no declared return value.

| Input | Represented width | Visible use |
|---|---:|---|
| First | Signed 32-bit quantity | Passed unchanged to the first indirect call and shifted left by 26 bits to form that call’s fourth argument. |
| Second | 32 uninterpreted bits | Passed unchanged as the first indirect call’s second argument. |
| Third | 32 uninterpreted bits | Passed unchanged as the first indirect call’s third argument. |

The first indirect call receives, in order, the first input, second input, third input, and the first input shifted left by 26 bits.

Under 32-bit truncating shift semantics, the fourth argument contains the first input’s low six bits in positions 26–31; all other bits are zero. The represented signed type does not establish a signed application-level meaning.

No visible input validation, null check, length check, range check, or error-result check precedes this call.

### Implicit inputs and execution preconditions

The explicit arguments do not determine execution. Additional dependencies include:

- An incoming link-register value used to calculate the first indirect target.
- Incoming coprocessor-related operands used throughout the body.
- Two incompletely identified secondary register values passed to later calls.
- Executable mappings, instruction-set state, and a calling convention compatible with the indirect targets.
- Memory, globals, devices, and other state accessed by dependencies.

Reaching a later operation requires every preceding call and coprocessor-directed operation to complete and return control along the represented path. No visible mechanism makes those conditions hold.

Most indirect targets are calculated from code-location-derived bases and fixed offsets. These expressions do not establish valid entry points, target alignment, or stable dependency identities.

### Results, widths, and layout

There is no declared output value and no visible normal return.

Intermediate results are represented as either 32-bit words or 64-bit quantities. For a 64-bit result:

- The low word is bits 0–31.
- The high word is bits 32–63.
- Logical right shifts used to extract these words do not imply any in-memory byte order.

The visible endpoint is an attempted store of the 16-bit value `0x00CD`, followed by an invalid-instruction termination marker. Neither successful storage nor reaching the marker is guaranteed.

No input-buffer layout, allocated object, ownership transfer, or release obligation is established. Unknown dependencies may acquire, retain, modify, or release resources; the caller’s visible dataflow does not establish their ownership contracts.

## Behavior

### Ordering and interpretation

The represented body is straight-line: no conditional branch, loop, retry, or explicit error handling appears.

All calls remain behaviorally relevant even when their results are unused. Their effects cannot be removed from the contract merely because no return value is consumed.

The following phases preserve the significant ordering and result relationships. They group operations for readability; intervening calls still occur in their represented order. “No displayed arguments” does not prove that a dependency consumes no implicit register or machine state.

Arithmetic simplifications below are conditional on fixed-width machine semantics. They are not claims that all represented signed expressions have portable language-level behavior or that the instruction decoding is valid.

### Initial forwarding and split-result transformations

The procedure first performs the four-argument forwarding call described in the interface.

The next indirect call produces a 32-bit word used by the first coprocessor-directed operation. Four further indirect calls follow; their results are not visibly consumed. Another indirect call then produces a 64-bit result:

1. Its low word supplies the second coprocessor-directed operation.
2. Its low word and its high word shifted left by 20 bits become the two displayed arguments of the next call.
3. That call produces another 64-bit result.
4. The next call receives its low word, its high word, and its high word shifted left by 12 bits.

Three more calls occur without visibly consumed results. A following call produces a 32-bit word. The procedure retains that word shifted left by 11 bits and passes the original word to **F-0e02e4c13579**. No result from that identified dependency is visibly consumed.

### Initial zero state and subsequent result forwarding

A represented unsigned 32-bit working value is explicitly set to zero. A later derivation shifts this value right by two bits and then left by 13 bits. That derivation is zero under the displayed dataflow.

After a call receiving explicit zero and four further calls without displayed arguments, another call produces a 64-bit result. The next call receives:

- Its low word shifted left by four bits.
- Its unchanged high word.
- The previously retained left-shift-by-11 word shifted left by another 17 bits.

Under 32-bit truncation, the third argument is equivalent to the original retained source word shifted left by 28 bits. Only that source word’s low four bits can survive.

The returned 64-bit result is then passed onward as low word, high word, and the zero-derived value. The resulting 32-bit word is forwarded to another call. After another call without displayed arguments, a two-zero-argument call produces a word that is also forwarded.

The next four-argument call receives:

1. The retained left-shift-by-11 word shifted left by another 31 bits.
2. An incompletely identified secondary register value.
3. The zero-derived value shifted left by 26 bits.
4. The zero-derived value shifted left by two bits.

Under 32-bit truncation, the first, third, and fourth arguments are zero. The first is zero because the retained value already has at least eleven low zero bits.

This call’s 32-bit result supplies the third coprocessor-directed operation and is then passed unchanged to another call.

### Further forwarding and signed reconstruction

Additional calls follow, including:

- A chain in which one 32-bit result is passed through another dependency and then to a further dependency.
- Calls whose results are not visibly consumed.
- A result-producing call followed by a consumer of that result.
- A call with explicit zero.
- A three-argument call receiving zero, another incompletely identified secondary register value, and zero.

A later dependency returns a represented signed 32-bit value. The procedure retains that value shifted left by five bits. The next call receives the original value, a zero-derived argument, and the retained shifted value, in that order.

A further result-producing call and its consumer execute before **F-3de949054bdd** is invoked with no displayed explicit arguments. No result from that identified dependency is visibly consumed.

Two additional result-producer/consumer pairs follow. A subsequent two-argument call receives:

- Zero derived from the earlier zero chain.
- The retained left-shift-by-five value shifted right by five bits.

If the left shift truncates to 32 bits and the right shift is arithmetic, the second argument is the original value’s low 27 bits sign-extended to 32 bits. It is not necessarily the original value. This call’s result is then passed to another dependency.

### Later coprocessor operations and high-position bits

A subsequent 64-bit result supplies its low word to the fourth coprocessor-directed operation. Its high word shifted left by one bit becomes the sole displayed argument of the next call.

Another call produces a 32-bit word used unchanged by two consecutive coprocessor-directed operations. These operations have different incoming coprocessor operands.

A later call produces a 32-bit word that is shifted left by 27 bits and retained. A four-argument call receives zero, zero, that retained value, and zero.

After several intervening calls, another result supplies the seventh coprocessor-directed operation. Three further calls occur before the retained left-shift-by-27 value, shifted left by another 24 bits, supplies the eighth operation. A two-argument call then receives that same shifted value and zero.

Under 32-bit truncating arithmetic, the retained value has possible set bits only in positions 27–31. Therefore:

- Shifting it left by another 24 bits yields zero.
- Masking it with `0x007FFFFF` yields zero.
- Shifting that masked result left by four bits also yields zero.

After another call without displayed arguments, a result-producing call supplies the ninth coprocessor-directed operation. Another call follows.

### Final retained word and zero-derived chains

A call receiving three zero arguments returns a 32-bit word. This result produces two retained transformations:

- A left shift by one bit, supplied to the tenth coprocessor-directed operation.
- A left shift by four bits, used in later calls and arithmetic.

A four-argument call receives, in order:

1. The returned word shifted left by one bit.
2. The earlier masked-and-shifted zero.
3. The returned word shifted left by four bits.
4. Zero.

Two calls without displayed arguments follow. The eleventh coprocessor-directed operation receives explicit zero.

The procedure then masks the retained left-shift-by-four value with `3` and shifts the result left by 16 bits. This derived value is zero under 32-bit semantics because the retained value’s low four bits are already zero.

The twelfth coprocessor-directed operation also receives explicit zero. A call with one explicit zero argument returns a word.

The thirteenth coprocessor-directed operation receives the retained left-shift-by-four value shifted left by another eight bits. A following three-argument call receives:

- That same additional-left-eight transformation.
- The retained left-shift-by-four value shifted left by another 12 bits.
- The most recent call result shifted left by nine bits.

A subsequent call produces a 64-bit result. Its low word, high word, and low word shifted left by one bit are passed to the next call.

The zero derived from the low-bit mask, shifted left by another eight bits, initiates a three-call result-forwarding chain. Another call without displayed arguments follows.

### Destination selection and final store

A later call produces a 64-bit result. The next call receives its low word, high word, and high word shifted left by 11 bits, and itself produces another 64-bit result.

This final 64-bit result has two visible uses:

- Its low word is passed to another dependency.
- Its upper bits determine the final store destination.

The destination calculation first logically shifts the 64-bit result right by 41 bits and narrows it to 32 bits. The retained quantity has at most 23 significant bits, so its represented signed interpretation is nonnegative. Shifting it right by another 11 bits therefore yields the original result’s top 12 bits without a signedness difference.

Let \(t\) be those top 12 bits, interpreted as an unsigned integer. The attempted destination is:

\[
A=t+\left\lfloor t/4\right\rfloor,\qquad 0\leq t\leq4095.
\]

Thus \(0\leq A\leq5118\), with gaps in the possible addresses. The addition cannot overflow a 32-bit word. No separate destination base is supplied.

After the low-word consumer, three more calls occur:

1. A call producing a 32-bit word.
2. A call receiving that word and a zero-derived second argument.
3. A call with no displayed arguments.

The procedure then attempts a 16-bit store of `0x00CD` at \(A\). The next represented event is the invalid-instruction termination marker.

## Side effects

### Dependency effects

Every dependency call is potentially observable. Unavailable dependency bodies may read or modify memory, globals, device state, or execution state; manage resources; invoke other code; raise faults; or fail to return.

The visible caller establishes neither global-state preservation nor ownership preservation across these boundaries.

### Coprocessor-directed operations

Thirteen such operations appear. Their explicit data operands, in order, are:

| Position | Data operand |
|---:|---|
| 1 | An early 32-bit call result. |
| 2 | The low word of an early 64-bit result. |
| 3 | The result of the four-argument call containing an unresolved secondary register value. |
| 4 | The low word of a later 64-bit result. |
| 5 | A later 32-bit call result. |
| 6 | The same word as operation 5. |
| 7 | Another call result. |
| 8 | The retained left-shift-by-27 value shifted left by another 24 bits; conditionally zero. |
| 9 | Another call result. |
| 10 | A call result shifted left by one bit. |
| 11 | Explicit zero. |
| 12 | Explicit zero. |
| 13 | The retained left-shift-by-four value shifted left by another eight bits. |

The operation selectors and incoming coprocessor operands vary. Their semantics, privilege requirements, architectural ordering, and exception behavior are unknown. They must not be assumed to be harmless register assignments or ordinary memory writes.

### Direct memory access and failure visibility

The final halfword store is the only direct ordinary-memory write shown in the body. Its byte order is unspecified.

The calculated destination may be unmapped, protected, unaligned, ordinary memory, or device memory. No caller-side alignment adjustment, bounds check, or access check is shown.

No caller-side rollback or compensation is shown. Earlier effects may remain observable after a later failure. Unknown dependency behavior, exception handling, or external mechanisms could nevertheless perform rollback or compensation; the visible evidence does not establish either their presence or their absence.

## Dependencies

Only two dependency identities are supplied:

| Opaque function ID | Visible boundary |
|---|---|
| **F-0e02e4c13579** | Receives one 32-bit call result. No return value is visibly consumed. |
| **F-3de949054bdd** | Receives no displayed explicit arguments. No return value is visibly consumed. |

The remaining indirect dependencies have unresolved identities. No invented function IDs are assigned to them.

Their visible contracts consist only of the ordered argument/result relationships above. Displayed argument counts may omit implicit inputs. Nothing establishes whether repeated-looking target calculations identify the same callable dependency.

## Acceptance examples

These are conditional static examples, not executed tests.

### Initial shift truncation

Assuming 32-bit truncating left shifts:

| First input | First call’s fourth argument |
|---|---|
| `1` | `0x04000000` |
| `64` | `0x00000000` |
| `0xFFFFFFFF` | `0xFC000000` |

An interpretation using unbounded multiplication would not preserve these results.

### Derived zeros and retained bits

Under the displayed fixed-width dataflow:

- Shifts of the explicitly initialized zero remain zero.
- The original word shifted left by 11 and then by 17 retains only its original low four bits, moved into positions 28–31.
- That left-shift-by-11 value shifted left by 31 instead yields zero.
- A word shifted left by 27 has no set bits under mask `0x007FFFFF`.
- A word shifted left by four has no set bits under mask `3`.

These conclusions constrain the corresponding call and coprocessor operands; they do not imply that intervening calls lack effects.

### Signed reconstruction boundary

Assuming truncating 32-bit left shift and arithmetic right shift:

| Original word | Left-five/right-five result |
|---|---|
| `0x0000001F` | `0x0000001F` |
| `0x04000000` | `0xFC000000` |
| `0x08000000` | `0x00000000` |

Always substituting the original word would violate this conditional behavior.

### Final destination

If execution reaches the store:

| Final 64-bit result’s top 12 bits | Attempted address |
|---:|---:|
| 0 | 0 |
| 1 | 1 |
| 3 | 3 |
| 4 | 5 |
| 4095 | 5118 |

Every case attempts to write the halfword `0x00CD`. Lower bits of that result do not affect the calculated address, although its low word is separately passed to a dependency.

Address 1 is possible. The contract does not require that an unaligned halfword access succeed.

### Interrupted execution

If an earlier dependency never returns, subsequent operations in the represented sequence are not reached.

If the final store faults, the following invalid-instruction marker need not be reached. Earlier effects may remain visible; no caller-side recovery is shown, while dependency or fault-handler compensation remains unresolved.

## Uncertainties

The following remain **documented_only**:

- Validity of instruction decoding, function boundaries, and the apparent straight-line path.
- Actual calling conventions and completeness of displayed arguments.
- Whether represented 64-bit results are genuine return values or reconstructed register pairs.
- Provenance of the two secondary register operands.
- Actual machine shift semantics corresponding to represented signed expressions.
- Identities, effects, ownership contracts, and completion behavior of indirect dependencies.
- Coprocessor semantics, privilege requirements, ordering, and exceptions.
- Memory mappings, alignment policy, endianness, and store success.
- Fault handling, rollback, compensation, or externally managed recovery.
- The actual event represented by the invalid-instruction termination marker.

Extensive zero-producing arithmetic and the low-address store may reflect genuine behavior or invalid decoding. Neither interpretation is established.

## Evidence

This contract uses only the supplied static representation and dependency identity map: the represented signature, ordered calls, explicit argument/result dataflow, thirteen coprocessor-directed operations, halfword store, and invalid-instruction warnings.

The width reductions, zero deductions, signed-reconstruction examples, and destination range are conditional arithmetic deductions, not execution observations.

No dependency bodies, separate implementation, runtime execution, or tests were accessed. No successful completion, memory-write result, rollback behavior, or architectural exception was observed.
