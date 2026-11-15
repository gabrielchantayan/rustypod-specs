<!-- Trusted provenance: F-ee2227deaf87; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ee2227deaf87

## Purpose

F-ee2227deaf87 describes an ordered interaction with **227 indirect call sites**, interrupted by four explicit one-byte memory writes. Several call results are transformed before being passed to subsequent dependencies. One result also selects the basis for the first write address.

No application-level purpose is established. The representation does not identify a protocol, object type, device, allocation scheme, or success condition. The repeated interactions do not by themselves establish initialization, decoding, rendering, or table construction.

**Evidence boundary:** the supplied representation includes invalid-instruction and truncated-control-flow warnings. This contract describes the visible path and its data relationships, conditional on that representation accurately reflecting execution. It does not establish a valid function boundary, an executable complete path, or a normal return.

## Interface

### Declared interface and implicit inputs

The represented declaration has no explicit parameters and no returned value. No normal-return endpoint is visible.

The behavior nevertheless depends on incoming machine state:

| Descriptive input | Represented type or width | Use |
|---|---|---|
| Entry control-link value | Integer in the apparent 32-bit model; actual address width unresolved | Determines the first indirect target after addition of `0xFFE`. |
| Entry address seed | Signed integer in the apparent 32-bit model | Supplies the first write’s basis when the relevant producer result has bit 30 clear. |
| Entry payload word | Unsigned 32-bit value | Supplies the first write’s low byte and the second write’s extracted byte. |

These labels describe dependencies, not recovered formal parameters. The actual calling convention and the validity of treating the referenced incoming values as stable across intervening calls remain unresolved.

Other inputs include dependency results, dependency-visible state, executable mappings for reached targets, and writable mappings for computed destinations. No assumption is made that dependencies preserve globals or memory.

### Outputs and layout

Directly visible outputs are:

- Four one-byte writes.
- Ordered indirect calls, including their explicit argument relationships.
- Any additional effects performed by those dependencies, whose contracts are unavailable.

No returned pointer, status, output structure, buffer length, or aggregate layout is established. Address scaling does not establish an array stride or record size.

The representation uses unsigned 32-bit results, signed integer results in an apparent 32-bit model, and opaque four-byte results. An opaque four-byte result is not established to be a pointer or an arithmetic scalar.

### Preconditions for reaching the endpoint

A complete traversal of the represented path requires:

- Every reached indirect target to be executable and callable under the actual ABI and execution mode.
- Every dependency to return normally.
- Every direct byte destination to permit the write.
- Arithmetic, integer conversions, and address formation to be meaningful under the actual execution semantics.

No visible validation establishes these conditions.

## Behavior

### Dependency naming and ordering

**F-aux-001 through F-aux-227** are document-local opaque identifiers for call sites, assigned in invocation order. They do not establish distinct implementation identities. Different call sites may ultimately reach the same implementation.

“No explicit arguments” means that no arguments are shown in the representation. It does not establish that a dependency ignores incoming registers, memory, globals, or other machine state.

“Result not consumed” means only that this function shows no later use of that result. It does not make the call removable.

### Interaction sequence

An ordinary **scaled-transfer group** contains five ordered call sites:

- Three calls whose results are not explicitly consumed.
- A producer call.
- A consumer call receiving the producer’s result shifted left by two bits.

Each group has its own call sites. This description does not imply a runtime loop or interchangeable dependencies.

