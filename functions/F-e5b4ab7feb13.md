<!-- Trusted provenance: F-e5b4ab7feb13; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e5b4ab7feb13

## Purpose

**F-e5b4ab7feb13** conditionally performs up to two explicit 16-bit memory stores, then enters an unconditional dispatch loop containing two indirect calls and a software interrupt. The supplied static representation does not establish an application-level purpose.

Under the represented 32-bit arithmetic:

- A nonzero low nibble in the second argument bypasses all prefix memory accesses.
- A zero low nibble causes an initial halfword transfer.
- Additional zero-result gates can permit one further halfword store, using a halfword read as a numeric destination address.
- The comparison chain following that store necessarily fails. The later pointer acquisition, signed-halfword operations, and paired 32-bit stores are unreachable.
- Whenever the prefix completes, the first dispatch target is the incoming link-like address plus `0x0EB6`, modulo 32 bits. Reachable prefix operations do not change that base numerically.

There is no represented normal return. Invalid accesses, nonreturning callees, or interrupt-driven control transfers may prevent the loop or later operations from executing.

## Interface

### Explicit inputs

| Argument position | Represented width | Established use |
|---|---:|---|
| First | 32 bits | No explicit use in the visible body. |
| Second | Unsigned 32 bits | Controls prefix routing and supplies arithmetic operands. |
| Third | 32 bits | No explicit use in the visible body. |
| Fourth | 32 bits | No explicit use in the visible body. |
| Fifth | 16 bits | Its storage address participates in arithmetic; its incoming value is not explicitly loaded. |

The fifth argument’s address is a behavioral input distinct from its 16-bit value. Equal argument values at different addresses need not produce equal behavior.

The absence of explicit uses for the first, third, and fourth arguments establishes no register-preservation or calling-convention guarantee.

### Implicit inputs

The prefix also consumes incoming machine state. The following descriptive roles distinguish those inputs without assigning an undocumented ABI:

- **Original halfword source:** an incoming address used for the initial unsigned 16-bit read.
- **Byte-offset input:** an unsigned 32-bit quantity added to that source address.
- **Equality-peer input:** an unsigned 32-bit quantity compared with the byte-offset input.
- **Subtraction input:** an unsigned 32-bit quantity used to update the source-address state and subsequently compared with it.
- **Arithmetic input:** a 32-bit quantity used in a carry-aware addition.
- **Incoming link-like address:** an address used as the initial dispatch base and, on the most restrictive prefix path, as a halfword source.

Initialization and ABI placement of these inputs are not established.

A separate machine-state location receives a scaled address on the restrictive prefix path. Its later apparent consumer is unreachable, so an unknown incoming value for that location does not affect any reachable operation established here.

### Outputs and termination

No return value or ordinary completion path is represented.

Observable effects may include:

- One initial 16-bit memory store.
- One additional, conditionally reachable 16-bit memory store.
- Repeated indirect calls.
- Repeated software interrupts with immediate `0x5B`.
- Effects of the callees and interrupt handler, whose contracts are unavailable.

There is no visible error return, recovery branch, rollback, allocation protocol, or cleanup protocol.

### Widths and address interpretation

The behavioral analysis uses 32-bit address bit patterns and arithmetic reduced modulo \(2^{32}\), with carry evaluated separately where explicitly represented.

Reachable explicit data accesses are unsigned 16-bit reads and 16-bit stores. A halfword-pointer increment of one advances its numeric address by two bytes. Cast-based address additions described as byte offsets do not acquire that scaling.

This interpretation does not establish portable pointer validity. Byte order, alignment restrictions, executable-address encoding, address-space mapping, and invalid-access behavior remain unspecified.

## Behavior

### 1. Initial routing

The first routing value is the second argument multiplied by `0x10000000`, reduced to 32 bits. It is zero exactly when the second argument’s low nibble is zero.

If that nibble is nonzero:

- The initial transfer does not execute.
- Every dependent prefix region is disabled.
- Dispatch begins with the incoming link-like address as its base.

