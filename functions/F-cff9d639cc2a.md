<!-- Trusted provenance: F-cff9d639cc2a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cff9d639cc2a

## Purpose

F-cff9d639cc2a manages buffered output and processes new input through one of two dependency-driven paths. It first attempts to deliver previously recorded output. It then processes positive-length input when the input address is nonzero, delivering generated output before proceeding to another input chunk.

One processing path uses persistent transformation state. The other assembles three-byte input groups and retains an incomplete group across invocations. The exact transformations are not established.

The return value is not an output-delivery count:

- Failure while draining previously recorded output returns the nonpositive output-operation result.
- Failure while draining output generated from new input returns the original requested input length.
- Normal completion with positive input returns the original requested input length.
- An invocation without eligible new input returns zero after successful completion or omission of the initial drain.

Consequently, a positive return does not establish that all requested input was processed or that generated output was delivered.

## Interface

### Arguments and result

The supplied representation exposes three 32-bit arguments and a signed 32-bit result.

| Argument | Meaning |
|---|---|
| Context address | Address of the outer context. |
| Input address | Address passed to processing or transfer dependencies. Zero suppresses new-input processing. |
| Requested length | Signed 32-bit input length. Only positive values permit new-input processing. |

Addresses and address calculations are represented with 32-bit values. A wider-address interface is not established.

### Outer context

| Byte offset | Width | Observable use |
|---|---:|---|
| +0x14 | 32 bits | Flags. The low four bits are cleared on entry; bit 0x100 selects the processing path. |
| +0x20 | 32 bits | Subordinate-state address, loaded once on entry. |
| +0x24 | 32 bits | First argument to each output operation, reread for every call. |

The subordinate-state address is loaded before the flags are modified. The wrapper continues using that captured address even if a dependency changes the address stored in the outer context.

### Subordinate state

| Byte offset | Width or extent | Observable use |
|---|---:|---|
| +0x00 | Signed 32 bits | Recorded generated-output length. |
| +0x04 | Signed 32 bits | Recorded output progress. |
| +0x08 | Signed 32 bits | Retained-input count for the grouping path. |
| +0x10 | Signed 32 bits | Initialization marker, compared with 1. |
| +0x1c | Unknown extent | Address supplied to initialization and stateful processing. |
| +0x7c | Unknown extent | Output-storage address supplied to processing and output operations. |
| +0x65a | At least three bytes for the intended grouping scheme | Retained-input storage. |

These locations do not establish complete structure sizes, buffer capacities, or nonoverlap between regions.

### Preconditions and limits

The wrapper performs no explicit pointer, capacity, alignment, or accessible-range validation.

Ordinary defined operation requires accessible context and state storage, readable input for every dependency read, and sufficient writable output and retained-input storage. Address advances, signed differences, additions, and dependency results must remain valid under the target arithmetic and memory model.

For the intended grouping behavior, the retained count on entering a grouping iteration should be 0, 1, or 2. The wrapper does not enforce this invariant.

Input overlapping the context, state, or output storage is not rejected. Such overlap can affect subsequent wrapper reads and writes. Safety depends on actual overlap and dependency behavior; no general overlap guarantee is established.

## Behavior

### Entry and conditional initialization

Before checking the input address or requested length, the wrapper:

1. Loads the subordinate-state address.
2. Reads the flags and writes them back with the low four bits cleared.
3. Reads the initialization marker.

If the marker differs from 1, the wrapper writes, in order:

1. Marker = 1.
2. Generated-output length = 0.
3. Output progress = 0.
4. Retained-input count = 0.

It then calls F-45d85654c723 with the address at state offset +0x1c. Its direct result is unused.

Thus, reinitialization clears the records of pending output and retained input before the initialization dependency runs. It does not explicitly erase their storage. Subsequent behavior uses the state left by that dependency; preservation of the cleared fields is not assumed.

The marker is not checked again during the invocation.

### Initial drain of recorded output

After any initialization call, the wrapper reads the generated-output length and output progress. Their signed 32-bit difference becomes a locally maintained remaining-output amount.

While that amount is positive, the wrapper calls F-8d1c9f620c59 with:

- The current value at outer-context offset +0x24.
- Output storage at state offset +0x7c, advanced by the current recorded progress.
- The local remaining-output amount.

For a positive result, the wrapper rereads recorded progress after the call, adds the result, and writes the sum back. It subtracts the same result from the local remaining-output amount.

For a zero or negative result, the wrapper calls F-aeffcebde61f with the outer-context address and returns the output-operation result. No new input is processed.

The remaining-output amount is not recomputed from the recorded length after each call. A dependency changing the recorded length therefore does not directly change that local drain bound. A dependency changing recorded progress can affect both the subsequent progress update and the next output address.

No check rejects a positive output result larger than the requested amount. Subject to valid arithmetic, such a result can make the local remainder nonpositive and end the drain.

An initially nonpositive length-minus-progress difference skips this drain. Progress greater than length is not independently reported as an error.