| Call-site range | Required visible relationship |
|---|---|
| F-aux-001 | Initial call, with a target derived from the entry control-link value. Its result is not explicitly consumed. |
| F-aux-002–003 | Producer followed by a consumer receiving its result shifted left by two bits. |
| F-aux-004–007 | Three calls with unconsumed results, followed by the producer used for the first write. |
| F-aux-008 | After the first write, receives two identical copies of the selected first-write basis. |
| F-aux-009–013 | Three calls with unconsumed results, then an opaque four-byte producer result forwarded unchanged to a one-argument consumer. |
| F-aux-014–058 | Nine ordinary scaled-transfer groups. |
| F-aux-059–063 | Three calls with unconsumed results, a producer, the second byte write, then a two-argument consumer. |
| F-aux-064–073 | Eight calls with unconsumed results, followed by a producer and a consumer receiving its result shifted left by two bits. |
| F-aux-074–113 | Eight ordinary scaled-transfer groups. |
| F-aux-114–118 | Three calls with unconsumed results, a producer, the third byte write, then a two-argument consumer. |
| F-aux-119–123 | Three calls with unconsumed results, then an opaque four-byte producer result forwarded unchanged to a one-argument consumer. |
| F-aux-124–168 | Nine ordinary scaled-transfer groups. |
| F-aux-169–173 | Three calls with unconsumed results, a producer, the fourth byte write, then a two-argument consumer. |
| F-aux-174–178 | Three calls with unconsumed results, then an opaque four-byte producer result forwarded unchanged into both argument positions of a consumer. |
| F-aux-179–223 | Nine ordinary scaled-transfer groups. |
| F-aux-224–227 | **Four further calls**, in order, with no explicit arguments and no explicitly consumed results. The invalid-instruction/truncation marker follows F-aux-227. |

The final four calls are part of the dependency interaction contract. In particular, F-aux-227 is not a return operation or part of the truncation marker. Its effects are unknown, just as for the other unconsumed-result calls.

### First write: conditional address basis

F-aux-007 returns an unsigned 32-bit value. Bit 30 of that value determines the selected basis:

- When bit 30 is set, the basis comes from that result shifted left by two bits.
- When bit 30 is clear, the basis comes from the entry address seed shifted left by ten bits.

The destination address is twice the selected basis. The function writes exactly one byte there, using the low eight bits of the entry payload word.

The write occurs after F-aux-007 and before F-aux-008. F-aux-008 receives the selected basis in both explicit argument positions. Neither argument is the doubled destination address.

The predicate is specifically a bit-30 test. It is not a sign test, nonzero test, or general threshold comparison.

The result-derived shift is represented as unsigned arithmetic before its conversion to the integer basis. The seed-derived shift and subsequent doubling require separate consideration of signed arithmetic and address conversion.

### Second write: payload extraction

F-aux-062 produces the integer used for the second destination and the following consumer arguments.

The function:

- Writes one byte at the address derived by shifting that result left by three bits.
- Takes the stored byte from payload bits 22 through 29, inclusive.
- Then invokes F-aux-063 with two distinct arguments: first the producer result multiplied by four, then the producer result shifted left by ten bits.

The payload extraction uses an unsigned right shift by 22 followed by byte narrowing. Payload bits 30 and 31 do not contribute to the stored byte.

The destination transformation and the two argument transformations are different. Their order and scale factors are not interchangeable.

### Third and fourth writes: single-byte zeroing

F-aux-117 supplies the third write’s basis. F-aux-172 supplies the fourth write’s basis.

At each site:

- The destination is formed by shifting the corresponding producer result left by three bits.
- Exactly one byte is written with value zero.
- The subsequent consumer receives two identical copies of that producer result shifted left by fourteen bits.

The third write precedes F-aux-118. The fourth precedes F-aux-173.

These operations establish only two individual zero-byte stores, not clearing of words, records, buffers, or allocations.

### Arithmetic and address interpretation

The representation supports an apparent 32-bit integer model, but does not independently establish the target ABI or pointer width.

Within that model:

- The payload and F-aux-007’s result are unsigned 32-bit values.
- Their unsigned shifts operate on 32-bit values; left-shifted high bits are discarded under that interpretation.
- Assigning an unsigned shifted result to a signed integer may require a conversion whose treatment of out-of-range values is unresolved.
- Other producer results used in shifts and multiplication are represented as signed integers.
- A byte write retains the selected low eight bits.
- Converting an integer expression to a destination address does not establish a valid object or writable mapping.

