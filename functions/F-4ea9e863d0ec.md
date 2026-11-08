<!-- Trusted provenance: F-4ea9e863d0ec; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4ea9e863d0ec

## Purpose

F-4ea9e863d0ec transforms a sequence of signed 16-bit samples into interleaved pairs of identical 16-bit values in storage embedded in a caller-supplied state object. It uses a persistent phase, a persistent increment, and a saved preceding sample to select and interpolate samples across calls.

The established behavior is linear interpolation followed by duplication into two output slots. Interpreting those slots as stereo channels, and interpreting the phase increment as a sample-rate ratio, are inferred intent rather than established interface guarantees.

The function reports the number of pairs written through a caller-provided count pointer. It returns no value. On a normal return, it also updates the saved sample and persistent phase.

There is no output-capacity argument, input validation, or explicit failure result. Termination and memory safety depend on the supplied state, sample count, storage capacity, and aliasing relationships.

## Interface

The four arguments are:

1. **State address.** Addresses an object containing the fields and output region described below.
2. **Input samples.** Addresses signed 16-bit samples.
3. **Input count.** A signed 32-bit integer used both in a boundary comparison and to locate the final input sample.
4. **Output-count address.** Addresses a writable signed 32-bit integer.

The supplied representation uses the following widths:

| Quantity | Width and interpretation |
|---|---|
| Input and saved samples | Signed 16-bit |
| Each output slot | 16-bit |
| Persistent phase | Unsigned 32-bit |
| Increment | Signed 32-bit when loaded; participates in unsigned phase addition |
| Input count and output count | Signed 32-bit |
| Interpolation arithmetic | Signed 32-bit, with unsigned 32-bit phase operations |

The state layout, expressed as byte offsets from its base, is:

| Offset | Access | Meaning |
|---|---|---|
| `0x24` | Read at entry; written on normal return | Persistent phase |
| `0x28` | Read at entry | Phase increment |
| `0x2C` | Read at entry; written on normal return | Saved preceding sample |
| `0x30` onward | Written sequentially | Pairs of identical 16-bit output values |

Each emitted pair occupies four bytes. The output pointer advances by four bytes per pair. The output count measures pairs, not individual 16-bit values or bytes.

No allocation, release, or ownership transfer is visible. The caller supplies all accessed storage. The function does not establish that the output region is large enough.

For ordinary sample-processing use, the caller must provide:

- A valid, suitably aligned state object.
- A valid writable output-count location.
- At least one readable input sample, even when no output will be emitted.
- A positive input count and storage covering every input sample actually accessed.
- Enough output storage for every pair emitted before termination.
- A phase and increment that cause the processing loop to reach its exit condition.
- Non-overlapping storage if the caller expects the input, state fields, and count to remain independent.

These are conditions for safe intended use, not validations performed by the function.

## Behavior

### Entry reads and initialization

The observable access sequence begins with:

1. Read the persistent phase.
2. Read the increment.
3. Store zero through the output-count address.
4. Read and sign-extend the saved preceding sample.
5. Read and sign-extend the first input sample.

The phase and increment are cached for the rest of the invocation. Subsequent writes to their state locations do not change those cached values.

The first input sample is read unconditionally. A zero input count, or a starting phase that leads directly to termination, does not suppress this read.

### Phase interpretation and interpolation

The phase is handled as an unsigned 32-bit value with a 16-bit integer part and a 16-bit fractional part. Phase additions and subtractions use 32-bit unsigned arithmetic, including wraparound.

For interpolation, a fractional position \(f\), between 0 and 65,535, produces the integer weight

\[
w=\left\lfloor f/2\right\rfloor.
\]

The resulting sample is

\[
y=a+\operatorname{ASR}_{15}\big((b-a)w\big),
\]

where \(a\) and \(b\) are sign-extended 16-bit endpoint samples and \(\operatorname{ASR}_{15}\) denotes an arithmetic right shift by 15 bits.

Under arithmetic-right-shift semantics, negative products round toward negative infinity. The least significant fractional phase bit is discarded, so adjacent fractional values can produce the same weight.

