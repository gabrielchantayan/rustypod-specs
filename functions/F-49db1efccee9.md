<!-- Trusted provenance: F-49db1efccee9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-49db1efccee9

## Purpose

F-49db1efccee9 performs state-dependent memory operations and then enters an unconditional repeating dispatch sequence. The supplied static representation does not establish a higher-level purpose.

The routine has one explicit unsigned 32-bit input. Its behavior also depends on incoming address values, other machine words, and memory. The input determines whether an initial halfword transfer is attempted; it does **not** initialize the terminal dispatch base. On the ordinary bypass path, that base comes from the incoming link-like address.

There is no represented normal return. Each completed dispatch iteration consists of two indirect calls followed by software interrupt `0x5b`. Execution repeats if both calls return and the interrupt resumes.

This contract distinguishes:

- **Established static behavior:** represented operations, control dependencies, widths, and ordering.
- **Derived consequences:** conclusions obtained from those operations under the represented 32-bit arithmetic model.
- **documented_only:** matters requiring unavailable instruction, ABI, memory-map, or dependency information.

No runtime tests have been performed.

## Interface

### Explicit input

One unsigned 32-bit integer.

Only its low 16 bits contribute to the displayed calculations. These calculations include multiplication by `0x10000`, multiplication by `0x20000`, a test of bit 14, and multiplication of the low halfword by `0x8000`.

There is no explicit buffer length, capacity, output parameter, ownership token, or error destination.

### Implicit inputs

The represented calculations depend on:

| Incoming state | Role |
|---|---|
| Link-like address | Initial dispatch base; also used by a conditional halfword access |
| Halfword-source address | Source of the initial transfer |
| Transfer-offset word | Byte displacement for the initial destination; also used in comparisons and arithmetic |
| Source-adjustment word | Subtracted from the initial destination to update the working source; also used in comparisons |
| Comparison word | Compared with the transfer-offset word |
| Addition word | Used by the first addition-with-carry stage |
| Numerical address of an incoming stack location | Used in address calculations and arithmetic |
| Referenced memory | Supplies transferred values, derived addresses, and conditional stored values |

The integer words and address bit patterns are represented as 32 bits. These dependencies do not establish a conventional calling convention or additional formal parameters.

A separately assigned machine-state location is **not an independent incoming dependency** of the later arithmetic. Every displayed path to that arithmetic first assigns the location from the numerical stack-location address multiplied by 64. The assignment dominates its later read, although its architectural meaning remains unknown.

### Widths and memory layout

The representation uses unsigned 16-bit loads and stores, signed 16-bit loads with 32-bit sign extension, 32-bit stores, and a signed 8-bit conversion for one address lookup.

Unless qualified otherwise:

- Integer and address arithmetic retains the low 32 bits.
- A halfword store retains the low 16 bits.
- Halfword-address element increments represent two-byte displacements.
- Explicit integer address displacements are measured in bytes.
- Numerical right shifts of unsigned address representations are logical shifts.

These are properties of the apparent machine-width model, not guarantees that the expressions constitute valid portable pointer arithmetic.

Endianness, alignment requirements, address validity, and memory-mapped access semantics are not established.

### Outputs and ownership

No explicit return value is represented.

Possible outputs include memory writes, changes to working machine state, indirect calls, and a software interrupt. There is no identifiable allocation, release, or explicit transfer of memory ownership in the routine itself.

The callees and interrupt handler may have additional effects. Their ownership and preservation behavior is unknown.

## Behavior

### Initial partition and transfer

The routine forms a scaled address value from the input multiplied by `0x10000`, retaining 32 bits. That value is zero exactly when the input’s low 16 bits are zero.

**Nonzero low halfword:** the initial transfer and all subsequent gated memory-processing blocks are bypassed. The terminal dispatch base is the **incoming link-like address**, not the scaled input value. The first indirect-call target is therefore that incoming address plus `0xeb6`, with 32-bit address arithmetic.

**Zero low halfword:** the routine prepares:

- A stack-derived address equal to the incoming link-like address plus the numerical stack-location address shifted right by one. The displayed bit-14 contribution is zero on this branch.
- A doubled scaled value from the input multiplied by `0x20000`; this is also zero on this branch.
- A transfer destination equal to the incoming source address plus the transfer-offset word, in bytes.

