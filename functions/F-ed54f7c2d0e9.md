<!-- Trusted provenance: F-ed54f7c2d0e9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ed54f7c2d0e9

## Purpose

F-ed54f7c2d0e9 repeatedly performs an ordered sequence of indirect calls, byte accesses, and 32-bit word accesses. Its visible control flow is an unconditional loop. There is no visible normal return, termination condition, validation branch, or error-reporting path.

The operations resemble repeated construction or updating of encoded state, but that interpretation is **inferred intent**, not an established application-level purpose. The supplied material does not establish a device, protocol, allocator, object model, or meaningful relationship between the addressed regions.

The behavioral contract is therefore an ordered memory-and-call trace, conditional on the supplied representation accurately describing executable behavior. Its unusual address calculations, incoming hidden state, and unresolved call targets prevent a stronger semantic contract.

## Interface

### Explicit inputs

The displayed interface has four input positions:

| Position | Visible width | Visible use |
|---|---:|---|
| First | 32 bits | Forwarded to the first indirect call; replaced at the end of each completed iteration. |
| Second | 32 bits | Forwarded with the first input; replaced at the end of each completed iteration. |
| Third | 32 bits | No explicit use appears in the supplied body. |
| Fourth | 32 bits | Used in scaled address and argument calculations; replaced before the next iteration. |

The displayed signed types do not establish that negative inputs are meaningful or rejected. No range checks are visible.

### Additional incoming state

The explicit inputs are insufficient to determine execution. The body also consumes:

- An incoming control-location value used to select the first indirect call target.
- An incoming address-like value used for the first byte lookup.
- An incoming byte used for the first byte store.
- An incoming 32-bit value used in a left-shifted call argument.
- An additional call-result register value used late in the iteration, without a corresponding explicit result assignment.
- A 32-bit global value read during each iteration.

These are required environmental inputs, not initialized local state. Their entry values and calling-convention guarantees are unknown.

### Outputs and termination

There is no explicit result value or normal return in the visible body. Observable outputs consist of:

- Indirect call invocations and their arguments.
- Byte and 32-bit memory writes.
- Memory reads that may themselves be observable.
- Updated state carried into the next iteration.

If a dependency fails to return, execution does not reach subsequent operations. Invalid memory accesses or invalid call targets may prevent progress. No recovery behavior is visible.

### Preconditions

For a complete iteration to proceed:

1. Each reached indirect target must be executable under the actual calling convention.
2. Each reached read and write must be permitted at its effective address and access width.
3. The hidden incoming state must be meaningful to that execution environment.
4. Every dependency whose return is needed for continued execution must return.

No evidence establishes alignment handling, ordinary RAM semantics, mapped low-address regions, or protection against overlapping accesses.

## Behavior

### Arithmetic and trace notation

The visible operations use bytes, 32-bit words, and several results represented as 64-bit pairs.

For the trace below:

- “Scaled” and “shifted” describe the displayed fixed-width operations.
- A byte destination receives only the low eight bits of its supplied value.
- A byte loaded for address formation is zero-extended: its numeric range is 0–255.
- A split result provides a low 32-bit half and a high 32-bit half.
- A **four-call acquisition** means three calls with no explicit arguments and discarded results, followed by a fourth call with no explicit arguments whose result is retained. These are four separate calls, not one helper.

The equations below use 32-bit modular arithmetic to make the apparent machine-level dataflow explicit. This is not a claim that overflowing signed expressions in the displayed representation have independently established language-level semantics. Actual instruction-level overflow behavior remains **documented_only**.

Values retained across calls mean the values shown as retained in the supplied representation. They do not imply that unresolved dependencies preserve arbitrary registers or globals.

### 1. Initial calls and first stores

Each iteration begins by calling an unresolved dependency with the current first and second inputs.

A following call produces a split result. Both halves are passed, low half first, to another dependency. One further call has no explicit arguments and its result is discarded. The next call produces a 32-bit value.

Execution then:

1. Reads a byte at the incoming address-like state plus 1.
2. Adds 17 to that byte’s numeric value.
3. Stores the incoming byte at the resulting address.
4. Calls a dependency with the preceding produced value shifted left by 2.

