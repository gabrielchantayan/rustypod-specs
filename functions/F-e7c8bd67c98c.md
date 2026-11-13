<!-- Trusted provenance: F-e7c8bd67c98c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e7c8bd67c98c

## Purpose

F-e7c8bd67c98c performs an ordered sequence of indirect dependency invocations, byte accesses, and 32-bit memory accesses. Its arguments and stored values depend on explicit inputs, additional incoming execution state, dependency results, and memory reads.

The represented flow ends at an invalid-instruction termination marker, not a normal return. No application-level purpose, protocol, or object type is established. Packed-field or address manipulation is **inferred intent only**.

This contract separates:

- **Established static evidence:** access widths, represented ordering, argument relationships, masks, shifts, and consumed results.
- **Conditional arithmetic consequences:** behavior requiring a 32-bit machine interpretation of signed calculations.
- **documented_only unknowns:** matters not resolved by the supplied representation.

All stages below are conditional on preceding accesses and dependencies completing and execution continuing along the represented flow.

## Interface

### Explicit inputs

| Input | Width | Established use |
|---|---:|---|
| First | 32 bits | Supplied to F-d01-01. |
| Second | 32 bits | No explicit use represented. |
| Third | 32 bits | No explicit use represented. |
| Fourth | Signed 32-bit quantity | Shifted left by 10; the resulting pattern is supplied twice to F-d02-03 and subsequently stored. |

Unused explicit inputs are not proven unobservable to dependencies through unspecified calling state.

### Additional incoming state

The function also requires:

- An incoming address-like value for the first byte read at offset 1.
- An incoming byte-sized value for the first byte write.
- Incoming control-transfer state from which the first indirect target is derived.
- Executable indirect-call targets.
- Readable and writable memory at the computed addresses.

The representation does not initialize or validate these prerequisites.

### Outputs and completion

No normal return value is represented. Dependency results are consumed as 32-bit quantities, except for one 64-bit result split into upper and lower 32-bit components.

After the final dependency invocation, execution reaches the represented invalid-instruction marker. Its actual runtime outcome—trap, exception, termination, or another environment-specific effect—is **documented_only**.

### Widths and address terminology

The contract uses these terms:

- **Quad-scaled value:** an index multiplied by 4.
- **Sixteen-scaled value:** an index multiplied by 16.
- **Word slot at an address:** a 32-bit access beginning 20 bytes beyond that address.
- **Low byte:** the least significant eight bits.

These terms describe address arithmetic, not an established array or object layout. No implicit object base is present.

Unsigned 32-bit calculations retain the low 32 bits; unsigned right shifts introduce zeros. Several index calculations, address calculations, and left shifts instead involve represented signed 32-bit quantities. Their overflow and signed-shift semantics are not independently established.

Where stated, the **conditional machine model** treats these signed operations as producing their low 32-bit machine patterns. This is not a claim that the supplied representation independently proves those semantics.

## Behavior

### Dependency naming and sequencing

Dependency IDs identify invocation positions, not verified distinct functions. For example, F-d03-04 identifies the fourth invocation in stage 3. Different IDs may resolve to the same implementation.

“No explicit arguments” means none are displayed; it does not establish an empty machine-level input state.

Within each stage, operations occur in the order described, except where read-expression ordering is explicitly unresolved. A dependency may modify memory or other shared state. No preservation guarantee is assumed.

### 1. Initial byte transfer

Read one unsigned byte from the incoming address-like value plus 1. Retain this as the **initial selector**.

Write the incoming byte-sized value at address 17 plus the initial selector. The destination therefore lies from 17 through 272 inclusive.

Invoke, in order:

- F-d01-01 with the first explicit input.
- F-d01-02, F-d01-03, and F-d01-04 without explicit arguments.
- F-d01-05 without explicit arguments; retain its 32-bit result as the **initial index**.

The initial selector is retained from the first read, not reread before its stage-2 use.

### 2. Input scaling and split result

Invoke F-d02-01 with the zero-extended initial selector and the initial index shifted left by two. Then invoke F-d02-02 without explicit arguments.