### Empty or ineligible input

After the initial drain completes or is skipped, the wrapper clears output progress and then clears generated-output length.

New input is processed only if the input address is nonzero and the requested length is positive. Otherwise, the wrapper returns zero.

An empty or ineligible invocation can therefore initialize state, discard recorded state during initialization, deliver pending output, encounter an output failure, and modify flags and bookkeeping.

If initialization is unnecessary, such an invocation does not explicitly flush or clear retained input.

### Chunk and mode selection

For eligible input, the wrapper maintains a current input address and a signed remaining-input count. Each iteration initially selects the smaller of that count and 1024.

It rereads flag bit 0x100 each iteration:

- Clear: stateful processing.
- Set: three-byte grouping.

Dependency changes to the flags can affect a later iteration. The grouping path can replace or adjust the initially selected chunk size.

### Stateful processing

The wrapper calls F-38f4acd6ca2c with:

- State address +0x1c.
- Output-storage address +0x7c.
- Address of the generated-output-length field.
- Current input address.
- Selected chunk length.

The direct result is ignored. The generated-output-length field supplies the subsequent drain amount.

The selected chunk is treated as consumed regardless of the generated length. This path does not explicitly clear or consume the retained-input count.

### Grouping with no positive retained count

When the retained count is less than 1, behavior depends on the selected chunk length.

For one or two bytes, the wrapper invokes the unmapped transfer dependency with retained-input storage as destination, current input as source, and the selected length. It then records that length as the retained count and immediately returns the original requested length.

No output transformation or drain follows that transfer within the invocation. The transfer's direct result is not inspected.

For at least three bytes, the wrapper calls F-870c4fcf7295 with the selected length and 3. It subtracts the secondary register result of that call from the selected length.

The wrapper then calls F-ee799ab58b1f with output storage, current input, and the adjusted length. Its result is stored as generated-output length. The adjusted length is also the amount subsequently treated as consumed input.

The secondary result's arithmetic meaning is not supplied. Rounding the chunk down to a multiple of three is inferred intent, conditional on that result being a suitable remainder. The wrapper itself does not establish that interpretation or validate the adjusted length.

A negative retained count takes this same branch; it is not separately rejected.

### Grouping with retained input

When the retained count is at least 1, the wrapper selects the smaller of:

- Total input remaining in the invocation.
- Three minus the retained count.

It invokes the unmapped transfer dependency with the destination advanced by the retained count, the current input address, and the selected amount.

After the transfer, it rereads the retained count, adds the selected amount, and stores the sum. This reread matters if the transfer modifies state.

If the sum is less than 3, the wrapper immediately returns the original requested length.

Otherwise, it calls F-ee799ab58b1f with output storage and retained-input storage. Only those two arguments are exposed at this call site; an explicit length argument is not established.

The result is recorded as generated-output length, after which the wrapper clears the retained count.

Counts of 3 or greater are not repaired or rejected. They can cause zero or negative transfer lengths. Their resulting behavior is outside the ordinary grouping invariant and remains dependent on arithmetic and callee contracts.

### Consumption and delivery of newly generated output

After processing that does not return early, the wrapper:

1. Clears output progress.
2. Decreases local remaining input by the selected or adjusted consumption amount.
3. Advances the local input address by that amount.
4. Reads generated-output length into a local drain amount.

While that drain amount is positive, it calls F-8d1c9f620c59 using the current outer output argument, output storage advanced by recorded progress, and the local drain amount.

Positive results update progress using a post-call reread and reduce the local drain amount, as in the initial drain.

A zero or negative result causes a call to F-aeffcebde61f and an immediate return of the original requested input length. Under the established entry conditions, this length is positive. A represented alternative return of the output result for an original zero length is unreachable through this processing loop.

On failure, the wrapper does not perform normal post-drain clearing. Dependency effects aside, recorded output length and completed progress remain available for a later invocation. The input chunk has already been treated as consumed locally, but remaining chunks are not processed.

After a completed drain, or when generated length is initially nonpositive, the wrapper clears generated-output length and then output progress. Only then can another input iteration begin.

Normal completion returns the original requested input length.

## Side effects

Direct wrapper writes include flag clearing, conditional initialization records, generated-output lengths, progress updates, retained-input counts, and bookkeeping resets. Input and output storage are also passed to dependencies that may read or modify memory.

The wrapper has no explicit allocation or release operation. Passing an address does not establish ownership transfer, pointer retention, allocation, or release behavior by a dependency.

No dependency is assumed to preserve globals or shared memory. Observable distinctions include:

- The subordinate-state address is captured once.
- Flags are reread for each input iteration.
- The outer output argument and recorded output progress are read for each output request.
- Progress is reread after positive output results.
- Retained count is reread after an append transfer.
- Drain bounds and remaining-input counts are maintained locally rather than continuously reconstructed from state.

The wrapper does not expose an independent consumed-input count. Returning the requested length after a generated-output failure can conceal unprocessed later input.

