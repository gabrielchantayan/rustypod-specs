<!-- Trusted provenance: F-e5bccf6c109a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e5bccf6c109a

## Purpose

F-e5bccf6c109a conditionally accesses halfword-sized memory, then enters an unconditional repetition containing two indirect calls and a software interrupt. There is no visible normal-return path.

No application-level purpose is established. The supplied body does not justify identifying this routine as a decoder, allocator, dispatcher, or handler for a particular data format.

The contract distinguishes three evidence levels:

- **Established static evidence:** operations and control-flow relationships visible in the supplied body.
- **Static deductions:** reachability and arithmetic consequences under the stated 32-bit machine model.
- **Unknowns:** executable fidelity, environmental behavior, and dependency contracts not established by that body.

No runtime tests have been performed.

## Interface

### Explicit interface

The routine declares no explicit parameters and no return value. It nevertheless consumes incoming machine state; absence of declared parameters does not imply absence of inputs.

The visible repetition has no exit. An indirect callee, interrupt, or fault may prevent further execution, so the absence of a normal return does not guarantee that the routine continues executing indefinitely.

### Implicit inputs

The following names describe behavioral roles, not implementation identifiers.

| Input role | Width in the supplied model | Use |
|---|---:|---|
| Primary value | 32 bits | Gates the initial transfer and participates in the conditional chain. |
| Link address | 32 bits | Supplies the initial call base and, on a restricted path, a halfword-read address. |
| Transfer address | 32 bits | Supplies the initial halfword source address. |
| Offset value | 32 bits | Supplies a byte displacement for the initial destination and participates in later conditions. |
| Comparison value | 32 bits | Compared with the offset value through the conditional chain. |
| Adjustment value | 32 bits | Subtracted from the initial destination address; later compared with the resulting working address. |
| Auxiliary value | 32 bits | Participates in a conditional addition. |
| Incoming carry | One architectural carry bit | Contributes to an early address calculation before the working carry is cleared. |
| Stack-reference address | 32 bits | Its address value participates in arithmetic; its stored contents are not visibly read. |
| Additional architectural state | Not fully established | Includes a synthetic state location written on a restricted path. |

The supplied body identifies a two-byte stack-associated item. Taking its address does not constitute reading its stored value.

### Widths and addressing

The supplied model distinguishes:

- Unsigned 16-bit memory reads and writes.
- Signed 16-bit reads in later conditional blocks.
- 32-bit integer and address values.
- 32-bit pointer-sized memory accesses in those later blocks.
- Single-bit carry conditions.

Address arithmetic in this contract is interpreted modulo \(2^{32}\). This is a machine-level interpretation of the displayed 32-bit operations, not a claim about portable pointer arithmetic.

An integer displacement added to an address is measured in bytes. Advancing a halfword pointer by one element adds two bytes. This distinction is material to both reachability and indirect-call target calculations.

Byte order, alignment requirements, address tagging, instruction-mode selection, and fault handling are not established.

### Preconditions for continued execution

There is no visible input validation. Continued execution requires the memory accesses and execution boundaries actually reached to be supported by the environment.

In particular:

- A zero primary value requires a readable halfword at the transfer address and a writable halfword at the derived destination.
- The rare mutation requires additional accesses through address zero under the reduced model.
- Each indirect target must support the displayed call.
- Each call and the software interrupt must return control if the following action is to occur.

These are execution requirements, not evidence that the routine owns the addressed memory or that the environment satisfies them.

## Behavior

### Entry and initial transfer

When the primary value is nonzero, none of the displayed conditional prefix memory operations executes. The initial call base remains the incoming link address.

When the primary value is zero, an additional working address is first calculated as the sum of:

- The incoming link address.
- The incoming carry.
- The stack-reference address logically shifted right by one bit.

This calculation uses the original carry, before the working carry is cleared.

The initial halfword transfer then has the following ordering:

1. Form the destination address by adding the offset value to the incoming transfer address as a byte displacement.
2. Read an unsigned 16-bit value from the incoming transfer address and retain that value for the transfer.
3. Update the working transfer address to the destination address minus the adjustment value.
4. Write the retained 16-bit value to the previously formed destination address.

Thus, the working-address update occurs **after the source read and before the destination write**. The write still uses the destination formed before that update; it does not use the adjusted working transfer address.

This ordering is established by the supplied body. It does not establish that the intermediate working-address state is externally observable.

After the transfer, the working carry is cleared, and the conditional-chain working value is set to the incoming link address. Source and destination may coincide; no non-overlap restriction or alias-based suppression of the access is visible.

### Conditional-chain reachability

The following conditional stages propagate only through zero results. A failed stage disables the subsequent stages rather than permitting an independent later condition to restart the chain.

The chain can initially progress only when both:

- The primary value is zero.
- The incoming link address is zero.

Although several expressions contain variable shifts, each such early shift is reached only when its shift-count value is already zero. No general policy for oversized shift counts is needed to characterize these reachable stages.

Under the 32-bit model, entry to the rare mutation requires all of the following:

- Primary value zero.
- Incoming link address zero.
- The additional working address calculated at entry equal to zero.
- Comparison value equal to the offset value.
- Adjusted working transfer address equal to the adjustment value.
- Auxiliary value zero.
- Stack-reference address plus twice the offset value equal to zero modulo \(2^{32}\).

The working carry is zero through the comparisons preceding the final address condition.

These requirements admit a stronger reduction. With a zero incoming link address, the additional working address is the logical half of the stack-reference address plus the incoming carry. The logical half lies between zero and \(2^{31}-1\); adding a single carry bit cannot wrap this sum to zero. Consequently, the incoming carry must be zero, and the stack-reference address must be either zero or one.

The later sum of the stack-reference address and twice the offset value cannot be zero for an odd stack-reference address. Therefore the reduced requirements for the rare mutation are:

- Primary value and incoming link address zero.
- Stack-reference address zero.
- Incoming carry zero.
- Offset value either zero or `0x80000000`.
- Comparison value equal to that offset value.
- Auxiliary value zero.
- Incoming transfer address plus the offset value equal to twice the adjustment value modulo \(2^{32}\).

These are deductions about the supplied representation. They do not establish that a supported execution environment can place the relevant stack reference at address zero.

### Rare additional mutation

If the preceding requirements hold, the routine performs a further mutation before entering the repeated calls.

The supplied ordering establishes:

- A write to a synthetic architectural-state location, with a value derived by multiplying the stack-reference address by 64.
- Calculation of a candidate updated link address by adding that same byte displacement to the incoming link address.
- A halfword mutation using reads through the incoming link address.
- Advancement of a working halfword pointer from the primary value by one element.
- Clearing of the working carry and adoption of the candidate updated link address.

The mutation uses a halfword read through the incoming link address as a zero-extended destination address. Another halfword-read occurrence through that address supplies a value increased by 15; the resulting store is truncated to 16 bits.

The two read occurrences appear within one store expression. Their relative evaluation order and their correspondence to physical memory transactions are not established. If the memory is stable, both supply the same value. For volatile, device-backed, or asynchronously changing memory, the destination-selecting value and the value being incremented must not be assumed equal.

Under the reduced entry requirements:

- The old and updated link addresses are both zero.
- The synthetic architectural-state write has value zero.
- The working halfword pointer advances from address zero to address two.

The synthetic state write precedes the halfword mutation. No meaning beyond the displayed assignment is established for that state location.

### Unreachable later blocks

Only the rare mutation enables the next conditional chain. On that path, the additional working address and updated link address are zero.

The first subsequent zero-result operation succeeds. The next combines the zero additional working address with the working pointer that has advanced to address two, producing two rather than zero. This disables the remaining conditional stages.

Accordingly, the following displayed operations are unreachable from the supplied entry under the stated model:

- A later shifted-address subtraction chain.
- A working-address increment of 36 bytes.
- Later increments of the offset and adjustment values.
- A pointer-sized load using an address derived from a narrowed character value.
- A signed-halfword read followed by two pointer-sized stores.
- A final signed-halfword read followed by stores at large displaced addresses.

These unreachable blocks do not establish an application-level structure layout. Their exclusion depends on the displayed entry, control flow, widths, and pointer scaling; independently established alternative execution semantics would require reassessment.

### Repeated external actions

After the reachable prefix, each iteration performs these actions in order:

1. An indirect call at the current call base plus `0xEB6` bytes.
2. An indirect call at address `0x0857A94B`.
3. A software interrupt with immediate `0x5B`.
4. Replacement of the call base with `0x083D4CC2`.

The first call has three visible 32-bit argument bit patterns:

- `0x00000000`
- `0x00000000`
- `0xFFFFFFA5`

The last bit pattern represents −91 as signed 32-bit or 4,294,967,205 as unsigned 32-bit. Its interpretation by the callee is unknown.

The second call has no explicit arguments in the supplied body. It may nevertheless consume implicit machine state.

The first iteration uses the incoming link address as its call base, except that the rare mutation supplies its updated link address, which reduces to zero.

Once an iteration reaches the call-base replacement, the first-call target for the next iteration is `0x083D5B78`. This calculation follows from the fixed replacement plus the byte displacement. It is conditional on the preceding calls and interrupt returning as displayed.

There is no visible loop-exit condition, cancellation check, success result, or error result.

## Side effects

### Memory and state

The reachable explicit prefix effects are:

- A conditional unsigned-halfword read and write.
- A much more restricted halfword mutation.
- A synthetic architectural-state write on that restricted path.

For the initial transfer, the established sequence is source read, working transfer-address update, then destination write. Only the read and write are established as memory accesses; the intermediate working-address assignment is not independently established as an externally visible event.

No stored stack-item contents are visibly consumed.

### External execution

The two indirect calls and software interrupt are observable execution boundaries. Their effects may include memory changes, device interactions, faults, termination, or non-returning control transfer.

No preservation guarantee is assigned to either callee or to the interrupt. In particular, the contract does not assume that they preserve globals, architectural state, memory, or ownership state. The later fixed call-base assignment describes the supplied continuation after those boundaries return.