Shift the fourth input left by 10, producing the **scaled input**. Invoke F-d02-03 with that same pattern in both argument positions.

Invoke F-d02-04 without explicit arguments. Split its represented 64-bit result numerically:

- Upper 32 bits: **upper component**.
- Lower 32 bits: **lower component**.

This establishes no in-memory ordering of the two components.

Write the scaled input to the word slot at the quad-scaled upper component. Then invoke F-d02-05 with the upper component itself, not its scaled address.

### 3. Upper-component propagation

Invoke F-d03-01 through F-d03-04 without explicit arguments. The last result supplies an index.

Write the upper component to the word slot at that index’s quad-scaled value. Invoke F-d03-05 with the quad-scaled value.

Then invoke F-d03-06 through F-d03-09 without explicit arguments. Their results are not explicitly consumed.

### 4. Lower-component transformation

Write the low 32 bits of the lower component multiplied by \(2^{26}\) to address:

**20 + low 32 bits of (lower component × \(2^{28}\))**

The address multiplier and stored-value multiplier differ. These multiplications use unsigned 32-bit arithmetic.

Mask the lower component with `0x1fffffff`, retaining the masked quantity before the following dependency sequence.

Invoke F-d04-01 with the value just written. Invoke F-d04-02 through F-d04-05 without explicit arguments. The last result supplies the next index.

### 5. Masked-component propagation

Logically shift the retained masked quantity right by nine. Write the result to the word slot at the stage-4 index’s quad-scaled value.

Invoke F-d05-01 with that quad-scaled value. Invoke F-d05-02 through F-d05-05 without explicit arguments. The last result supplies a new index.

### 6. Separate store and argument-read locations

Write the stage-5 index’s quad-scaled value to the word slot at that index’s sixteen-scaled value.

Invoke F-d06-01 with:

1. An unsigned 32-bit value read from the word slot at the quad-scaled value, logically shifted right by 12.
2. The quad-scaled value.
3. An unsigned 32-bit value read from that same word slot, shifted left by two.

Two read expressions are represented. Their relative evaluation order and whether they correspond to two separate machine reads are unresolved. Equality of the two observed values is not guaranteed.

The preceding store uses a sixteen-scaled location, whereas the argument reads use a quad-scaled location. Substitution of the stored value for either read is generally invalid.

Invoke F-d06-02 through F-d06-05 without explicit arguments. The last result supplies a new index.

### 7. Address propagation

Write the quad-scaled value retained from stage 6 to the word slot at the new index’s quad-scaled value.

Invoke F-d07-01 with the destination’s quad-scaled value. Invoke F-d07-02 through F-d07-05 without explicit arguments. The last result supplies another index.

### 8. Lower-component argument reuse

Write the stage-7 index’s quad-scaled value to its sixteen-scaled word slot.

Invoke F-d08-01 with:

1. That quad-scaled value.
2. The same value again.
3. The lower component shifted left by 26.

The third argument uses unsigned 32-bit arithmetic.

Invoke F-d08-02 through F-d08-05 without explicit arguments. The last result supplies a new index.

### 9. Address propagation

Write the quad-scaled value retained from stage 8 to the word slot at the new index’s quad-scaled value.

Invoke F-d09-01 with the destination’s quad-scaled value. Invoke F-d09-02 through F-d09-05 without explicit arguments. The last result supplies a new index.

### 10. Self-address store

Write the stage-9 index’s quad-scaled value to its sixteen-scaled word slot.

Invoke F-d10-01 with the quad-scaled value. Invoke F-d10-02 through F-d10-05 without explicit arguments. The last result supplies a new index.

### 11. Address propagation

Write the quad-scaled value retained from stage 10 to the word slot at the new index’s quad-scaled value.

Invoke F-d11-01 with the destination’s quad-scaled value. Invoke F-d11-02 through F-d11-05 without explicit arguments. The last result supplies a new index.

### 12. Self-address store and sampling

Write the stage-11 index’s quad-scaled value to its sixteen-scaled word slot.

Next, read an unsigned 32-bit value from its quad-scaled word slot. Retain this as the **sampled word**.

Invoke F-d12-01 with the quad-scaled value. Invoke F-d12-02 through F-d12-05 without explicit arguments. The last result supplies the **retained index**.