A four-call acquisition produces another value. Define the **primary word** as that value shifted left by 2.

The function writes the primary word as a 32-bit word at:

\[
4096 \times \text{current fourth input} + 20.
\]

It then calls a dependency with four arguments, in order:

1. Current fourth input multiplied by 1024.
2. The same scaled input.
3. Incoming carried 32-bit state shifted left by 12.
4. Primary word.

A four-call acquisition supplies an index. The primary word is written at four times that index plus 20. A dependency is then called with four times the index.

### 2. Two scaled-word stages

A four-call acquisition supplies a new index. Its value multiplied by 4 becomes the **first scaled word**.

The function writes that word at sixteen times the index plus 20, then calls a dependency with three identical arguments, each equal to the first scaled word.

A four-call acquisition supplies another index. The first scaled word is written at four times this index plus 20. A dependency receives four times that index.

Another four-call acquisition supplies a new index. Its value multiplied by 4 becomes the **second scaled word**. The function:

1. Writes the second scaled word at sixteen times this index plus 20.
2. Retains a **byte-region base** equal to this index multiplied by \(2^{30}\).
3. Calls a dependency with three copies of the second scaled word.

A further four-call acquisition supplies an index. The second scaled word is written at four times that index plus 20, and a dependency receives four times that index.

The byte-region base is not established to be an allocated object. Under 32-bit modular arithmetic it has only four possible bit patterns: zero, \(2^{30}\), \(2^{31}\), or \(3 \times 2^{30}\).

### 3. Byte clearing and indexed word read

A four-call acquisition produces a retained **selector**.

The function writes zero first at byte-region base plus 17, then at byte-region base plus 1. It calls a dependency with the selector shifted left by 2.

The next dependency returns a 32-bit value, which is passed to another dependency. A four-call acquisition follows. Its result, shifted left by 2, is passed to a dependency, but only after the function again writes zero at:

1. Byte-region base plus 17.
2. Byte-region base plus 1.

These repeated writes remain distinct observable operations.

A four-call acquisition supplies a new index. The function retains four times this index as a **later call argument** and writes a word at sixteen times the index plus 20. The stored value is the earlier index used to address the second scaled word’s four-times-index destination, shifted left by 31.

Next, it reads a 32-bit word from:

\[
((\text{selector} \mathbin{\&} (2^{30}-1)) \gg 20) + 20.
\]

The added index lies between 0 and 1023. The read is not visibly scaled by four; alignment is not established.

A dependency receives the later call argument. The immediately following call produces a retained **link value**. Another dependency receives:

\[
(\text{loaded word} \gg 22) \ll 28.
\]

With 32-bit arithmetic, only original word bits 22–25 survive that final expression.

### 4. Indirect byte updates

A four-call acquisition supplies an index. The function writes a zero word at four times that index plus 20, then calls a dependency with four times the index.

Another four-call acquisition produces a retained **byte-update value**. The function:

1. Reads a byte at byte-region base plus 1.
2. Writes the low byte of the byte-update value shifted left by 2 at the loaded byte’s numeric value plus 17.
3. Calls a dependency with the byte-update value shifted left by 2.

A following call returns a value. A dependency receives that result and the byte-update value shifted left by 30, in that order.

A four-call acquisition supplies another result. The function now reads byte-region base plus 1 again and retains the loaded byte. It writes:

\[
(\text{byte-update value} \mathbin{\&} (2^{30}-1)) \gg 20
\]

to the retained byte’s numeric value plus 17, truncating to eight bits.

It calls a dependency with the latest acquired result shifted left by 2. Four further calls with no explicit arguments follow, and their results are not used here.

The function then writes the link value shifted left by 2 as a 32-bit word at the retained byte’s numeric value plus 20.

The retained byte is not reloaded for this word store, despite the intervening calls.

### 5. Global sample and pair-derived updates

The function samples a 32-bit global, then calls a dependency with the later call argument retained in stage 3.

A four-call acquisition produces a split result. The low half shifted left by 2 becomes the **first pair-derived word**.

