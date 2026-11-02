<!-- Trusted provenance: F-fc1bdf7565c0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fc1bdf7565c0

## Purpose

F-fc1bdf7565c0 clears a caller-supplied byte, then selects one of two dependencies using an unsigned 32-bit field in a caller-supplied object. One dispatch path supplies an address within that object; the other supplies a 32-bit value read from a separate field. Five additional 32-bit arguments are forwarded unchanged.

The visible behavior establishes a dispatcher, not the meaning of the operation being dispatched. The meanings of the object, its discriminator, the forwarded arguments, and the cleared byte are unknown. In particular, the byte cannot be identified as a success indicator merely because this function sets it to zero.

The two discriminator comparisons are separate reads. That distinction is part of the behavioral contract: the first read decides whether to take the lower-valued path; only if that path is not taken does a second read decide between the other dependency and a non-returning dependency.

## Interface

### Inputs

The function receives seven arguments, in this order:

| Position | Contract-visible interpretation |
|---|---|
| 1 | An object base address used to derive two field locations and one interior address. |
| 2–6 | Five opaque 32-bit values, forwarded in their original order to the selected returning dependency. |
| 7 | An address at which this function writes one zero-valued byte. |

The first argument is represented by a 32-bit integer in the supplied material and is used as an address. A portable pointer type or host-independent address model is not established. The five forwarded values are each four bytes wide; their signedness, internal layout, and possible interpretation as addresses are not established.

### Relevant object layout

Offsets below are byte offsets from the first argument:

| Offset | Width or use | Meaning established here |
|---|---|---|
| +8 | 4-byte unsigned read | Dispatch discriminator, potentially read twice. |
| +16 | 4-byte read, conditional | Opaque first argument for F-8cf06e604c65. |
| +20 | Address formation, conditional | Interior address supplied as the first argument to F-7b23d6f64a0d. |

No contents at offset +20 are read directly by F-fc1bdf7565c0. Forming and passing that address is distinct from reading the storage there. What storage must exist at and beyond that address depends on F-7b23d6f64a0d.

The visible offsets do not establish the full object size, any fields in the gaps, alignment requirements beyond those of the execution environment, or the meaning of either field.

### Output and return

There is no explicit return value. On either ordinary dispatch path, the selected dependency is called and the wrapper returns if that dependency returns normally.

The write through argument 7 is the only direct output write visible in this function. It occurs before dispatch. Its final value at function return is not guaranteed to remain zero: subsequent dependency effects and aliasing are not characterized.

### Preconditions

For defined use in the represented environment:

- Argument 7 must permit a one-byte write when the function begins.
- The object location at offset +8 must permit a four-byte read after that write.
- If the first discriminator read is at least 2, the same location must permit the second four-byte read.
- If the second discriminator read equals 2, the location at offset +16 must permit a four-byte read.
- If the first discriminator read is below 2, deriving the interior address at offset +20 and passing it to F-7b23d6f64a0d must be valid under the applicable address model.
- The selected dependency’s own argument and storage requirements must also hold.

No null checks, range checks, object-size checks, or explicit recovery from invalid memory access are visible. Address-offset calculations must not be assumed to make invalid or overflowing addresses safe.

## Behavior

### Initial observable write

The function first stores zero into the single byte addressed by argument 7.

This happens before any discriminator read and before any dependency call. The write is not conditional on the discriminator value. There is no visible restoration of the previous byte value.

Because no non-aliasing requirement is established, this write can affect subsequent dispatch. For example, if argument 7 addresses a byte of the discriminator, the discriminator is read after that byte has been cleared. If it addresses a byte of the field at offset +16, a later read of that field observes whatever value results from the write, subject to any other changes.

The byte-to-word effect of such aliasing depends on byte order and the byte selected. Neither should be guessed.

### Lower-valued dispatch

After clearing the byte, the function reads the four-byte discriminator at offset +8 and interprets it as unsigned.

If this first read yields 0 or 1:

- F-7b23d6f64a0d is called.
- Its first argument is the object’s interior address at offset +20.
- Its remaining five arguments are arguments 2–6, unchanged and in order.
- No second discriminator read or read of the field at offset +16 is performed by this wrapper.
- If the dependency returns normally, the wrapper returns.

Both values take the same visible path. The wrapper does not pass the discriminator as a separate argument, although the dependency may have access to related storage through its supplied address or other state.

### Dispatch after a first value of at least 2

If the first discriminator read is not below 2, the function reads the discriminator again from offset +8.

If the second read equals 2:

- The function reads four bytes from offset +16.
- That value becomes the first argument to F-8cf06e604c65.
- Arguments 2–6 become its remaining five arguments, unchanged and in order.
- If the dependency returns normally, the wrapper returns.

The first argument in this branch is the stored value, not the address of the field containing it. Whether the value itself denotes an address is unknown.

If the second read is anything other than 2, the function calls F-a817d8d58c10 with no explicit arguments. The supplied material identifies that dependency as non-returning. This path has no established normal return.

### Consequences of the two reads

For a stable discriminator, the visible dispatch rule is:

| Discriminator value after the initial byte write | Outcome |
|---|---|
| 0 or 1 | Call F-7b23d6f64a0d with the interior address. |
| 2 | Read offset +16 and call F-8cf06e604c65 with that value. |
| 3 through 4,294,967,295 | Call F-a817d8d58c10 on the non-returning path. |

That table assumes the value does not change between reads.

The actual decision when two reads occur depends on both observations. A first value of 3 followed by a second value of 2 selects F-8cf06e604c65. A first value of 2 followed by a second value of 1 selects the non-returning dependency; it does not fall back to the lower-valued dispatch.