The sampled word is captured before F-d12-01. Subsequent dependency effects do not cause a represented reread.

### 13. Sampled-word extraction

Logically shift the sampled word right by 12. Write the result to the word slot at the retained index’s quad-scaled value.

Invoke F-d13-01 with that quad-scaled value. Invoke F-d13-02 through F-d13-05 without explicit arguments. The last result supplies a new index.

### 14. Retained-index bit arguments

Write the stage-13 index’s quad-scaled value to its sixteen-scaled word slot.

Invoke F-d14-01 with:

1. The retained index’s quad-scaled value, masked with `0x3f`, then shifted left by 14.
2. The stage-13 index’s quad-scaled value.
3. The retained index shifted left by 30.

Under the conditional machine model, the first argument can occupy only bit positions 16 through 19. The third depends only on the retained index’s bottom two bits.

Invoke F-d14-02 through F-d14-05 without explicit arguments. The last result supplies a new index.

### 15. Final payload acquisition

Write the quad-scaled value retained from stage 14 to the word slot at the new index’s quad-scaled value.

Invoke F-d15-01 with the destination’s quad-scaled value. Invoke F-d15-02 through F-d15-05 without explicit arguments. Retain the last result as the unsigned 32-bit **final payload**.

### 16. First late byte write

Read an unsigned selector byte from the address formed by multiplying the retained index by \(2^{28}\), then adding 1.

Under the conditional machine model, this address is:

**1 + low 32 bits of (retained index × \(2^{28}\))**

Write the low byte of the final payload shifted left by two to address 17 plus this selector.

Invoke F-d16-01 with the final payload shifted left by two in both the first and second argument positions, and the final payload shifted left by four in the third position.

Invoke F-d16-02 without explicit arguments. Pass its 32-bit result to F-d16-03.

Invoke F-d16-04 through F-d16-07 without explicit arguments. The last result supplies another index.

### 17. Second late byte write

Read the selector byte again using the same address expression as stage 16. This is a fresh read after the intervening dependencies.

Mask the final payload with `0x3fffffff`, logically shift right by 20, and write the result’s low byte to address 17 plus the newly read selector.

The written byte contains original payload bits 20 through 27. Bits 28 and 29 survive the shift but are discarded by the byte store.

Invoke F-d17-01 with the stage-16 index shifted left by two. Invoke F-d17-02 through F-d17-05 without explicit arguments. The last result supplies another index.

### 18. Terminal invocations

Invoke F-d18-01 with:

1. The stage-17 result shifted left by two.
2. The selector from the second late byte read, zero-extended to 32 bits.

Invoke F-d18-02 without explicit arguments. The represented flow then reaches the invalid-instruction termination marker.

## Side effects

The direct memory effects represented are:

| Access category | Count and location |
|---|---|
| Byte writes | Three: stages 1, 16, and 17. |
| 32-bit writes | **Fourteen: exactly one in each stage from 2 through 15 inclusive.** |
| Byte reads | Three: the initial selector and two late selector reads. |
| 32-bit read expressions | Three: two in stage 6 and one in stage 12. |

Stages 1 and 16–18 contain no direct 32-bit writes. The stage-6 count describes read expressions, not independently verified machine-load events.

All described indirect invocations are additional observable operations. Their effects are not included in the direct-access counts.

Computed addresses may alias, including across access widths. Successive stores need not affect distinct storage. Dependencies may alter memory subsequently read or invalidate storage; this contract assumes no global-memory preservation.

No allocation, deallocation, ownership transfer, locking, rollback, or restoration is directly represented. This does not exclude such actions within dependencies.

There is no represented validation or recovery path. If an access fails or a dependency does not return, later operations are not established as occurring, and earlier effects are not shown as being reversed.

## Dependencies

The stage descriptions define the invocation order, explicit arguments, and consumed results. Unconsumed results are discarded by the represented caller, not proven absent.

The first target depends on incoming control-transfer state. Later targets are formed from represented target-producing values. Most use an added offset of 4094 bytes; F-d16-01 uses 4076 bytes, 18 bytes less. This difference is established, but its meaning and executability are unknown.