The initial transfer has this represented order:

1. Read an unsigned halfword from the incoming source address.
2. Update the working source address to the transfer destination minus the source-adjustment word.
3. Store the previously read halfword at the transfer destination.

Thus the working-source update occurs **after the read and before the store**. The write does not reread the source. Aliasing is permitted by the representation; there is no bounds or overlap check.

Further gated processing requires the incoming link-like address to be zero. Otherwise, dispatch uses that incoming address after the transfer.

### First gate sequence

The following gates execute consecutively only while each preceding result is zero. A failed gate disables later conditional processing; it is not an error report.

The first sequence requires:

1. The stack-derived address to be zero.
2. Equality of the initial scaled value and the stack-derived address.
3. Equality of the initial scaled value and the doubled scaled value.
4. The same equality again.
5. Equality of the transfer-offset word and comparison word.
6. Equality of the source-adjustment word and updated working source address.

The represented exclusive-or gates obtain their shift counts from the low byte of a preceding zero result. Their executed shift counts are consequently zero. They act as equality checks without changing the propagated carry at those points.

On entry to this sequence, the incoming link-like address and both scaled values are zero. Consequently, the first gate requires the numerical stack-location address shifted right by one to be zero.

### Addition gates

If the first sequence passes, the routine adds the addition word, the input’s low halfword multiplied by `0x8000`, and the propagated carry. The result must be zero for processing to continue.

Its carry is the unsigned overflow from the first pairwise addition or the subsequent carry addition. Signed overflow is not a substitute.

The next stage adds:

- The numerical stack-location address.
- Twice the transfer-offset word.
- The preceding unsigned carry.

Its carry is calculated from those unsigned 32-bit additions. Its result must also be zero.

On paths reaching these additions, the input contribution and incoming carry are zero. The first zero-result addition therefore requires the addition word to be zero and produces no carry.

### Conditional link-derived mutation

If both addition results are zero, the routine:

- Assigns the internal machine-state location from the numerical stack-location address multiplied by 64.
- Forms an advanced link-derived address using the same byte displacement.
- Performs a halfword-derived-address store through the **original** link-like location.
- Advances the working scaled halfword address by one element, or two bytes.
- Clears the propagated carry.
- Adopts the advanced link-derived address as the current link-like address.

For the halfword-derived-address store, an unsigned halfword read through the original link-like address supplies the numerical destination address. The stored halfword is the value read through that original location plus 15, truncated to 16 bits.

The representation references that source location more than once within the store expression. It does not establish the physical load count or relative evaluation order of those repeated reads. If the reads observe different values, the destination-defining value and payload-defining value need not match. At least the destination-defining read must precede the store.

The assignment to the internal machine-state location and formation of the advanced address precede this store. The two-byte working-address advancement and adoption of the advanced link-like address follow it.

### Second gate sequence and reachability

Further processing requires the advanced link-derived address to be zero. The second sequence then checks:

1. The stack-derived address against that zero address.
2. The advanced working scaled address against the stack-derived address.
3. The initial scaled value against the doubled scaled value, twice.
4. The transfer-offset word against the comparison word.
5. The source-adjustment word against the updated working source address.

Again, the executed shift counts are zero.

There is a decisive static contradiction under the represented arithmetic:

- Reaching this sequence requires the zero-low-halfword branch, so the initial scaled value is zero.
- The mutation advances its working copy to numerical address `2`.
- Passing the first gate requires the stack-derived address to be zero.
- The second gate then compares `2` with `0` and fails.

Consequently, the later subtraction, lookup, record-write, and final-write regions cannot execute under this literal control-flow and arithmetic model. Their represented behavior is retained below as conditional documentation; these descriptions do not assert a feasible execution path. Whether the representation accurately describes executable instructions remains **documented_only**.

### Represented subtraction stages

If the second sequence were satisfied, the routine would add the stack-derived address, current carry, and `0x085740e8`. It would then left-shift the result by its own low eight bits.

