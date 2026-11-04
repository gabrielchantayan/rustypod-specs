<!-- Trusted provenance: F-522ae6cca2ae; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-522ae6cca2ae

## Purpose

F-522ae6cca2ae conditionally requests an adjustment on an indirectly obtained object. The request depends on an input discriminator, successful object resolution, a nonzero intermediate field, and a signed comparison between two object fields.

Independently of whether an adjustment is requested, every normally completing invocation returns the second input’s 32-bit value in the upper half of a 64-bit result, with the lower half cleared.

**Established static evidence:** the function performs conditional reads and indirect calls. It contains no explicit memory stores, allocation, deallocation, or error reporting.

**Inferred intent:** the comparison and adjustment amount suggest enforcing a relationship with a margin of five units. The evidence does not establish what the objects or units represent. Names such as position, extent, scrolling, or visibility would impose unsupported semantics.

## Interface

### Inputs

| Input | Width | Contract |
|---|---:|---|
| First input | 32 bits | Not consulted by the visible function body. No interpretation or validity requirement is established. |
| Second input | 32 bits, unsigned | Used both as an address-like value for an initial read and as the value encoded in the return result. |

The second input is not checked for nullness or accessibility. The first operation requires a readable 32-bit field at byte offset `0x04` from its address-like value.

A nonzero second input is not, by itself, sufficient for safety. Conversely, the evidence does not establish a separate nonzero requirement if the addressed field is otherwise readable in the execution environment.

### Result

The externally declared result is 64 bits. Its bit layout on normal return is:

| Result bits | Value |
|---|---|
| 63–32 | The second input’s original 32-bit bit pattern |
| 31–0 | Zero |

Equivalently, the result bit pattern is the second input, zero-extended to 64 bits and shifted left by 32 positions.

If the interface exposes this bit pattern as a signed 64-bit integer, inputs with bit 31 set produce a negative signed result. The return value is not a success flag and does not distinguish skipped work from a completed adjustment request.

### Relevant memory layout

Offsets below are byte offsets. All listed fields are accessed as 32-bit words.

| Role | Offset | Observed use |
|---|---:|---|
| Input record | `0x04` | Discriminator compared with 5 |
| First resolved object | `0x00` | Dispatch-table address |
| Intermediate object | `0x34` | Nonzero value required for further resolution |
| Intermediate object | `0x84` | Signed comparison value and adjustment operand |
| Second resolved object | `0x00` | Dispatch-table address |
| Second resolved object | `0x8C` | Signed comparison value and adjustment operand |

The dispatch-table entries used are at byte offsets `0xF4`, `0xFC`, and `0x90`. The evidence supports a 32-bit pointer-oriented layout; it does not establish a portable layout for other pointer widths.

The minimum readable object extent depends on the branch reached. No object-size validation is visible.

## Behavior

### Initial gate

The function first reads the 32-bit discriminator at offset `0x04` from the second input.

If that value is not 5, it makes none of the dependency calls described below and proceeds to the fixed return encoding.

If the value is 5, it calls F-938c950e5046 without visible arguments. A null result ends the conditional work. A nonnull result becomes the first resolved object.

There is no later recheck of the input discriminator.

### Intermediate-object resolution

For a nonnull first resolved object, the function obtains an indirect callable from that object’s dispatch table at byte offset `0xF4`.

It calls that dependency with:

1. The first resolved object as receiver.
2. A value read from one fixed external data location.

The contents, type-level meaning, and ownership implications of that external value are unknown.

The visible representation treats this indirect call’s result as 64 bits. Only its lower 32 bits govern subsequent progress:

- A zero lower half ends the conditional work.
- A nonzero lower half is treated as the address of the intermediate object.

The upper 32 bits are extracted in the visible representation, but they do not independently enable any later operation. When the lower half is nonzero, the value relevant to the next gate is instead read from the intermediate object at offset `0x34`. When the lower half is zero, subsequent work is rejected regardless of the upper half.

Consequently, an implementation must not treat a nonzero upper half as a substitute for a missing intermediate object.

### Second-object resolution

For a nonnull intermediate object, the function reads its 32-bit field at offset `0x34`.

If that field is zero, it ends the conditional work.