For every dependency, these matters remain **documented_only**:

- Actual identity, implementation, and possible target aliasing.
- Calling convention and hidden machine-state inputs.
- Global-state and memory effects.
- Allocation, release, and ownership behavior.
- Failure, exception, blocking, and nonreturning behavior.
- Whether the represented result width matches the actual implementation’s interface.

No dependency is assumed to preserve globals, own referenced memory, or release it.

## Acceptance examples

These are conditional static expectations, not executed tests. They assume valid accesses, executable targets, dependencies returning into the represented continuation, and any arithmetic conditions stated.

1. **Initial selector boundary.** An initial selector of 255 causes the first byte write at address 272. No represented range check intervenes.

2. **Fourth-input scaling.** A fourth input of 1 produces 1024. F-d02-03 receives 1024 twice. With an upper component of 3, stage 2 writes 1024 at address 32.

3. **Signed-shift limitation.** Under the conditional machine model, a fourth-input pattern of `0x00400000` produces a zero scaled input. Without that model, the signed left shift is unresolved; a portable arithmetic result is not established.

4. **Lower-component arithmetic.** A lower component of `0x12345678` causes stage 4 to write `0xe0000000` at address `0x80000014`. Stage 5 later writes `0x00091a2b` to its separately determined destination.

5. **Different store and read locations.** A stage-5 index of 2 causes stage 6 to write 8 at address 52. Its argument reads instead use address 28. That store alone establishes neither read value.

6. **Retained sample.** If stage 12 samples `0x12345000`, stage 13 writes `0x00012345`. A dependency subsequently changing the sampled memory does not alter this retained value.

7. **Retained-index fields.** Under the conditional machine model, a retained index of 19 makes F-d14-01’s first argument `0x00030000` and third argument `0xc0000000`. Its second argument still comes from the separate stage-13 result.

8. **Fresh late selector.** If stage 16 reads selector 4 and stage 17 reads selector 9, their byte writes target addresses 21 and 26. Reusing the first selector for the second write would violate the represented sequence.

9. **Late payload bytes.** A final payload of `0x0ab00003` produces byte `0x0c` in stage 16 and byte `0xab` in stage 17. The destination addresses need not match.

10. **Direct write accounting.** If the entire represented sequence is reached, it contains fourteen direct 32-bit writes and three direct byte writes, irrespective of additional writes performed by dependencies.

11. **Interrupted execution.** If F-d12-01 does not return, the stage-12 store and sample precede that interruption; stages 13–18 are not reached through the represented sequence. No compensating operation is established.

## Uncertainties

- **Control-flow completeness — documented_only.** The supplied representation reports invalid instruction data and truncated flow. A valid, complete executable path is not independently established.
- **Signed arithmetic — documented_only.** Overflowing signed multiplication, signed address addition, and signed left shifts require instruction-level evidence.
- **Memory layout — documented_only.** Endianness, alignment, address validity, and whether accesses reach ordinary memory or device state are unknown.
- **Read observability — documented_only.** Stage 6 does not establish relative machine-read order, distinct load events, or protection against external changes.
- **ABI and hidden state — documented_only.** Calling conventions, additional inputs, and actual dependency interfaces are unverified.
- **Ownership — documented_only.** No ownership or lifetime guarantee is established for referenced storage.
- **Application meaning — inferred intent only.** Scaling and bit extraction suggest packed-value manipulation but establish no named format.
- **Termination — documented_only.** The final marker establishes neither a normal return nor a particular runtime failure mechanism.

## Evidence

This contract rests solely on static inspection of the supplied representation. The prior draft, reviewer feedback, and embedded warnings are not treated as independent proof of behavior.

The direct 32-bit write count follows from one represented write in each stage 2–15: fourteen writes total. The three byte writes occur separately in stages 1, 16, and 17.

Established observations include represented access widths, offsets, unsigned transformations, dependency argument relationships, consumed results, and operation ordering. Consequences requiring a particular interpretation of signed arithmetic are explicitly conditional.

No runtime tests, executable validation, dependency inspection, or independent instruction-level verification have been performed.