If the nibble is zero, the initial transfer executes. On that path:

- The separately extracted bit 2 of the second argument is necessarily clear.
- The second argument multiplied by `0x20000000`, reduced to 32 bits, is also zero.
- Both argument-derived products used in subsequent comparisons are therefore zero.

Upper bits of the second argument do not alter these conclusions.

### 2. Initial halfword transfer

On the zero-low-nibble path, an address-like comparison operand is formed from:

- The incoming link-like address.
- The numeric address of the fifth argument shifted right by one.
- The extracted bit contribution, which is zero on this path.

The transfer destination is the original halfword source address plus the byte-offset input, modulo 32 bits.

The represented sequence is:

1. Read an unsigned 16-bit value from the original halfword source.
2. Update the source-address state to the transfer destination minus the subtraction input.
3. Store the previously read value as 16 bits at the transfer destination.
4. Use the incoming link-like address as the result controlling the next zero gate.

The source-state update occurs before the store in the representation. Whether that state update is externally visible as a register change is not established.

The source read precedes the destination store even when the addresses are equal. No atomicity, transfer elision, or non-aliasing guarantee is present.

A nonzero incoming link-like address stops the remaining prefix chain after this transfer. It does not prevent dispatch.

### 3. Gates permitting the additional halfword store

Continuation beyond the initial transfer requires the incoming link-like address to be zero.

The following comparisons use exclusive-or results as zero tests. Every reached variable right shift in these comparisons has count zero: the preceding gate requires its result to be zero, and that result’s low byte supplies the shift count. These operations therefore do not introduce arbitrary data-dependent shifts on reachable paths.

For the additional halfword-store region to execute, the gates collectively require:

- Both argument-derived products to be zero.
- The separate fifth-argument-derived comparison operand to be zero.
- The equality-peer input to equal the byte-offset input.
- The updated source-address state to equal the subtraction input.
- The carry-aware arithmetic result to be zero.
- A final address-derived addition to produce zero.

The carry entering the arithmetic-input addition is zero. The second argument’s low nibble is also zero, so its scaled contribution is zero. The addition therefore requires the arithmetic input itself to be zero and produces no outgoing carry.

The final addition consequently has no incoming carry. Its result is the numeric fifth-argument address plus twice the byte-offset input, reduced to 32 bits.

These conditions further constrain the fifth-argument address:

1. The incoming link-like address is already zero.
2. The earlier address comparison therefore requires the fifth-argument address shifted right by one to be zero.
3. As an unsigned 32-bit address, it can only be zero or one.
4. Twice an unsigned 32-bit quantity has an even low bit, including after wraparound.
5. A zero result from the final addition therefore requires the fifth-argument address to be even.
6. The fifth-argument address must consequently be zero.

The final addition then also requires the byte-offset input to be either `0` or `0x80000000`.

These are static arithmetic constraints, not evidence that an ordinary execution environment can supply valid storage and calling state satisfying them.

### 4. Additional address-derived halfword store

If all preceding gates succeed, the region:

- Assigns a machine-state quantity from 64 times the numeric fifth-argument address.
- Forms a replacement link-like address by adding that same scaled address to the incoming link-like address.
- Performs a halfword store whose destination and value are derived from unsigned halfword dereferences through the incoming link-like address.
- Advances a comparison operand derived from the first argument-product value by one halfword.

The established entry constraints make both the incoming link-like address and fifth-argument address zero. Therefore:

- The scaled machine-state assignment is numerically zero.
- The replacement link-like address is zero.
- The halfword dereferences use numeric address zero.
- The advanced comparison operand becomes numeric address `2`, because its previous value was zero and its increment is two bytes.

For the store, one unsigned halfword dereference supplies the numeric destination address. Another supplies the value increased by `0x000F`. The stored value is reduced to 16 bits.

