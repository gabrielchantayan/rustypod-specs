<!-- Trusted provenance: F-7e9a6d7ffc47; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7e9a6d7ffc47

## Purpose

F-7e9a6d7ffc47 conditionally transfers a halfword and updates represented machine state, then enters an unconditional dispatch cycle. Each completed cycle invokes two call targets, requests software interrupt `0x5B`, and selects a fixed dispatch base for the next iteration.

No application-level purpose is established. The supplied representation does not establish a conventional buffer operation, object construction, recognized protocol, or successful termination.

This contract distinguishes:

- **Established static evidence:** represented accesses, widths, arithmetic, conditions, and invocation order.
- **Derived static consequences:** conclusions obtained from that evidence under the stated 32-bit numerical model.
- **Inferred intent:** possible interpretations that are not required behavior.
- **documented_only:** unresolved behavior for which no executable guarantee is asserted.

A substantial suffix of the preprocessing is represented but unreachable under the literal zero gates and 32-bit address model. Its operations are documented separately rather than presented as reachable effects.

## Interface

### Explicit input

The explicit argument, **X**, is an unsigned 32-bit value.

Define:

- **P:** X multiplied by `0x10000`, retaining the low 32 bits.
- **Q:** initially P; replaced on the zero-P path by X multiplied by `0x20000`, retaining the low 32 bits.

P is zero exactly when X’s low 16 bits are zero. On that path, replacement Q is also zero. A nonzero X can therefore enter zero-P preprocessing.

Bit `0x4000` of X supplies an initial carry-like condition. That bit is necessarily clear whenever P is zero.

### Implicit inputs

The representation also consumes these incoming values:

| Label | Meaning |
|---|---|
| **L** | Link-like numerical address; initial dispatch base |
| **H** | Halfword-source address |
| **A** | Unsigned 32-bit comparison operand |
| **B** | Unsigned 32-bit byte displacement and arithmetic operand |
| **C** | Unsigned 32-bit address adjustment and comparison operand |
| **D** | Unsigned 32-bit addition operand |
| **S** | Numerical address of the represented entry stack slot |

These labels describe data dependencies, not a verified public calling convention.

No direct value use of the represented entry stack slot is shown. Its address S is used numerically and in typed address arithmetic. **Indirect accesses may nevertheless alias the slot and consume or overwrite its contents.** In particular, the initial halfword read consumes memory at S when H equals S. The slot is not excluded from any other computed access.

An additional machine-state location is assigned during preprocessing and referenced later. Its incoming value is not an established input: every represented route to that reference passes through the assignment, with no dependency call intervening. The location’s storage and aliasing semantics remain unresolved.

### Results and completion

No explicit return value or normal return path is represented.

If preprocessing completes and all reached calls and interrupt requests continue normally, execution remains in the dispatch cycle indefinitely. Faults, exceptions, non-returning dependencies, or external intervention may prevent continuation. These outcomes are **documented_only**, not specified error returns.

### Widths and layouts

The represented model uses:

- Unsigned 16-bit memory transfers.
- Signed 16-bit reads followed by 32-bit sign extension.
- 32-bit numerical addresses, arithmetic results, and pointer-sized stores.
- Two-byte scaling for halfword-pointer increments.
- Four-byte spacing between the fields of each represented store pair.

Unless stated otherwise, numerical arithmetic retains the low 32 bits. This is a model of the supplied representation, not an independently verified execution-platform contract.

Endianness, alignment requirements, executable-address interpretation, character signedness, and the actual ABI remain **documented_only**.

## Behavior

### Entry selection and initial transfer

When P is nonzero, preprocessing performs no represented memory access and proceeds to dispatch using L as its base.

When P is zero, preprocessing derives:

- **U:** L plus the initial carry-like value plus S logically shifted right by one. The carry-like contribution is zero on this path.
- **T:** H plus B bytes.
- **H′:** T minus C.

It reads an unsigned halfword from H and subsequently writes the captured value to T. H′ is retained as the adjusted source-address value for later comparisons.