The weight ranges from 0 to 32,767. The sample difference ranges from −65,535 to 65,535, and the product magnitude is at most 2,147,385,345. Thus the interpolation multiplication fits signed 32-bit arithmetic for valid signed 16-bit endpoints.

### Initial interval: saved sample to first input sample

While the cached phase is less than 65,536, the function emits samples interpolated between:

- The saved preceding sample as the lower endpoint.
- The first input sample as the upper endpoint.

In this interval, the phase itself supplies the fractional position.

For each pair, the function:

1. Computes the interpolated value.
2. Reads an external scalar used by the upper-clamping branch.
3. Selects the value to store.
4. Writes the first 16-bit output slot.
5. Writes the identical value to the second 16-bit output slot.
6. Reads and increments the output count.
7. Adds the cached increment to the cached phase using unsigned 32-bit arithmetic.

The input endpoints remain cached throughout this initial interval.

When the phase is no longer below 65,536, the function subtracts 65,536 once. This happens even if the initial interval emitted nothing.

### Input intervals

After that subtraction, the high 16 bits of the working phase select an input interval index.

Before accessing a newly selected interval, the function tests whether the unsigned interval index is at least the unsigned value of the input count minus one. If so, it finishes the call without emitting a pair for that phase.

Otherwise, it reads two consecutive samples:

- The sample at the interval index.
- The sample immediately following it.

These samples remain cached while the interval index stays unchanged. If later phase arithmetic revisits an interval after selecting a different one, its endpoints are read again.

Within an interval, the low 16 bits of the phase supply the fractional position. Each emitted pair follows this sequence:

1. Compute the interpolated value.
2. Read the external upper-clamping scalar.
3. Write the first output slot.
4. Write the identical second slot.
5. Advance the output pointer by four bytes.
6. Add the cached increment to the working phase.
7. Read and increment the output count.

The next iteration recomputes the interval index and either reuses the cached endpoints, loads another pair, or exits.

### Value selection

The visible value-selection rule is:

- A computed value below −32,768 becomes −32,768.
- A computed value from −32,768 through 32,767 is retained.
- A computed value of at least 32,768 selects an external scalar, whose low 16 bits are stored.

The external scalar’s value is not supplied. It must not be assumed to be 32,767.

With ordinary signed 16-bit endpoints and the interpolation weights above, the computed result remains within the signed 16-bit range. Neither clamping branch changes such a result. Nevertheless, the external scalar is visibly read for every emitted pair.

### Normal termination and persistent state

On reaching the boundary condition, the function:

1. Reads the input sample at index “input count minus one.”
2. Stores that sample into the saved-sample field.
3. Computes the residual phase.
4. Stores the residual phase into the persistent-phase field.
5. Returns.

For a positive input count \(n\), working interval index \(j\), and fractional part \(f\), the boundary condition is \(j\ge n-1\). The stored phase is

\[
f+(j-n+1)\times65{,}536,
\]

using 32-bit arithmetic.

A boundary reached exactly at \(j=n-1\) leaves only the fractional part. Overshoot beyond that boundary is preserved as whole intervals.

The increment field is not explicitly written. The output count is not rewritten at termination; absent aliasing, its last incremented value is the reported total.

### Termination limitations

The routine contains no independent iteration limit.

A zero increment can cause indefinite output if the current phase is in an emitting region. A negative increment is added as an unsigned 32-bit quantity, not interpreted as a separate reverse-processing mode. Large increments can wrap the phase and skip a boundary.

The interval index can only range from 0 through 65,535. Consequently:

- A positive input count above 65,536 cannot satisfy the ordinary boundary comparison.
- A zero or negative input count produces a large unsigned boundary threshold that no interval index can reach.
- Positive counts within 1 through 65,536 do not alone guarantee termination; the phase trajectory still matters.

These cases can continue writing beyond available output storage. No error is reported first.

## Side effects

The function visibly performs:

- One initial zero store through the output-count address.
- Two sequential 16-bit output stores per emitted pair.
- One output-count read and increment per emitted pair.
- One saved-sample store on normal return.
- One persistent-phase store on normal return.
- Input reads at entry, on interval changes, and at termination.
- One external scalar read per emitted pair.

It does not visibly clear unused output storage or modify input samples through the input pointer.