There are two source dereference occurrences in this expression. Their mutual evaluation order, and whether execution realizes them as separate observable reads, are not established. If reads can differ because of memory-mapped behavior or external mutation, the destination-producing value and value-producing value must not be assumed equal.

Both required source values must be obtained before the resulting store can complete. This region follows the initial transfer, so an aliasing initial store can affect its source reads.

### 5. Mandatory failure of the following chain

The additional store does not permit execution of the later arithmetic and write regions.

The following chain is disabled entirely if the additional-store region never executes. If that region executes, its first continuation gate tests the replacement link-like result:

- A nonzero replacement would stop the chain immediately.
- A zero replacement allows the first exclusive-or comparison, which produces zero because the fifth-argument-derived operand is also zero.
- The next comparison uses the advanced operand, now `2`, against the zero fifth-argument-derived operand. Its result is `2`, not zero, so continuation stops.

The stronger entry constraints above establish that the replacement is zero in this representation, making the final case the actual continuation.

Consequently, all of the following are unreachable under the stated 32-bit interpretation:

- The later self-counted left shift.
- The subtraction-dependent gate sequence.
- The pointer acquisition involving a character-width conversion.
- The increments associated with that acquisition.
- The signed-halfword-derived paired 32-bit stores.
- The final pair of 32-bit stores.
- Their associated dispatch-base adjustments.

Their presence in the supplied representation does not make them possible routine effects.

### 6. Dispatch loop

Every path completing the prefix reaches the loop with a dispatch base numerically equal to the incoming link-like address.

The only reachable prefix assignment that could replace that base requires both the old base and the scaled fifth-argument address to be zero. It therefore replaces zero with zero. The later base-adjustment regions are unreachable.

Each loop iteration performs the following sequence, provided preceding transfers return normally:

1. Indirectly call the current dispatch base plus `0x0EB6`, modulo 32 bits, with three explicit 32-bit arguments: `0`, `0`, and `0xFFFFFFA5`.
2. Indirectly call numeric target `0x0857A94B`, with no explicit arguments represented.
3. Issue software interrupt `0x5B`.
4. Assign `0x083D4CC2` as the next iteration’s dispatch base.

Thus:

- The first iteration’s first target is always the incoming link-like address plus `0x0EB6`, modulo 32 bits.
- After an iteration reaches its final assignment, the next iteration’s represented first target is `0x083D5B78`.
- These are computed code targets, not evidenced loads of function pointers from those addresses.

The constant-base assignment follows the interrupt; reaching it requires the interrupt to resume. The loop has no exit condition.

## Side effects

### Memory and ordering

The reachable prefix contains at most two explicit stores, each 16 bits wide:

1. The initial halfword transfer.
2. The gated address-derived halfword store.

The prefix contains no reachable 32-bit load or store. In particular, the signed-halfword and paired-store expressions do not contribute possible side effects.

There is no visible bounds checking, null rejection, alignment repair, synchronization, or rollback. Numeric address zero is not rejected before the restrictive store region attempts its source reads.

Earlier completed stores remain observable if a later operation faults. Aliasing is permitted by the representation, so effects must be considered in execution order.

### Ownership and dependency effects

No reachable operation is identified as allocation, release, object construction, or ownership transfer.

The dependency bodies are unknown. Their memory, ownership, global-state, and machine-state effects must not be assumed absent. No preservation guarantee follows from the visible call sequence.

### Machine state

The prefix updates arithmetic, comparison, address-like, and explicitly represented machine-state quantities. Whether particular updates escape through registers or other externally visible state is **documented_only**.

The dispatch-base conclusions describe the represented control flow and assignments; they do not establish a general callee-preservation ABI.

## Dependencies

Only **F-e5b4ab7feb13** has a supplied verified opaque function ID. No resolved opaque function IDs are available for the external targets; none are invented here.