Before forwarding the high half, the function rereads byte-region base plus 1 and writes the low eight bits of the sampled global shifted right by 22 at the loaded byte’s numeric value plus 17.

A dependency receives the high half. A following call produces a value. Another dependency receives that value and the first pair-derived word shifted left by 28.

A four-call acquisition supplies a result. The function rereads byte-region base plus 1 and writes zero at the loaded byte’s numeric value plus 17. It then calls a dependency with the acquisition result shifted left by 2.

A four-call acquisition supplies another result. A dependency receives two copies of that result shifted left by 2.

A further four-call acquisition produces a value whose left shift by 2 is the **third scaled word**. The function writes this word at:

\[
4096 \times \text{first pair-derived word} + 20,
\]

then calls a dependency with the first pair-derived word shifted left by 14.

A four-call acquisition supplies an index. The third scaled word is written at four times that index plus 20. A dependency receives four times the index.

### 6. Later split result and final word stages

A four-call acquisition produces another split result. Its low half shifted left by 2 becomes the **second pair-derived word**.

The function reads a byte at:

\[
16384 \times \text{first pair-derived word} + 1
\]

and writes the low eight bits of:

\[
(\text{third scaled word} \mathbin{\&} (2^{30}-1)) \gg 20
\]

at the loaded byte’s numeric value plus 17.

A dependency receives the new split result’s high half. A following call returns a value, which is passed to another dependency. The next call produces a split result, and both halves are forwarded, low first, to a dependency.

One call with no explicit arguments has a discarded result. The next call supplies a result.

The function reads a byte at:

\[
2^{28} \times \text{second pair-derived word} + 1
\]

and writes zero at the loaded byte’s numeric value plus 17. A dependency receives the latest result shifted left by 2.

A four-call acquisition produces a retained **final index**. The function establishes the **next address-like state** as four times the second pair-derived word.

It reads a byte at that state plus 1 and writes zero at the loaded byte’s numeric value plus 17. A dependency receives the second pair-derived word shifted left by 10.

A following call returns a value. Another dependency receives that value and the link value shifted left by 28.

A four-call acquisition supplies a result. The function again reads next address-like state plus 1 and clears the byte at the loaded byte’s numeric value plus 17. A dependency receives the latest result shifted left by 2.

Four calls with no explicit arguments follow; their results are unused here.

The **final word** is the final index shifted left by 14. The function writes it at:

\[
4096 \times (4 \times \text{final index}) + 20,
\]

then calls a dependency with four times the final index, shifted left by 14.

A call with no explicit arguments follows. Another dependency receives:

1. Final word shifted left by 28.
2. An additional result-register value not explicitly assigned in the supplied representation.

Under the stated modular interpretation, the first argument is necessarily zero: the final word already has fourteen low zero bits.

A four-call acquisition supplies an index. The final word is written at four times that index plus 20. A dependency receives four times the index.

### 7. Next-iteration state

A final four-call acquisition produces the next carried value after shifting its result left by 2.

The function rereads next address-like state plus 1 and writes the low eight bits of:

\[
(\text{final word} \mathbin{\&} (2^{30}-1)) \gg 20
\]

at the loaded byte’s numeric value plus 17.

A dependency receives the final index shifted left by 26. A following call produces the next first input.

Before repeating:

- The fourth input becomes the newly acquired carried value.
- The second input becomes that updated fourth input shifted left by 28.
- The carried address-like state becomes the next address-like state.
- The carried byte becomes zero.
- The carried shifted-input state becomes the updated fourth input.
- The control-location value used to select the next iteration’s first target is replaced by a fixed location represented in the supplied body.

The third input remains visibly unused.

## Side effects

All explicit stores are one byte or four bytes wide. No allocation, release, buffer copying, ownership transfer, or resource lifetime operation is established.

Byte-based destinations are especially constrained: after loading a byte and adding 17, the destination address is between 17 and 272. Corresponding byte-derived word destinations, formed by adding 20, lie between 20 and 275. These calculations are numeric address formation, not evidence of a conventional pointer field.