Otherwise, it obtains another indirect callable from the first resolved object’s dispatch table, this time at byte offset `0xFC`. This dispatch-table access occurs after the earlier indirect call; it must not be assumed to use an unchanged table merely because the receiver address is the same.

The dependency receives:

1. The first resolved object as receiver.
2. The intermediate object’s previously read `0x34` field.
3. A value read from a second fixed external data location.

The returned value is treated as a 32-bit object address. A null result ends the conditional work. A nonnull result becomes the second resolved object.

No ownership transfer, reference acquisition, or lifetime extension is established by either resolution step.

### Comparison and adjustment request

Once the second resolved object is available, the function compares:

- The signed 32-bit field at offset `0x84` in the intermediate object.
- The signed 32-bit field at offset `0x8C` in the second resolved object, increased by five.

When the arithmetic is representable as signed 32-bit arithmetic:

- If the intermediate value is greater than the second value plus five, no adjustment call occurs.
- If it is less than or equal to that threshold, an adjustment call occurs.

For the adjustment, the function obtains an indirect callable from the second resolved object’s dispatch table at byte offset `0x90`. The arguments are:

1. The second resolved object as receiver.
2. The intermediate object’s `0x84` field minus the second resolved object’s `0x8C` field, minus five.
3. Zero.
4. Zero.

The two object fields appear as reads again in the adjustment expression. The contract therefore must not require the comparison’s values to be cached and reused. If those fields can change between accesses, the requested adjustment can reflect different values from those that admitted the branch.

With stable field values and representable arithmetic, the adjustment is nonpositive. Equality at the threshold produces an adjustment of zero, and the indirect call still occurs.

The adjustment dependency’s result is ignored.

### Completion

All ordinary branches converge on the same return encoding. There is no explicit retry, rollback, fallback object, or alternate return status.

This normal-return guarantee does not establish that invalid memory accesses, invalid indirect call targets, or nonreturning dependencies are recoverable.

## Side effects

### Direct effects

The visible body performs memory reads but contains no explicit memory writes.

Its observable interaction sequence is:

1. Read the input discriminator.
2. If eligible, call F-938c950e5046.
3. If resolution succeeds, dispatch the first indirect call.
4. If an intermediate object exists, read its `0x34` field.
5. If that field is nonzero, dispatch the second indirect call.
6. If a second object exists, read the comparison fields.
7. If the comparison admits adjustment, access the adjustment dispatch entry, read the adjustment operands, and invoke the final indirect call.
8. Return the encoded second input.

This sequence specifies branch and call ordering. It does not establish a total ordering among separate loads used within the same comparison or call expression.

### Indirect effects and lifetime

Any dependency may have effects not visible here. In particular, this contract does not assume that dependencies preserve global data, dispatch tables, object contents, or object lifetimes.

The first resolved object is used again after the first indirect call. The intermediate object is used again after the second indirect call. Defined completion therefore requires those later accesses to remain valid. The function performs no visible lifetime checks or retain operations to ensure this.

No release operation is visible. That absence is not evidence that returned objects are owned by the caller or that they should be released.

The two external data locations are consulted at their respective call sites. Their values must not be assumed to have been captured together or to remain constant across intervening dependency calls.

No synchronization, locking, atomic access, or concurrency protection is established.

## Dependencies

The unresolved indirect dependencies below receive local opaque labels for this contract only. These labels do not assert resolved target identities.

| Opaque function ID | Invocation and observable role | Unknowns |
|---|---|---|
| F-938c950e5046 | Called without visible arguments after the discriminator equals 5; supplies the first resolved object or null. | Actual return type, ownership, side effects, failure causes, and lifetime guarantees. |
| F-unknown-01 | Indirect dispatch through byte offset `0xF4` of the first resolved object’s table; receives that object and the first external value. Its result’s lower 32 bits supply the intermediate-object address. | Target identity, semantic meaning, true return convention, and side effects. |
| F-unknown-02 | Indirect dispatch through byte offset `0xFC` of the first resolved object’s table; receives that object, the nonzero intermediate field, and the second external value. Supplies the second object or null. | Target identity, argument meanings, ownership, and side effects. |
| F-unknown-03 | Indirect dispatch through byte offset `0x90` of the second resolved object’s table; receives that object, an adjustment, zero, and zero. | Target identity, adjustment semantics, return type, and effects. |