| Unresolved dependency role | Established interaction | Unknown contract |
|---|---|---|
| First indirect target | Computed from the dispatch base; receives three explicit 32-bit arguments. | Identity, ABI, actual additional inputs, return behavior, memory effects, ownership, preservation. |
| Second indirect target | Numeric target `0x0857A94B`; no explicit arguments represented. | Identity, ABI, implicit inputs, return behavior, memory effects, ownership, preservation. |
| Interrupt handler | Invoked by software interrupt `0x5B`, after both calls return. | Identity, interrupt ABI, resumption, memory effects, ownership, machine-state effects. |

Numeric target addresses do not establish stable semantic function identities.

## Acceptance examples

These are conditional static criteria, not runtime observations.

### Prefix bypass

Given a second argument of `1` and otherwise valid dispatch conditions:

- No prefix data-memory access occurs.
- The first target is the incoming link-like address plus `0x0EB6`, modulo 32 bits.
- Its explicit arguments are `0`, `0`, and `0xFFFFFFA5`.
- If both calls return, the interrupt follows.
- If the interrupt resumes and the iteration completes, the next first target is `0x083D5B78`.

### Initial transfer only

Given a second argument of `0x10`, a nonzero incoming link-like address, and valid initial source and destination storage:

- Both argument-derived products are zero.
- One unsigned halfword is read and transferred to the byte-offset destination.
- The later prefix chain stops at the nonzero link-like address.
- The first dispatch target remains the incoming link-like address plus `0x0EB6`.

### Restrictive additional store

Assume the representation admits:

- Fifth-argument address zero.
- Incoming link-like address zero.
- Byte-offset and equality-peer inputs zero.
- Original halfword source address twice the subtraction input, modulo 32 bits.
- Arithmetic input zero.
- Valid required memory accesses.

Then the additional halfword-store region passes its entry gates. If its two source dereference occurrences both yield `0xFFF8`, it stores `0x0007` at numeric address `0x0000FFF8`.

Afterward, the comparison using numeric operand `2` fails its zero test. No pointer acquisition or 32-bit store follows. The first dispatch target is `0x00000EB6`.

### Invalid initial source

With a zero low nibble and an unreadable initial halfword source:

- The representation provides no recovery path.
- Neither the initial store nor arrival at dispatch is guaranteed.
- No error return is required.

### Nonreturning dependency

If the first indirect target does not return, the second call and interrupt do not follow through the represented sequence. If the second target does not return, the interrupt does not follow.

## Uncertainties

The following remain **documented_only**:

- Application-level purpose and validity of the represented function boundary.
- ABI and initialization of implicit inputs.
- Whether an actual environment admits the restrictive prefix state, including fifth-argument address zero and readable address zero.
- Alignment, byte order, address mapping, and executable-address conventions.
- Evaluation details of the repeated halfword dereferences.
- External visibility of represented machine-state updates.
- Dependency identities, inputs, ownership effects, preservation rules, and interrupt resumption.
- Platform behavior on invalid accesses or invalid code targets.

The later regions are not merely environmentally uncertain: they are unreachable under the stated arithmetic and control-flow interpretation. Their shift semantics, character signedness, and signed-store expressions therefore do not affect reachable behavior.

A representation artifact or specialized machine-state operation is a possible explanation for the unusual structure, not an established purpose.

## Evidence

This contract uses only the supplied static representation. No runtime tests have been performed.

Established static evidence includes argument widths, the initial transfer’s ordering, the zero-gated comparisons, reachable halfword-store widths, computed call targets, interrupt immediate, and unconditional loop.

The principal deductions are:

- Multiplication by `0x10000000` partitions routing by the second argument’s low nibble.
- Reached comparison shifts have count zero.
- Entry to the additional-store region requires a zero fifth-argument address.
- The halfword increment changes a zero comparison operand to `2`, forcing the subsequent chain to fail.
- The later arithmetic, pointer acquisition, and 32-bit stores are unreachable.
- The only reachable replacement of the dispatch base leaves it numerically unchanged.

No dependency bodies or runtime observations support stronger claims about execution validity, ownership, preservation, or intended purpose.