### Ownership and lifetime

No allocation, release, acquisition, or ownership transfer is explicit in the reachable body.

Accessing an incoming address does not establish ownership of the corresponding storage. The allocation, release, and ownership behavior of the external dependencies remains unknown.

### Errors

There is no explicit error reporting or recovery.

Invalid memory addresses, unsupported alignment, invalid call targets, or an unsupported interrupt may prevent continuation. Fault classes, installed handlers, and recovery behavior are environmental unknowns.

## Dependencies

No resolved opaque function ID is supplied for either indirect callee. Their addresses identify execution boundaries, not authoritative function identities.

| Boundary | Established invocation | Unknown contract |
|---|---|---|
| First indirect call | Current call base plus `0xEB6`; three explicit 32-bit arguments. | Identity, calling convention, implicit inputs, outputs, return behavior, preservation, and ownership effects. |
| Second indirect call | Address `0x0857A94B`, after the first call returns. | Identity, implicit inputs, outputs, return behavior, and side effects. |
| Software interrupt | Immediate `0x5B`, after the second call returns. | Service identity, argument convention, outputs, state changes, and continuation behavior. |
| Execution environment | Supplies memory access, indirect execution, and interrupt handling. | Mapping, byte order, alignment, fault policy, and executable-address interpretation. |

No dependency contract is inferred from the numeric constants alone.

## Acceptance examples

These are conditional static acceptance cases, not executed tests.

### Nonzero primary value

Given a nonzero primary value and incoming link address `0x00001000`, with valid execution targets and returning external operations:

- No conditional prefix data-memory operation occurs.
- The first call targets `0x00001EB6` with argument bit patterns zero, zero, and `0xFFFFFFA5`.
- The second call targets `0x0857A94B`.
- Interrupt `0x5B` follows.
- The next iteration’s first call targets `0x083D5B78`.

### Initial transfer with an adjusted working address

Given:

- Primary value zero.
- Transfer address `0x00002000`.
- Offset value six.
- Adjustment value two.
- Halfword `0xABCD` readable at `0x00002000`.
- Writable memory at `0x00002006`.
- A nonzero incoming link address.

The routine forms destination `0x00002006`, reads and retains `0xABCD`, updates its working transfer address to `0x00002004`, and then writes the retained value to `0x00002006`.

The address update does not redirect the store to `0x00002004`. Its intermediate state is not asserted to be externally observable. The nonzero link address prevents the rare mutation.

### Aliased initial transfer

With primary value zero and offset value zero, the initial source and destination coincide.

The displayed behavior remains a halfword read followed, after the working-address update, by a halfword write to the same address. It is not specified as a skipped no-op. This distinction matters when accesses themselves have observable effects.

### Conditional rare mutation

Assume an environment supporting the zero stack-reference address, the synthetic state write, and accesses through address zero. Let:

- Primary value and incoming link address be zero.
- Incoming carry be zero.
- Stack-reference address be zero.
- Offset and comparison values be zero.
- Transfer address be `0x00000200`.
- Adjustment value be `0x00000100`.
- Auxiliary value be zero.
- Stable memory at address zero contain halfword `0x0030`.

The initial transfer reads at `0x00000200`, updates the working transfer address to `0x00000100`, and writes the saved halfword back to `0x00000200`.

The rare mutation writes zero to the synthetic state location and stores halfword `0x003F` at address `0x00000030`. The first indirect call targets `0x00000EB6`.

The later pointer-store blocks do not execute.

### Non-returning dependency

If the first indirect call does not return, the second call, interrupt, and call-base replacement are not reached for that iteration. The unconditional repetition does not override a dependency’s non-returning behavior.

## Uncertainties

The following remain **documented_only**:

- Fidelity of the supplied body to the executable entry and instruction semantics.
- The authoritative calling convention and identities of the implicit inputs.
- The nature and externally visible effects of the synthetic architectural-state location.
- Whether a supported environment permits the required zero stack-reference address.
- Byte order, alignment, mapping, and fault behavior.
- The ordering and physical realization of the two halfword-read occurrences in the rare mutation.
- Indirect-target address tagging and instruction-mode rules.
- Callee identities, signatures, implicit inputs, preservation rules, and ownership behavior.
- The meaning and continuation behavior of interrupt `0x5B`.
- Whether intermediate working-state changes are externally observable.
- Any application-level meaning of the constants or mutations.

Inferred intent is limited to recognizing a repeated external-action sequence. No stronger purpose is established.

## Evidence

The supplied body is the sole static evidence.

It establishes a parameterless declared interface, incoming-state consumption, conditional halfword accesses, zero-gated conditional stages, two indirect calls, a software interrupt, and repetition without a visible exit.

For the initial transfer, the source read precedes the working transfer-address update, which precedes the destination write. The saved source value and previously formed destination remain the operands of that write.

The rare-path conditions, exclusion of later blocks, and numeric call-target calculations are static deductions under the stated 32-bit model. They are not runtime observations.

No runtime execution, memory-access observation, dependency resolution, or executable-level validation has been performed.