B is a byte displacement, not a halfword index. If B is zero, the read and write target the same address; both accesses remain represented. If H equals S, this operation reads the entry stack slot indirectly. If T equals S, it writes that slot indirectly.

No overlap, mapping, alignment, or write-permission check precedes the transfer. Later gate failure does not undo it.

### Gates leading to the additional state update

Deeper preprocessing first requires L to be zero. Successive gates then require:

- U to be zero.
- P to equal U.
- P to equal Q, twice.
- B to equal A.
- C to equal H′.

The comparisons use exclusive-or and logical right shifts. Each relevant shift count comes from the low byte of an intermediate result already required to be zero. Consequently, these reached shifts have count zero and the gates are equality tests.

Carry-like state is preserved through those zero-count shifts. Its value is false on this path.

A subsequent addition combines D, the low 16 bits of X multiplied by `0x8000`, and that carry-like value. Continuation requires a zero 32-bit result. Carry-out is separately computed from the component additions.

The preceding conditions already make X’s low 16 bits zero and the incoming carry false. Therefore this gate requires D to be zero and produces no carry-out.

The next addition combines S, twice B, and the resulting carry. Continuation again requires a zero 32-bit result. The represented carry calculation includes numerical addition and an address expression scaled by the entry slot’s two-byte type.

The literal gates imply:

- L is zero.
- U being zero requires S logically shifted right by one to be zero.
- Thus S is either zero or one.
- Twice B has an even low-32-bit value, so adding S equal to one cannot produce zero.
- Therefore S is zero.
- B is either zero or `0x80000000`.
- A equals B, C equals H′, and D is zero.

Doubling B is itself reduced to 32 bits before its use as an addition operand. A bit discarded by that multiplication is not automatically an addition carry-out.

### State assignment and dependent halfword write

If those gates succeed, the representation:

- Assigns the additional machine-state location a numerical value based on S multiplied by `0x40`.
- Forms an updated link-like address by advancing L by S multiplied by `0x40` bytes.
- Performs a halfword write whose destination is obtained from an unsigned halfword read through the prior L.
- Obtains the written value from a halfword read through that same prior L, adds `0xF`, and retains 16 bits.
- Advances a primary-address derivative by two bytes.
- Clears the carry-like state used by subsequent comparisons.

The destination-producing and value-producing expressions each reference the halfword at the prior L. Their relative evaluation order, and whether execution performs one or two physical reads, are not established. Both dependencies precede the dependent store.

For stable ordinary memory containing value v at the prior L, the destination is v zero-extended to 32 bits, and the stored value is v plus `0xF`, reduced to 16 bits. If the two references observe different values, the representation does not justify requiring one common v.

Every literal path reaching this stage has S and L equal to zero. Therefore:

- The assigned numerical value is zero.
- The updated link-like address is zero.
- The halfword source address for the dependent write is zero.
- The advanced primary-address derivative is `0x00000002`.

Because S is also zero, these source reads alias the represented entry stack slot. The absence of a direct use of the named incoming slot value does not eliminate these indirect reads.

The machine-state assignment precedes the dependent write. Its unresolved storage semantics prevent assuming that it cannot affect aliased memory. Permission to access address zero is an environmental prerequisite, not a guarantee.

### Static barrier to later preprocessing

After the dependent write, further gates require the updated link-like address to be zero and to match U. Those conditions hold on the literal path just derived.

The next gate requires the advanced primary-address derivative to equal U. The derivative is two and U is zero, so this gate fails.

Under the stated 32-bit model:

- Later subtraction stages are unreachable.
- Pointer extraction and paired field stores are unreachable.
- The later reference to the additional machine-state location is unreachable.
- The final conditional store pair is unreachable.

This contradiction does not depend on resolving the later large-shift behavior.

Reachable preprocessing leaves the first dispatch base equal to incoming L. Its only reachable link-like update occurs when both the prior and updated values are zero.

### Represented operations beyond the barrier

The following describes dormant structure, not an executable alternative to the preceding reachability conclusion.

**Subtraction stages.** After additional equality gates, a quantity combines U, carry-like state, and `0x085740E8`. It is shifted left by a count taken from its own low byte. That count is not established to be below 32.