This describes the visible read sequence, not a guarantee of safe concurrent access. No synchronization, atomic access contract, or memory-ordering mechanism is established.

### Width and arithmetic constraints

The discriminator comparison is unsigned and 32-bit. A bit pattern that would denote a negative signed integer is therefore not below 2; its treatment on the second read still depends on whether that read equals exactly 2.

The offsets +8, +16, and +20 are address displacements. No multiplication, count-based traversal, narrowing of the forwarded values, or numerical conversion of the loaded field at +16 is visible.

## Side effects

The directly established effects are:

1. A one-byte zero write through argument 7.
2. One discriminator read, or two if the first value is at least 2.
3. A conditional four-byte read at offset +16.
4. A call to the dependency selected by those reads.

The wrapper performs no visible allocation, deallocation, resource acquisition, or resource release. This does not establish that the dependencies refrain from such operations.

Ownership is not visibly transferred or relinquished by the wrapper. Passing an interior address or an opaque value to a dependency does not, by itself, prove whether that dependency borrows, retains, modifies, or releases any associated storage.

No guarantee is established that any dependency preserves global state, the object, the byte addressed by argument 7, or memory reachable through the forwarded arguments. Consequently, the wrapper’s initial zero write is guaranteed as an ordered local effect, not as a guaranteed final output value.

The non-returning path still follows the initial byte write. It does not bypass that write, and no rollback is visible.

## Dependencies

| Opaque function ID | Invocation condition | Explicit arguments | Established return behavior |
|---|---|---|---|
| F-7b23d6f64a0d | First discriminator read is below 2 | Object base plus 20 bytes, then the five forwarded 32-bit values | Wrapper returns if the dependency returns normally. |
| F-8cf06e604c65 | First read is at least 2 and second read is exactly 2 | Four-byte value read at object offset +16, then the five forwarded 32-bit values | Wrapper returns if the dependency returns normally. |
| F-a817d8d58c10 | First read is at least 2 and second read is not 2 | No explicit arguments | Identified as non-returning in the supplied material. |

The wrapper neither uses nor forwards an explicit return value from these calls. Dependency internals, error conventions, memory effects, ownership rules, and nonlocal control effects are unknown.

For F-a817d8d58c10, termination, exception transfer, process abort, and indefinite execution are not distinguishable from the available evidence. If it unexpectedly returns despite its stated non-returning property, the resulting execution is outside the established contract.

## Acceptance examples

These examples are conditional consequences of the static evidence, not reports of executed tests. Unless stated otherwise, assume valid storage, no relevant aliasing, stable field values, and dependencies satisfying their own preconditions.

### Stable lower-valued discriminator

Given a discriminator of 0, the function first clears the output byte and then calls F-7b23d6f64a0d with the address at object offset +20 and the five original forwarded values. It does not directly read offset +16.

A discriminator of 1 produces the same visible selection.

### Stable discriminator of 2

Given a discriminator of 2 and a four-byte value of `0x12345678` at offset +16, the function clears the output byte, reads the discriminator twice, reads the stored value, and calls F-8cf06e604c65 with `0x12345678` as its first argument. It does not substitute the address of offset +16.

If that dependency returns normally, the wrapper returns without an explicit result.

### Stable unsupported value

Given a discriminator of 3, the function clears the output byte and, after both discriminator reads, calls F-a817d8d58c10. No normal return is established.

The same selection holds for a stable discriminator of `0xFFFFFFFF`; unsigned interpretation does not classify that value as below 2.

### Different observations across reads

If a permitted execution produces a first discriminator observation of 5 and a second observation of 2, F-8cf06e604c65 is selected.

If the first observation is 2 and the second is 0, F-a817d8d58c10 is selected. The lower-valued path is decided only by the first read.

These examples do not establish that unsynchronized concurrent mutation is permitted.

### Aliasing of the cleared byte

Suppose argument 7 aliases a discriminator byte and clearing it changes the subsequently read discriminator to 1. The first read selects F-7b23d6f64a0d. Dispatch is not based on the discriminator’s value before entry.

Suppose instead that argument 7 aliases a byte of the field at offset +16 and the discriminator selects F-8cf06e604c65. The forwarded field value reflects the earlier byte write, absent other changes. Its exact numeric value depends on the aliased byte and the environment’s byte order.

### Dependency overwrites the byte

If a selected dependency writes a nonzero value to the same byte that the wrapper cleared, the final byte may be nonzero. Such an outcome does not contradict the wrapper’s established initial write.

## Uncertainties

The following remain **documented_only**:

- The semantic purpose of the object, discriminator, cleared byte, and five forwarded values.
- The complete object layout and the storage extent required by either dispatch dependency.
- Byte order, precise address arithmetic semantics, and platform-specific alignment requirements.
- Whether the two discriminator reads can observe different values in a valid execution.
- Any synchronization or atomicity guarantees supplied outside this function.
- Dependency ownership, retention, mutation, allocation, release, and global-state effects.
- The mechanism and externally visible consequences of the non-returning dependency.
- Behavior after invalid memory access or violation of dependency preconditions.
- Any stronger postcondition on the cleared byte after a dependency executes.

An inferred intent is to distinguish two supported dispatch forms and reject other stable discriminator values. That interpretation is not needed to specify the observed branches and does not establish application-level validity rules.

## Evidence

The supplied static material establishes the argument widths, one-byte initial store, unsigned discriminator comparison, separate second discriminator read, field offsets, conditional field load, dependency argument order, and absence of an explicit return value.

The non-returning property of F-a817d8d58c10 is stated in the supplied material; its implementation and mechanism are not available.

No runtime tests have been performed. The acceptance examples express conditional expectations derived from the visible behavior, not measured outcomes.