For nonnegative signed 32-bit inputs, the following limits avoid a result exceeding the signed range:

| Transformation | Largest nonnegative input with a signed 32-bit result |
|---|---:|
| Multiply by two | `0x3FFFFFFF` |
| Multiply by four or shift left by two | `0x1FFFFFFF` |
| Shift left by three | `0x0FFFFFFF` |
| Shift left by ten | `0x001FFFFF` |
| Shift left by fourteen | `0x0001FFFF` |

These are individual-operation limits. Combined calculations may require tighter bounds. For example, the first write’s seed-derived branch includes both a ten-bit shift and doubling.

Negative signed operands, signed overflow, and integer-to-address conversions remain unresolved. The contract does not impose wrapping behavior on all signed operations merely because the representation uses shifts and multiplication.

### Errors, interruption, and endpoint

No error-status branch, bounds check, null check, retry, rollback, or recovery path is visible.

A dependency that does not return prevents later operations on this represented path from being reached. A faulting call or store has no visible local recovery. No explicit result is interpreted as a failure indicator.

After F-aux-223, the function still invokes F-aux-224, F-aux-225, F-aux-226, and F-aux-227 in that order. Only then does the representation reach its invalid-instruction/truncation marker.

The marker establishes neither normal return nor verified processor halt. Actual endpoint behavior is **documented_only**.

## Side effects

### Direct reads and writes

Four explicit data-memory stores occur, ordered as follows:

1. Payload low byte at the conditionally selected destination.
2. Payload bits 22–29 at the destination derived from F-aux-062.
3. Zero at the destination derived from F-aux-117.
4. Zero at the destination derived from F-aux-172.

No explicit data-memory read through these destinations is shown before the stores. The visible bit selections consume represented incoming values rather than establishing fresh memory reads.

The four destinations may alias. For ordinary memory, later reached stores occur after earlier stores in the represented sequence. However, intervening and subsequent dependencies may read or overwrite the same bytes. The last direct store to an address therefore does not establish that address’s value at the truncated endpoint.

No device-memory semantics, atomicity guarantee, synchronization effect, or cross-thread ordering guarantee is established.

### Dependency state and ownership

Every dependency, including F-aux-224–227, may have effects not exposed by the representation. No dependency is assumed to:

- Preserve globals or previously written memory.
- Allocate, retain, or release storage.
- Treat an argument as a pointer rather than a scalar or encoded value.
- Preserve incoming machine-state values beyond the guarantees of an unknown ABI.

No allocation, release, ownership transfer, borrowing interval, or retention policy is identifiable from the visible body. Passing a value once or twice does not establish such a policy.

No transaction, lock, reentrancy guarantee, or rollback boundary is shown.

## Dependencies

All 227 represented calls are indirect.

F-aux-001 is distinguished by deriving its target from incoming control-link state plus `0xFFE`. The remaining targets use represented fixed code-related bases plus the same displacement. This does not establish valid targets, callable signatures, or implementation identities.

The dependencies have these observable roles:

| Role | Call sites or relationship |
|---|---|
| Entry-dependent target | F-aux-001 |
| Ordinary scaled forwarding | The scaled-transfer groups, plus F-aux-002–003 and F-aux-072–073 |
| Unchanged opaque forwarding | F-aux-012 to F-aux-013; F-aux-122 to F-aux-123 |
| Duplicated unchanged opaque forwarding | F-aux-177 to both explicit arguments of F-aux-178 |
| First-write production and consumption | F-aux-007 and F-aux-008 |
| Second-write production and consumption | F-aux-062 and F-aux-063 |
| Zero-write production and consumption | F-aux-117 and F-aux-118; F-aux-172 and F-aux-173 |
| Final unconsumed-result interactions | F-aux-224 through F-aux-227 |

No dependency implementation, error convention, ownership contract, or global-state contract is supplied.