The shifted quantity is subtracted from Q, with an additional subtraction of one when the incoming carry-like value is false. A separate comparison determines outgoing carry-like state using the shifted quantity, Q, a wrapped difference, and incoming carry.

Further zero-gated stages subtract H′ from P, then subtract Q logically shifted right by one from P, each with its represented carry-dependent correction. A Q derivative advances by `0x24` bytes. The flag comparisons and arithmetic results are distinct expressions; they must not be replaced by an assumed architectural borrow convention.

**Pointer extraction and state reference.** A later stage increases B by eight and C by ten, and forms a link-like address thirteen bytes beyond the preceding one.

It reads a halfword through the preceding link-like address, narrows that value to character width, adds four, and reads a pointer-sized value from the resulting numerical address. Character signedness can affect the address.

A subsequent zero test combines `0x042BA0A0`, the additional machine-state location logically shifted right by one, the advanced link-like address, and carry-dependent corrections. Every represented route to the reference first performs the earlier assignment. Nevertheless, unresolved storage semantics and intervening aliasing effects prevent assuming that the location necessarily retains the assigned value.

**First store pair.** If the zero test succeeds, a halfword read through U is interpreted as signed 16-bit and sign-extended to 32 bits. The resulting pattern is stored at the advanced link-like address. The extracted pointer-sized value is stored four bytes later, after the first store.

The eventual dispatch base is adjusted by subtracting the advanced primary-address derivative. A U derivative advances by one byte, not one halfword.

**Final store pair.** Further zero gates involve the sign-extended value, extracted pointer, address derivatives, and adjusted comparison operands. If they succeed, a signed halfword is read thirteen bytes beyond the then-current dispatch base, and that base advances by twenty-seven bytes.

The sign-extended value is stored at its own numerical value minus `0x08574200`. The extracted pointer-sized value is then stored four bytes later.

Neither pair establishes an allocated object, valid destination, or ownership transfer.

### Unconditional dispatch cycle

Every normally continuing reachable path enters a cycle with this observable invocation order:

1. Call **F-000000000001** at the dispatch base plus `0xEB6` bytes, with explicit 32-bit argument patterns `0`, `0`, and `0xFFFFFFA5`.
2. Call **F-000000000002** at numerical target `0x0857A94B`, with no explicit arguments shown.
3. Request **F-000000000003** through software interrupt immediate `0x5B`.
4. Replace the dispatch base with `0x083D4CC2`.

The cycle repeats without a represented exit condition.

The first target is the calculated address itself, not a pointer fetched from a table. Its first-iteration address is incoming L plus `0xEB6`. After a normally completed iteration, the next first-call target is `0x083D5B78`.

The third argument’s established property is its 32-bit pattern. Interpreting it as signed −91 requires an unavailable callee contract.

## Side effects

Reachable direct effects are:

- The initial conditional halfword read and write.
- The more restrictive machine-state assignment and dependent halfword write.
- Repeated dependency calls and interrupt requests.

Indirect accesses may read or overwrite the entry stack slot when their computed addresses alias S. No general non-aliasing guarantee exists. Earlier writes can affect later reads, including repeated reads through the same numerical address.

Carry-like values are explicitly computed and propagated. Wrapping is not rejected or reported as an error. The representation does not establish that every discarded arithmetic bit contributes to a carry flag.

No explicit allocation, deallocation, reference-count operation, locking, ownership transfer, or error-reporting mechanism is established. Pointer-shaped values do not imply ownership.

Normal continuation requires the environment to permit every reached memory access and invocation. Mapping, bounds, alignment, write permission, and executable permission are not validated by the represented function.

Dependency effects on memory, globals, machine state, ownership, and continuation remain **documented_only**.

## Dependencies

These opaque IDs identify documentary dependency boundaries, not independently resolved implementations.

| Function ID | Established invocation | Unresolved contract |
|---|---|---|
| F-000000000001 | First call each iteration; target is dispatch base plus `0xEB6`; three explicit argument patterns | ABI, identity, return behavior, state effects, ownership |
| F-000000000002 | Second call each iteration; target `0x0857A94B`; no explicit arguments shown | Actual parameters, return behavior, preservation, ownership |
| F-000000000003 | Software interrupt request with immediate `0x5B` | Service identity, privileges, effects, exceptions, continuation |