No supplied evidence permits collapsing these indirect dependencies into F-938c950e5046 or assigning them additional behavior.

## Acceptance examples

These are conditional acceptance cases derived from static evidence, not executed tests. Unless stated otherwise, assume valid readable objects, callable dispatch entries, returning dependencies, stable fields, and representable signed arithmetic.

### Discriminator rejects processing

Given a discriminator of 4:

- F-938c950e5046 is not called.
- No indirect dependency is called.
- The function returns the second input in the result’s upper 32 bits, with a zero lower half.

The same behavior applies to every discriminator other than 5.

### Initial resolution fails

Given discriminator 5 and a null result from F-938c950e5046:

- F-938c950e5046 is called once.
- No indirect dependency is called.
- The normal encoded result is returned.

### Intermediate address is absent

Given successful initial resolution and a result from F-unknown-01 whose lower 32 bits are zero:

- The intermediate object’s `0x34` field is not read.
- F-unknown-02 and F-unknown-03 are not called.
- A nonzero upper half of the result does not change this outcome.

### Intermediate field rejects processing

Given a nonzero intermediate-object address and a zero field at offset `0x34`:

- That field is read.
- F-unknown-02 is not called.
- No comparison or adjustment is performed.

### Second resolution fails

Given a nonzero intermediate field and a null result from F-unknown-02:

- The comparison fields are not subsequently read by this function.
- F-unknown-03 is not called.
- The normal encoded result is returned.

### Comparison rejects adjustment

Given an intermediate comparison value of 26 and a second-object comparison value of 20:

- The threshold is 25.
- No adjustment call occurs.

### Boundary equality still invokes adjustment

Given comparison values of 25 and 20:

- Equality admits the adjustment branch.
- F-unknown-03 receives adjustment 0 and two further zero arguments.

A zero adjustment must not be optimized into omission of the call.

### Negative adjustment

Given comparison values of 12 and 20:

- The threshold is 25.
- F-unknown-03 receives adjustment −13 and two further zero arguments.

### Signed comparison matters

Given comparison values of −1 and 0:

- Signed comparison admits the branch.
- The adjustment is −6.

An unsigned comparison would not preserve the evidenced behavior.

### Return encoding does not report adjustment

For a second-input bit pattern of `0x12345678`, every normally completing path returns `0x1234567800000000`.

For a second-input bit pattern of `0x80000000`, normal completion returns the bit pattern `0x8000000000000000`, regardless of its signed presentation.

## Uncertainties

The following are **documented_only**:

- **Overflow behavior.** Increasing the second comparison field by five can exceed the signed 32-bit range. The subtraction used for the adjustment can also overflow, including at an intermediate subtraction. The supplied representation does not establish a language-independent overflow policy. Wrapping, saturation, widening, and trapping must not be silently selected.
- **Address overflow and alignment.** Address calculations use a 32-bit address-oriented representation. Behavior when an offset crosses the address-space boundary, or when a word is misaligned, is not established.
- **Indirect return conventions.** The first indirect result is represented as 64 bits, but its actual ABI and semantic type are unresolved. Its lower-half use is established; the intended meaning of its upper half is not.
- **External data.** The values, initialization rules, and semantic types of the two fixed external data locations are unknown.
- **Dependency failures.** Null checks are established where described. Exception propagation, traps, nonlocal exits, and other dependency failure behavior are not.
- **Mutation and concurrency.** Repeated field accesses are visible, but atomicity, volatility, synchronization, and permitted concurrent mutation are not established.
- **Domain meaning.** The discriminator value, object roles, margin of five, and adjustment units have no established domain interpretation.

## Evidence

The sole evidence is the supplied static function representation and the supplied opaque dependency mapping.

It establishes the input widths, field offsets, explicit gates, dispatch offsets, visible argument values, signed arithmetic expressions, normal return encoding, and absence of explicit stores or release operations.

It does not establish the implementations of dependencies, external data contents, runtime object validity, or overflow behavior beyond the represented operations.

No runtime tests have been performed. The acceptance examples are conditional consequences of the static evidence, not observations from execution.