Aliasing can nevertheless change behavior. For example:

- The initial count reset can alter the saved sample or first input sample before either is read.
- Output stores can overwrite input samples that have not yet been read.
- If the count aliases output storage, a count increment reads the value remaining after the pair’s stores.
- The final saved-sample or phase store can overwrite the count when their locations overlap.

Therefore, the ordinary count and interpolation guarantees require independent storage. The access ordering above, rather than a non-aliasing guarantee, describes overlapping cases.

The signed output counter has no overflow check. Invocations requiring more than the signed 32-bit maximum number of pairs are outside a well-defined signed-arithmetic contract.

## Dependencies

No called function is visible in the supplied body. There are therefore no callee ownership, release, or global-preservation guarantees to rely on.

One externally stored scalar is read during value selection. Its initialization, mutability, width beyond the supplied scalar representation, and intended value are not established. No opaque function ID was supplied for that data dependency.

The behavior also depends on:

- The target’s signed 16-bit and 32-bit integer representation.
- Arithmetic right shifts for negative interpolation products.
- Valid addressing and alignment for the stated accesses.
- Sufficient caller-provided storage.

## Acceptance examples

These are conditional expectations derived from static evidence, not executed tests. They assume independent valid storage, sufficient output capacity, and the arithmetic semantics described above.

### Saved-sample transition followed by an input interval

Given:

- Starting phase: 0.
- Increment: 65,536.
- Saved sample: 100.
- Input samples: 300, 500.
- Input count: 2.

Expected behavior:

- Emit the pair `(100, 100)` in the initial interval.
- Emit the pair `(300, 300)` in the first input interval.
- Report two pairs.
- Save 500 as the preceding sample.
- Store phase 0.

The final input sample is retained for continuity; this call does not emit it at its boundary.

### Halfway through the initial interval

Given phase 32,768, increment 65,536, saved sample 0, and one input sample of 1,000:

- Emit `(500, 500)`.
- Report one pair.
- Save 1,000.
- Store phase 32,768.

### Negative interpolation rounding

Given phase 2, increment 65,536, saved sample 0, and one input sample of −1:

- The interpolation weight is 1.
- Arithmetic shifting of the negative product yields −1.
- Emit `(-1, -1)`, not `(0, 0)`.
- Report one pair and store phase 2.

This example depends on arithmetic-right-shift semantics.

### Overshoot without output

Given starting phase 262,144 and an input count of 2:

- Emit no pairs.
- Report zero.
- Save the second input sample.
- Store phase 131,072.

The first input sample is still read at entry. The increment does not affect this path.

### Zero increment in the initial interval

Given starting phase 0 and increment 0:

- Repeatedly emit the saved sample in identical pairs.
- Never reach the boundary through phase advancement.
- Eventually exceed finite output capacity unless execution is externally interrupted.

This is a nontermination hazard, not an accepted finite conversion.

## Uncertainties

**documented_only — external scalar.** Its value and intended saturation role are unknown. Valid interpolation does not require that value, but its reads remain visible.

**documented_only — target semantics.** The supplied representation supports the stated 16-bit and 32-bit model. No independent ABI evidence establishes endianness, alignment tolerance, or signed-shift behavior.

**documented_only — state and buffer extent.** No output capacity, complete object definition, or permitted overlap policy is supplied.

**documented_only — supported phase range.** No caller evidence establishes a narrower increment range or a termination invariant.

**Inferred intent.** The layout and arithmetic suggest streaming fixed-point resampling with duplicated output channels. Audio format, sample rate, and channel semantics are not established.

**Not established.** Thread safety, reentrancy under shared state, recovery from invalid memory, and external mutation behavior have no supplied contract.

## Evidence

This contract is based solely on the supplied function body: its field offsets, integer operations, branches, memory accesses, and loop structure.

Static evidence establishes the two interpolation phases, duplicate output stores, count updates, boundary comparison, saved-sample update, and residual-phase calculation. The width analysis establishes that valid interpolation products fit signed 32-bit arithmetic.

No runtime tests, execution traces, caller analysis, external-scalar inspection, or independent layout verification have been performed. Acceptance examples are static deductions, not observed results.