Shift counts of 32 or greater have **documented_only** behavior. No instruction-specific masking or large-count rule is supplied.

Three zero-result subtraction gates follow:

- Doubled scaled value minus the shifted result and the complement of the incoming Boolean.
- Initial scaled value minus the updated working source address and the complement of the propagated Boolean.
- Initial scaled value minus the doubled scaled value shifted right by one and the complement of the propagated Boolean.

Each Boolean complement contributes either zero or one. Each subtraction retains 32 bits.

The associated Boolean update explicitly compares the subtrahend with the minuend and also compares their wrapping difference with the incoming Boolean. The comparison direction and use of the complemented Boolean must not be replaced by an assumed conventional subtract-with-borrow rule.

The third stage advances a separate working halfword address by 18 elements, or 36 bytes, before forming its subtraction result. That advancement belongs to the stage even if its zero-result test fails.

### Represented lookup and record write

If the subtraction gates were satisfied, the routine would increment the transfer-offset word by 8 and the source-adjustment word by 10. It would form a link-derived address 13 bytes beyond the previous link-like address.

A halfword from the previous link-like address is converted to signed 8-bit form. Adding 4 produces the numerical address for a 32-bit address load. Under that signed conversion, the address before 32-bit interpretation ranges from −124 through 131. It is not an offset relative to the link-like address.

The following arithmetic gate uses the previously assigned internal machine-state value shifted logically right by one, the advanced link-derived address, `0x042ba0a0`, and two complemented-Boolean terms. Those Boolean terms cancel modulo \(2^{32}\). The zero condition is therefore equality between the shifted internal value and the wrapping sum of the advanced link-derived address and `0x042ba0a0`.

If that gate were zero, the record-write block has this represented order:

1. Advance a working read-address copy by **one byte**.
2. Read a halfword through the unchanged stack-derived address and sign-extend it from 16 to 32 bits.
3. Set the future dispatch base to the advanced link-derived address minus the advanced working scaled address.
4. Store the sign-extended value at the advanced link-derived address.
5. Store the previously loaded 32-bit address four bytes later.

The one-byte advancement precedes the read; it does not change that read’s address. The dispatch-base update precedes both stores. The two stores occupy adjacent four-byte regions, if usable, and the first precedes the second.

### Represented final write

Further gates would require:

- The sign-extended halfword to be zero.
- The loaded 32-bit address to be zero.
- Equality of the advanced scaled address and one-byte-advanced read address.
- Equality of the initial and doubled scaled values.
- Equality of the initial scaled value and the separately 36-byte-advanced address.
- Equality of the updated transfer-offset and comparison words.
- Equality of the updated source-adjustment word and working source address.

If all were satisfied, the final block would:

1. Form a read address from the current dispatch base plus 13 bytes.
2. Advance the dispatch base by 27 bytes.
3. Load a signed halfword from the previously formed read address.
4. Sign-extend it to 32 bits.
5. Store it at its own numerical value minus `0x08574200`.
6. Store the previously loaded 32-bit address four bytes beyond that destination.

The base advancement precedes the load. The load still uses the address formed before that advancement.

### Terminal dispatch

Every path reaching the terminal region enters an unconditional loop:

1. Invoke F-unknown-01 at the current dispatch base plus `0xeb6`, passing the 32-bit patterns `0`, `0`, and `0xffffffa5`.
2. If it returns, invoke F-unknown-02 at `0x0857a94b`.
3. If it returns, invoke F-unknown-03 through software interrupt selector `0x5b`.
4. If execution resumes, set the next dispatch base to `0x083d4cc2` and repeat.

The next first-call target is consequently `0x083d5b78`.

The first call’s third argument is established as a bit pattern. Signed interpretation as −91 requires an unavailable callee signature. The second call has no explicit arguments shown; that does not establish its actual ABI signature.

## Side effects

Memory operations are not transactional. Earlier writes remain represented effects even if a later access, call, or interrupt prevents progress.

The reachable memory-operation order is the initial halfword transfer, followed conditionally by the halfword-derived-address store. The subsequent lookup and paired 32-bit stores are present in the representation but blocked by the gate contradiction described above.

Working-state ordering is part of the static description, particularly:

- Initial source adjustment between transfer read and transfer write.
- Record read-address advancement before its load, and dispatch-base update before its stores.
- Final dispatch-base advancement before its load.

This does not establish physical register allocation, instruction scheduling, or architectural state visibility after a fault.

There is no explicit validation, error return, exception handler, recovery path, or termination condition. Fault delivery for invalid or misaligned accesses is **documented_only**.

## Dependencies

The supplied identity map identifies only F-49db1efccee9. The following are opaque documentary dependency IDs, not recovered function identities.

| Dependency | Invocation | Established interface |
|---|---|---|
| F-unknown-01 | Dispatch base plus `0xeb6` | Three explicit 32-bit argument patterns |
| F-unknown-02 | Address `0x0857a94b` | No explicit arguments shown |
| F-unknown-03 | Software interrupt `0x5b` | Trap interface unknown |

No dependency is assumed to preserve globals or other machine state, own or release memory, or return normally.

## Acceptance examples

These are conditional static cases, not executed tests.

### Nonzero low halfword

For input `1`, the scaled value is `0x00010000`, but it is not the dispatch base.

If the incoming link-like address is `0x10000000`, the first indirect-call target is `0x10000eb6`. No conditional memory-processing block executes.

Input `1` alone does not determine a numerical target. Inputs `1` and `0x12340001` make the same explicit-input contribution.

### Initial transfer followed by dispatch

For an input whose low halfword is zero and a nonzero incoming link-like address:

- Read the source halfword.
- Update the working source to destination minus source-adjustment word.
- Write the captured halfword to source plus transfer-offset word.
- Skip subsequent gated processing.
- Dispatch first to incoming link-like address plus `0xeb6`.

All addresses must be usable for execution to progress.

### Aliased transfer

With transfer offset zero, the read and write address coincide. The represented behavior remains a read, an intervening working-source update, and a write of the captured value. Equal ordinary-memory contents do not justify deleting observable accesses.

### First deep gate fails

With zero low input halfword, zero incoming link-like address, and a stack-location address whose right shift by one is nonzero, the initial transfer occurs but the first deep gate fails. The first dispatch target is `0x00000eb6`, if reached.

### Mutation succeeds but later processing stops

Suppose the numerical stack-location address, incoming link-like address, comparison word, transfer-offset word, source-adjustment word, source address, and addition word are all zero. Assume accesses at numerical address zero succeed.

The initial transfer and mutation can execute. The mutation’s working scaled address becomes `2`. The next sequence first accepts a zero stack-derived address, then rejects equality between `2` and `0`. No subsequent lookup or 32-bit record write occurs.

### Resumed loop

If both calls return and interrupt `0x5b` resumes, the next first-call target is `0x083d5b78`, with the same explicit argument patterns. The two-call-then-interrupt sequence repeats without a represented normal-return transition.

## Uncertainties

The following remain **documented_only**:

- Actual ABI, function boundary, and architectural interpretation of incoming state.
- Correspondence between the displayed regions and executable instructions.
- Address validity, alignment, endianness, and memory-mapped behavior.
- Physical multiplicity and ordering of repeated source reads within one expression.
- Large-count left-shift semantics.
- Architectural meaning of the internally assigned machine-state location.
- Callee signatures, preservation rules, ownership effects, and return behavior.
- Interrupt handling, fault delivery, and external termination.

A possible interpretation is that portions of the representation do not reflect intended executable behavior. No intended replacement is established. Conversely, under the literal represented model, the second gate sequence supplies a concrete contradiction; deep reachability is not merely an unspecified runtime condition.

## Evidence

The evidence consists exclusively of the supplied static body and identity map. The prior draft and feedback are not treated as independent proof.

Direct evidence establishes the input width, memory-access widths, gate dependencies, working-state assignment order, indirect calls, interrupt selector, and unconditional loop.

Derived conclusions include the low-halfword partition, incoming-link dispatch base on the bypass path, zero shift counts at executed equality gates, dominance of the internal assignment over its later read, the second-sequence contradiction, and the fixed target after a resumed interrupt.

No runtime trace, dependency implementation, or execution result was supplied. No runtime tests have been performed.