Dependencies occur between many reads and writes. They may modify memory later read by the function. Consequently:

- Repeated byte lookups cannot be replaced by one cached lookup.
- Repeated zero stores cannot be removed merely because the function previously wrote zero.
- The global sample cannot be moved across surrounding calls.
- The retained-byte word store must not be changed into a fresh byte lookup.
- Apparently equal or overlapping addresses must preserve the stated access order.

No atomicity, synchronization, volatility, or transaction guarantee is established.

## Dependencies

For this contract, unresolved call occurrences are identified by the opaque scheme **F-U001, F-U002, …**, numbered in their visible execution order within one iteration. These are occurrence identifiers, not recovered callee identities. Distinct occurrences may resolve to the same function.

The four-call acquisitions above each represent four successive opaque dependency occurrences. Calls with discarded results remain part of the observable sequence.

Established dependency characteristics are limited to visible argument passing and result consumption:

- Calls have zero through four explicit arguments.
- Several results are consumed as low/high 32-bit pairs.
- Some results become addresses, indexes, stored data, or later arguments.
- The first target depends on incoming control state.
- Other targets are expressed through fixed control-location-relative calculations.
- Several target calculations use offsets different from the predominant one.

Unknown dependencies must not be assumed to preserve global state, retain or release memory, validate addresses, or implement allocation. Their errors and side effects are **documented_only**.

## Acceptance examples

These examples are conditional trace checks, not executed tests.

1. **Byte-derived addressing.** If the initial byte lookup returns 7 and the incoming byte is 165, the first explicit store writes byte value 165 at numeric address 24.

2. **Primary word scaling.** If the fourth input is 2 and the acquisition producing the primary word returns 3, the corresponding word store writes 12 at address 8212, subject to successful preceding calls and accesses.

3. **Selector lookup boundary.** Selectors differing only in their top two bits select the same indexed word-read address. The address range is 20–1043 inclusive; it is not restricted to four-byte-aligned addresses.

4. **Byte truncation.** A byte-update value of 64 yields zero for its shifted-by-two byte store, while the following dependency receives the full 32-bit value 256.

5. **Reload versus retention.** If a dependency changes byte-region base plus 1 between two explicitly described lookups, the later lookup observes the changed byte. In contrast, the link-value word store uses its previously retained byte even if intervening calls change that memory.

6. **Global sample timing.** If a later dependency changes the global after its explicit sample, the stage-5 byte write uses the sampled value, not the new global value.

7. **Fixed-width collapse.** Under 32-bit modular arithmetic, shifting the final word left by 28 produces zero for every final index. A nonzero first argument at that call would contradict this arithmetic interpretation or the supplied dataflow.

8. **Continuation.** If all dependencies return and all accesses succeed, completion of the final stage starts another iteration. It does not return to the caller.

## Uncertainties

The following remain **documented_only**:

- Whether the supplied representation has correct function boundaries and complete reachable control flow.
- The exact calling convention and source of hidden incoming state.
- Concrete dependency identities, aliasing between targets, and dependency contracts.
- The meaning and layout of addressed memory.
- Whether memory is ordinary RAM, device-backed, protected, or unmapped.
- Byte order and unaligned word-access behavior.
- Instruction-level semantics of overflowing arithmetic.
- The origin and validity of the additional result-register value.
- Whether dependencies preserve the state that the representation depicts as retained.
- Exception, fault, cancellation, and external termination behavior.

The regular scaled stores suggest an encoded-state update process, but no domain interpretation is accepted as established behavior.

## Evidence

This contract uses only the supplied static representation. It establishes the displayed access widths, arithmetic expressions, call ordering, retained-value dependencies, global sample, repeated stores, and unconditional loop.

The supplied diagnostic annotations indicate uncertainty involving omitted unreachable material and overlapping global symbols. They do not establish runtime behavior or resolve the missing calling and memory contracts.

No runtime tests have been performed. No execution trace, instruction-level verification, dependency implementation, or external interface documentation was available. The acceptance examples are conditional consequences of the displayed dataflow, not reported observations.
