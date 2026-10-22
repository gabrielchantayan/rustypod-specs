<!-- Trusted provenance: F-d64c9b93d4be; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d64c9b93d4be

## Purpose

F-d64c9b93d4be is an indirect-dispatch adapter. It obtains a dispatch base from the object supplied as its first argument, reads a 32-bit target descriptor at byte offset 12 from that base, clears the descriptor’s two least significant bits, and invokes the resulting target with all eight incoming arguments.

The established behavior is dispatch and argument forwarding, not a particular operation on the object. The selected target determines any substantive result, mutation, allocation, release, or external interaction.

The adapter has no explicit result. If the selected target returns normally, the adapter returns normally without exposing a target return value.

## Interface

### Inputs

The interface has eight arguments:

| Position | Role | Established representation |
|---|---|---|
| 1 | Address of an object or dispatch-bearing record | Pointer to storage whose initial word supplies the dispatch base |
| 2–8 | Forwarded payload | Seven separate 32-bit values |

The first argument is both a source of dispatch information and the first argument passed to the selected target. It is not replaced by the loaded dispatch base or adjusted to point to a subobject.

The remaining arguments are opaque. The adapter does not interpret them as integers, addresses, handles, flags, or structures. Each occupies four bytes in the supplied representation. Any higher-level meaning belongs to the caller and the selected target.

### Required readable layout

In the represented 32-bit model, dispatch requires these two locations:

| Location | Width | Meaning |
|---|---:|---|
| Beginning of the object addressed by argument 1 | 32 bits | Dispatch-base value |
| Dispatch base plus 12 bytes | 32 bits | Target descriptor |

The descriptor’s upper 30 bits provide the effective target address. Its bottom two bits do not contribute to that address.

The dispatch-base value itself is not masked. The offset is applied to that value before the descriptor is read. There is no additional dereference through the descriptor before invocation.

### Output and completion

The adapter returns no explicit value. It does not convert a target return value into a status or store it in a caller-provided destination.

Normal completion requires the selected target to return normally. If that target does not return, the adapter does not independently complete.

### Preconditions for a defined successful invocation

The caller must provide an object address readable for the initial word, a dispatch base for which the offset-12 descriptor read is valid, and a descriptor whose masked value identifies a callable target.

That target must accept the forwarded eight-argument call under the applicable calling convention. Its own data, lifetime, and execution preconditions must also hold.

The adapter does not check any of these conditions.

## Behavior

### Dispatch reads

The adapter first obtains the dispatch-base value from the beginning of the object addressed by argument 1.

It then uses that value to locate a descriptor 12 bytes beyond the base. The offset is measured in bytes, not in 32-bit entries. For a conventional four-byte-entry table, this location would correspond to the fourth entry, but a table interpretation is not required by the established behavior.

These reads are dependent and ordered: the initial object read supplies the address information needed for the descriptor read. Both precede invocation of the selected target.

The adapter does not perform a second object-header lookup or descriptor lookup after the target returns. It does not validate that either value remains unchanged.

### Target derivation

The descriptor is treated as a 32-bit unsigned value for masking. The effective target address is obtained by retaining exactly the bits selected by the hexadecimal mask `FFFFFFFC`.

Consequently:

- Descriptor bits 0 and 1 are discarded.
- All other descriptor bits are retained.
- Descriptors differing only in their bottom two bits select the same effective target.
- The adapter does not branch according to the discarded bits.
- This operation rounds the numerical descriptor down to a multiple of four; it does not round to the nearest multiple.

The resulting address is used for the call. There is no explicit test for a zero address, an executable mapping, an expected target identity, or an approved address range.

### Argument forwarding

The selected target receives eight arguments in the same positional relationship as the adapter:

1. The original object address.
2. The original second argument.
3. The original third argument.
4. The original fourth argument.
5. The original fifth argument.
6. The original sixth argument.
7. The original seventh argument.
8. The original eighth argument.

The seven payload values are forwarded without adapter-level arithmetic, masking, interpretation, or dereferencing. Their four-byte representations are the relevant contract; the adapter establishes no signedness for their eventual use.

The object address is forwarded unchanged even though its initial word was used to find the target.

### Call and return

There is one explicit target invocation. The adapter contains no retry, alternate dispatch, fallback, or loop.

After a normal target return, the adapter returns without further explicit dispatch reads or object writes. A value returned by the target is not exposed as an adapter result.

This does not establish how a particular compiler implements the transfer, whether a stack frame exists, or which instruction form performs the call.

## Side effects

### Direct effects

The adapter explicitly performs:

- One object-header read.
- One target-descriptor read.
- One indirect invocation.

It contains no explicit data-memory write, allocation, release, logging operation, counter update, or persistent dispatch-state update.