The interrupt boundary does not assert an ordinary function-call ABI.

No explicit recursive invocation of F-7e9a6d7ffc47 is identified. No dependency is assumed to preserve globals or acquire, retain, or release memory.

## Acceptance examples

These are conditional static expectations, not runtime test results.

1. **Nonzero P.** With X equal to one, P is `0x00010000`. Preprocessing performs no represented memory access. The first dispatch target is L plus `0xEB6`.

2. **Nonzero X with zero P.** With X equal to `0x00010000`, P and replacement Q are zero. A halfword is read from H and its captured value is written to H plus B before later gates can reject deeper preprocessing.

3. **Entry-slot alias.** With zero P, H equal to S, and a permitted halfword read at S, the initial transfer consumes the entry stack slot indirectly. If B is also zero, it subsequently writes the captured halfword back there. This does not require a direct value use of the named incoming slot.

4. **Nonzero L.** On the zero-P path with nonzero L, the initial transfer occurs, but deeper preprocessing stops. Dispatch retains L as its base.

5. **Reachable low-address write.** Let X, L, S, A, B, C, D, and H all be zero. Assume the required accesses are permitted and both source references for the dependent write observe `0x0010` at address zero. The dependent write stores `0x001F` at address `0x00000010`. Its source reads alias S. The additional machine-state assignment has numerical value zero; the observation assumption includes any effects of that assignment.

6. **Wrapped doubling.** With S zero and B equal to `0x80000000`, twice B reduces to zero in 32 bits. If preceding gates hold, the corresponding zero-result gate succeeds. The multiplication’s discarded bit is not itself evidence of an addition carry-out, and no overflow error branch is represented.

7. **Dormant-store exclusion.** After any normally completed reachable dependent-write stage, the primary-address derivative is two and U is zero. The next relevant equality fails; the later pointer extraction and paired stores do not execute under this model.

8. **Completed iteration.** If both calls and the interrupt continue normally, the next iteration’s first target is `0x083D5B78`.

9. **Non-returning first dependency.** If F-000000000001 does not return or otherwise continue to its successor, the second call and interrupt are not reached in that iteration.

## Uncertainties

The following remain **documented_only**:

- Accuracy of the represented widths, recovered control flow, and function boundary.
- The ABI supplying implicit inputs.
- Storage and aliasing semantics of the additional machine-state location.
- Physical read count and relative evaluation order of repeated halfword references in the dependent write.
- Out-of-range left-shift behavior in the unreachable suffix.
- Correspondence between represented carry-like expressions and architectural flags.
- Character width and signedness relevant to dormant pointer extraction.
- Endianness, alignment, mappings, permissions, and fault behavior.
- Dependency implementations, executable-target interpretation, and continuation guarantees.

Actual aliasing depends on supplied addresses and storage semantics. It is not established that the entry stack slot is accessed on every invocation; it is equally unjustified to claim its contents are never consumed.

The unreachable suffix could reflect inaccurate recovery or intentional dormant structure. Neither explanation is established. “Persistent dispatch” describes control-flow structure, not verified application intent.

## Evidence

This contract is based solely on static examination of the supplied representation.

Direct evidence includes transfer widths, scaled and byte-based address arithmetic, ordered transfer statements, zero-dependent gates, explicit carry computations, assignment-before-reference control flow, call argument patterns, interrupt immediate, and the unconditional cycle.

The entry stack slot has no direct represented value use. The initial source address H and later computed addresses are not constrained to exclude S. Thus indirect consumption or modification of that slot is permitted by the representation; the all-zero acceptance case explicitly includes such aliasing.

The zero-P implications, restrictions on S and B, unchanged reachable first dispatch base, and unreachable suffix are derived static consequences under the stated 32-bit model.

No runtime tests have been performed. No memory mapping, dependency implementation, ABI, platform flag convention, or application purpose has been independently verified.