## Dependencies

| Identifier | Established interaction and limits |
|---|---|
| F-45d85654c723 | Conditional initialization using state +0x1c; direct result unused. |
| F-8d1c9f620c59 | Output operation receiving an outer-context value, output address, and signed length. Positive results count as progress; nonpositive results trigger error handling. |
| F-aeffcebde61f | Receives the outer-context address following a nonpositive output result. Its cleanup, state, memory, and ownership effects are unknown. |
| F-38f4acd6ca2c | Stateful processing with a writable output-length address. Direct result unused. |
| F-870c4fcf7295 | Receives a chunk length and 3. A secondary register result adjusts input consumption; its contract is unknown. |
| F-ee799ab58b1f | Its result becomes generated-output length. Three arguments are exposed for direct input; two are exposed for retained input. |
| Opaque ID unavailable | Unmapped transfer dependency receiving destination, source, and byte count. Exact transfer, overlap, failure, and ownership semantics are documented_only. |

The unavailable identifier is not replaced with an invented function ID.

## Acceptance examples

These are conditional predictions from static evidence, not executed tests. Unless specified otherwise, assume valid storage, representable arithmetic, marker 1, and no dependency mutations beyond those described. Successful output operations are assumed only where explicitly stated.

1. **Empty invocation.** With output length and progress zero and requested length zero, the wrapper clears the low flag nibble, clears progress and length, and returns zero without processing input.

2. **Pending output with absent input.** With length 5, progress 2, and input address zero, the first output request uses output offset 2 and length 3. If it returns 3, bookkeeping is cleared and the wrapper returns zero.

3. **Initial-drain failure.** With pending output, an output result of −7 causes F-aeffcebde61f to be called and −7 to be returned. Otherwise eligible input is not processed.

4. **Conditional initialization.** With marker 0 and pending-output records, the wrapper sets the marker and clears output length, progress, and retained count before initialization. If F-45d85654c723 leaves those fields clear, the previously recorded output is not delivered.

5. **Negative requested length.** With a nonzero input address and length −1, entry preparation and any initial drain still occur. If that drain completes, the wrapper returns zero without processing new input.

6. **Retaining one byte.** In grouping mode with retained count zero and one input byte, the wrapper invokes the transfer dependency for one byte, records retained count 1, and returns 1 without generating output.

7. **Completing a retained group: continuation and failure.** In grouping mode with retained count 2 and four new bytes, the wrapper selects one byte for the append. If the transfer leaves the existing count unchanged, the wrapper records count 3, invokes F-ee799ab58b1f on retained storage, records its result, and clears the retained count. It then treats one new byte as consumed. Processing of the remaining three bytes occurs only if generated-output draining completes, or if the generated length is nonpositive and draining is skipped. If positive-length output is generated and any output attempt returns zero or a negative value, the wrapper calls F-aeffcebde61f and returns 4 without processing those remaining three bytes.

8. **Conditional three-byte rounding.** With grouping mode, no retained input, and 1024 input bytes, suppose F-870c4fcf7295 supplies secondary result 1. The first transformation is given length 1023. If its drain completes or is skipped, and grouping mode remains selected, the final byte is retained and the invocation returns 1024.

9. **Generated-output failure.** With ten eligible input bytes, if processing generates positive-length output and the first output attempt returns zero, the wrapper invokes F-aeffcebde61f and returns 10. The return does not prove delivery.

10. **Partial delivery.** For generated length 7, output results 2 and 5 produce requests for 7 bytes at offset 0 and then 5 bytes at offset 2. If dependencies leave progress unchanged except through wrapper updates, completion clears length and progress.

## Uncertainties

The following remain **documented_only**:

- Transformation semantics and the external meaning of mode bit 0x100.
- Complete structure sizes, capacities, alignment, and allocation lifetimes.
- F-870c4fcf7295's secondary-result contract.
- The effective argument contract for the retained-input call to F-ee799ab58b1f.
- The unmapped transfer dependency's identity and semantics.
- Dependency mutations, ownership effects, pointer retention, and release behavior.
- Error reporting beyond the visible returns and error-handler calls.
- Target behavior for signed overflow, invalid addresses, invalid retained counts, and oversized output results.

Termination requires additional dependency properties. In particular, adjusted input consumption must permit progress, and output results must permit drains to finish. These properties are not independently enforced.

## Evidence

The supplied static representation establishes field offsets, 32-bit operations, signed comparisons, direct write order, visible call arguments, the 1024-byte initial chunk cap, early returns, and the distinct initial-drain and generated-output failure returns.

Three-byte grouping is inferred from retained storage, comparisons with 3, and the arithmetic dependency call. Exact rounding and transformation contracts are not established.

The retained-group continuation rule follows from the generated-output drain occurring before the next input iteration: a nonpositive output result exits immediately, despite returning the full requested input length.

Dependency implementations and authoritative calling-convention details were not supplied. No runtime tests have been performed.