Ordinary execution effects such as call-frame handling are outside the object-level behavior described here. The supplied representation does not establish their concrete machine-level layout.

### Effects attributable to the selected target

The target receives the object address and all seven payload values. Its behavior is unresolved.

In particular, the contract does not assume that the target:

- Preserves the object or its dispatch information.
- Preserves global state.
- Avoids allocating or releasing memory.
- Leaves pointer-like payload values valid.
- Returns normally.
- Produces no external effects.

If the target changes the object’s dispatch base or modifies the descriptor, the adapter does not explicitly inspect those changes after return.

### Ownership and lifetime

No adapter-local ownership transfer is expressed. Reading dispatch information and forwarding arguments do not, by themselves, establish an ownership convention.

The object and descriptor storage must remain valid for the adapter’s respective reads. Any lifetime requirements extending through the target invocation depend on the selected target.

The adapter has no explicit post-call object access. That fact does not establish that releasing the object during the target call is safe: the target’s own requirements remain unknown.

### Errors

There is no explicit error-reporting path. Invalid input is not converted into a documented error code or a successful no-op.

An unreadable object, an invalid descriptor location, or an unusable target address may lead to a memory-access or control-transfer failure under the execution environment. The supplied evidence does not establish a portable exception, signal, diagnostic, or recovery behavior.

## Dependencies

F-d64c9b93d4be depends on one dynamically selected callable target.

No distinct opaque function ID is supplied for that target. Its identity is determined at invocation time by the object-derived descriptor, so it cannot be named as a resolved dependency from the available evidence.

The dependency contract established here is limited to:

- Selection through the offset-12 descriptor.
- Clearing of the descriptor’s bottom two bits.
- Invocation with the eight forwarded arguments.
- Resumption of the adapter only if the target returns normally.

The target’s accepted payload meanings, return convention, memory ownership, global-state behavior, and failure behavior are unknown.

## Acceptance examples

These examples are conditional specifications derived from the supplied static representation. They are not reports of executed tests.

### Valid dispatch with ordinary payloads

Given readable dispatch storage and a descriptor whose masked address identifies a compatible returning target, that target must receive the original object address and the seven original 32-bit payload values. After the target returns, the adapter must return without an explicit result.

### Low-bit variants select the same address

Given otherwise identical valid state, descriptors `00001000`, `00001001`, `00001002`, and `00001003` all produce the effective target address `00001000`.

This example establishes masking only. It does not assert that this address is mapped or callable in any actual environment.

### Offset is applied to the dispatch base

Given an object whose initial word contains dispatch base B, the descriptor must be read at B plus 12 bytes. Reading at the object address plus 12 bytes would not satisfy the contract unless those addresses happen to coincide.

### Payload bit patterns are preserved

Given a compatible target and payload values containing zero, the highest bit set, or all bits set, the adapter must forward those four-byte values without adding validation or interpreting them as negative numbers.

Any meaning assigned to those patterns by the target remains outside this contract.

### Target changes dispatch state

Given a target that changes the object’s dispatch information and then returns, the adapter must not perform an explicit post-call redispatch. It completes after the single invocation.

This does not constrain whether the target itself performs further calls.

### Invalid effective target

Given a descriptor that masks to zero or another unusable address, the adapter has no established validation or fallback behavior. Acceptance must not require a particular recoverable error, diagnostic, or return value.

## Uncertainties

The following are **documented_only**:

- **Dispatch structure meaning.** A method-table or similar dispatch structure is plausible inferred intent, not an established object-model requirement.
- **Discarded-bit meaning.** The bottom two descriptor bits may encode metadata, but no interpretation or use of them is established.
- **Concrete ABI.** Register assignment, stack layout, pointer representation details, and instruction-set mode are not provided.
- **Address arithmetic at boundaries.** The offset is 12 bytes. Portable behavior for an invalid or overflowing dispatch-base calculation is not established; wraparound must not be assumed as supported behavior.
- **Alignment and byte order.** No portable unaligned-access guarantee or byte-order rule is established by the supplied representation.
- **Concurrency.** No synchronization, atomicity, or protection against concurrent modification of dispatch storage is shown.
- **Target semantics.** The selected target’s identity, ownership rules, state changes, return behavior, and errors remain unresolved.

## Evidence

Established static evidence consists of the supplied representation of F-d64c9b93d4be: eight incoming arguments, an initial object-word read, a descriptor read at byte offset 12, a 32-bit mask clearing two low bits, one indirect invocation forwarding all arguments, and return without an explicit result.

The object-model interpretation and possible metadata role of the discarded bits are inferred intent only.

No implementation execution, runtime inspection, or tests have been performed.