## Acceptance examples

These examples are **conditional static acceptance cases**, not executed tests. They assume faithful recovery, the stated arithmetic interpretation, normal returns where needed, and writable destinations.

### First-write branch with bit 30 clear

Suppose F-aux-007 returns zero, the entry address seed is 3, and the payload is `0x123456AB`.

- The selected basis is 3,072.
- The first write stores `0xAB` at address 6,144.
- F-aux-008 receives 3,072 in both explicit argument positions.

The zero producer result does not force a zero destination. This branch uses the seed.

### First-write branch with bit 30 set

Under the apparent unsigned 32-bit shift model, suppose F-aux-007 returns `0x40000003` and the payload’s low byte is `0x5A`.

The result-derived shift yields 12, which fits the signed basis without an out-of-range conversion. Provided the address interpretation permits this destination:

- The first write stores `0x5A` at address 24.
- F-aux-008 receives 12 twice.
- The seed does not determine this write’s destination.

This example does not establish wrapping semantics for unrelated signed calculations.

### Extracted payload byte

Suppose F-aux-062 returns 5 and the payload is `0x3FC00000`.

- The second write stores `0xFF` at address 40.
- F-aux-063 receives 20 as its first explicit argument and 5,120 as its second.

Passing 40 as either scaled argument, or reversing the arguments, does not satisfy this relationship.

### Zero-store timing

Suppose F-aux-117 returns 2.

- The third write stores zero at address 16.
- It occurs before F-aux-118.
- F-aux-118 receives 32,768 in both explicit argument positions.

The example does not require the byte to remain zero after that dependency executes.

### Ordinary scaled transfer

If an ordinary scaled-transfer group’s producer returns 7, its consumer receives 28. The group’s three preceding calls still occur before the producer. Their unconsumed results do not justify omitting or reordering them.

### Final interaction boundary

If execution reaches F-aux-223 and every remaining dependency returns, all four calls F-aux-224–227 occur before the truncation marker.

An interaction record ending after F-aux-226 omits a represented dependency. If F-aux-226 does not return, F-aux-227 is not reached; if F-aux-227 does not return, the marker is not reached. Neither case establishes rollback.

### Interrupted write sequence

If F-aux-063 does not return, the second direct write has already completed on that path, but the third and fourth direct writes are not reached. Dependency effects and any earlier stores are not visibly undone.

## Uncertainties

The following remain **documented_only**:

- Accuracy of the represented instruction interpretation and function boundary.
- Actual integer widths, pointer width, ABI, execution mode, and indirect-target validity.
- Whether incoming-state references accurately preserve the values used after intervening calls.
- Signed overflow, negative signed shifts, out-of-range unsigned-to-signed conversion, and integer-to-address semantics.
- Dependency identities, signatures, memory effects, ownership behavior, and failure conventions.
- Validity, aliasing, and application meaning of the four destinations.
- Whether destinations refer to ordinary memory, executable memory, globals, devices, or dependency-managed storage.
- Effects of the four final dependencies, including F-aux-227.
- Actual execution behavior at the invalid-instruction/truncation boundary.

**Inferred intent:** repeated producer/consumer relationships suggest systematic transformation and forwarding. They do not establish the meaning of the values or an application-level goal.

## Evidence

Established static evidence in the supplied representation includes:

- A declaration without explicit parameters or a returned value.
- Incoming machine-state references.
- **227 indirect call expressions**, including four calls after F-aux-223.
- A bit-30 condition governing the first write’s selected basis.
- Four explicit one-byte stores.
- Unsigned payload extraction and unchanged, duplicated, and scaled result forwarding.
- An invalid-instruction/truncation marker following F-aux-227.

Call-site numbering and interaction grouping are descriptive indexing introduced by this contract, not recovered dependency identities.

No implementation access beyond the embedded representation, dependency inspection, runtime execution, or runtime tests have been performed. No observed execution result is claimed.
