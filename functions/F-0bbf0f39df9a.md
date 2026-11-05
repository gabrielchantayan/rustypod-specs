<!-- Trusted provenance: F-0bbf0f39df9a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0bbf0f39df9a

## Purpose

F-0bbf0f39df9a conditionally prepares two related entries and submits their shared workspace. It first obtains a four-byte value, acquires a workspace, initializes descriptor bytes, advances a context counter, derives two targets from signed relative displacements, and invokes dependencies for each target.

A zero submission result enables two final context writes: a four-byte value is copied from temporary storage, then bit 0 of a context word is set. A nonzero result from either acquisition dependency or the submission dependency is returned without proceeding beyond that failure point.

“Token,” “descriptor,” “entry,” “submission,” and “publication” are descriptive roles inferred from use. Their application-level meanings are not established. The operation is not established as transactional: earlier writes and dependency effects are not explicitly rolled back on failure.

## Interface

### Arguments and result

The function accepts three 32-bit integer-shaped arguments:

| Argument | Use |
|---|---|
| Context address | Base of mutable context storage; supplied to every dependency. |
| First base address | Base for the first displacement lookup and target calculation; also an auxiliary argument for the second entry. |
| Second base address | Base for the second displacement lookup and target calculation; its original value plus four is an auxiliary argument for the first entry. |

The source represents these arguments as signed 32-bit integers, although they are also used as addresses. Signedness differs between particular address calculations, as specified below.

The return value is a 32-bit status:

- The result of F-80c2f77a86cc when nonzero.
- Otherwise, the result of F-b00cf3174ef8 when nonzero.
- Otherwise, the result of F-f7c5b7585b2d.

Zero from the last dependency enables final publication and a zero return. Every nonzero status takes the corresponding failure path; there is no separate negative-status rule.

The four-byte results of F-f45584ca1b1e are data passed to entry-population calls, not checked statuses. Results, if any, from F-dfe52f60212b do not control continuation.

### Storage layout

Context-relative accesses are:

| Offset | Width | Interpretation and use |
|---|---:|---|
| +2 | 2 bytes | Signed counter read; narrowed increment written back. |
| +12 | 4 bytes | Publication destination, written after successful submission. |
| +28 | 2 bytes | Unsigned displacement-location offset, read separately for each entry. |
| +44 | 4 bytes | Current word read and rewritten with bit 0 set after successful submission. |

The function provides two temporary storage regions:

- A four-byte token slot, whose address is passed to F-80c2f77a86cc.
- A five-word workspace totaling 20 bytes, whose address is passed to F-b00cf3174ef8 and F-f7c5b7585b2d.

Neither region is explicitly initialized before its first receiving dependency. The workspace’s first four-byte word supplies descriptor addresses. The remaining four words are not directly interpreted by this function.

Descriptor-relative direct writes are:

| Offset | Width | Value |
|---|---:|---|
| +8 | 1 byte | Zero. |
| +9 | 1 byte | One-byte narrowing of the incremented counter. |

### Preconditions

No explicit null, extent, alignment, overlap, or memory-access checks are present. Normal completion requires every executed access to be valid under the target environment.

After successful acquisition, the workspace must supply a usable first word at every point where this function reads it as a descriptor address. The token slot must contain a usable four-byte value whenever read. A dependency’s zero status alone does not establish how it satisfies these requirements.

No non-aliasing guarantee is visible. Overlap can affect later reads and the persistence of earlier writes.

## Behavior

### Token acquisition and workspace acquisition

F-80c2f77a86cc receives the context address and the address of the token slot.

If its result is nonzero, that result is returned immediately. No subsequent dependency or explicit context or descriptor write occurs.

On zero, F-b00cf3174ef8 receives:

1. The context address.
2. The four-byte value read from the token slot for this call.
3. The workspace address.

If its result is nonzero, that result is returned immediately. Descriptor initialization, counter advancement, target processing, submission, and publication are skipped. No explicit cleanup occurs.

On zero, execution continues using the workspace’s first word as an address source. The token value supplied to this acquisition call is not a guaranteed snapshot of the value eventually published: the token slot remains live temporary storage.

### Descriptor initialization and counter advancement

The following direct operations occur after both acquisitions succeed:

1. Write zero at descriptor offset +8, using the workspace’s first word as the descriptor address.
2. Read the signed 16-bit counter at context offset +2.
3. Add one in the promoted integer width and narrow the result to the signed 16-bit counter representation.
4. Write that result to context offset +2.
5. Write its one-byte narrowing at descriptor offset +9, using the workspace’s first word for that descriptor address.

The counter is read after the descriptor’s +8 write. Its value before acquisition, or even immediately before descriptor initialization, need not equal the value actually incremented. Acquisition dependencies can modify it, and the descriptor write can overlap it.

The +9 write uses the computed increment result rather than rereading the context counter. The context store itself can affect other storage if objects overlap; no stable, independently owned descriptor address is established merely by the first workspace read.

A signed 16-bit counter ranges from −32768 to 32767. Adding one after integer promotion produces a value between −32767 and 32768, so this addition does not itself overflow a signed 32-bit integer. The upper endpoint requires out-of-range signed 16-bit narrowing. On the represented target’s low-bit narrowing interpretation, 32768 becomes the stored pattern `0x8000`. Portable out-of-range signed-conversion rules are not independently established.

The one-byte descriptor field carries only the narrowed byte, not the full counter. Its represented low-eight-bit behavior maps an increment result of 256 to `0x00`.

### First displacement lookup and target

The function reads the unsigned 16-bit field at context offset +28 and converts it to a 32-bit unsigned value.

The first displacement location is:

**first base address + unsigned offset − 2, modulo 2³²**

The explicit unsigned conversion makes both the base addition and subtraction unsigned 32-bit arithmetic. Wraparound in this location calculation is established behavior, not an unresolved signed-overflow case. Whether the resulting location permits a valid two-byte read is a separate prerequisite.

A signed 16-bit displacement is read from that location. It is promoted with sign extension and added to the original first base address to produce the first target.

This target addition is signed 32-bit arithmetic in the supplied expression. It is distinct from the unsigned displacement-location calculation. Signed overflow of the target addition is not specified as a checked failure or established modulo operation.

F-f45584ca1b1e then receives:

1. Context address.
2. First target.
3. Zero.

After it returns, F-dfe52f60212b receives:

1. Context address.
2. The descriptor address read from the workspace’s first word after F-f45584ca1b1e returns.
3. Entry selector zero.
4. First target.
5. The four-byte result of F-f45584ca1b1e.
6. Original second base address plus four.
7. Four.

The descriptor argument is not required to equal the address previously used for initialization. The workspace address has escaped to a dependency, and preservation of its first word across later calls is not established.

The auxiliary addition of four uses the original second base address and signed 32-bit arithmetic. No overflow check appears.

No result from this population call is checked. Normal return permits processing of the second entry.

### Second displacement lookup and target

After both first-entry dependencies return, the function reads context offset +28 again as an unsigned 16-bit value.

The second displacement location is:

**original second base address + newly read unsigned offset − 2, modulo 2³²**

This calculation has the same unsigned 32-bit wraparound rule as the first displacement-location calculation. It does not use the auxiliary address formed by adding four.

A signed 16-bit displacement is read from the resulting location and added, with sign extension, to the original second base address. This target addition is signed 32-bit arithmetic, subject to the same unresolved overflow boundary as the first target addition.

The second lookup is live. First-entry dependencies may have changed the context offset, displacement storage, or other relevant memory. The function does not cache the first offset for reuse.

F-f45584ca1b1e receives the context address, second target, and zero.

After it returns, F-dfe52f60212b receives:

1. Context address.
2. The descriptor address then read from the workspace’s first word.
3. Entry selector one.
4. Second target.
5. The second four-byte result of F-f45584ca1b1e.
6. Original first base address.
7. Four.

This descriptor address need not equal either the initialization address or the first population call’s descriptor argument. Again, the population result is not checked.

### Submission and publication

After both population calls return, F-f7c5b7585b2d receives:

1. Context address.
2. Workspace address.
3. Zero.
4. 256.

A nonzero result is returned without the final publication writes. The function does not restore earlier values.

On zero, the following direct accesses occur in order:

1. Read the token slot after F-f7c5b7585b2d has returned and write that four-byte value to context offset +12.
2. Read the current 32-bit word at context offset +44.
3. Write that word back with bit 0 set.
4. Return zero.

The published value therefore reflects the token slot’s contents at this late read, not necessarily the earlier value passed to F-b00cf3174ef8. Dependencies have received its address; no preservation guarantee is supplied.

The status-word read also occurs after the token-field write. If those accesses overlap through unusual storage arrangements, the token write can affect the word subsequently read. The bit-setting operation preserves the other bits of the word as actually read at that step.

## Side effects

The function’s explicit writes, in execution order, are:

- Descriptor byte +8.
- Context counter at +2.
- Descriptor byte +9.
- Context four-byte field at +12, only after successful submission.
- Context word at +44, only after successful submission.

These are write events, not guarantees about final memory contents. Later overlapping writes or dependency effects can replace them.

The function contains no explicit rollback, allocation, deallocation, release, or cleanup operation. That absence does not establish that dependencies lack such effects.

The token slot and workspace are live local storage during the call. Dependencies that have received their addresses may affect later reads through retained access while those regions remain valid. This function establishes neither permission nor a valid lifetime for access after return.

Ownership of the token, descriptor, workspace contents, and associated resources is unknown. Submission does not by itself prove transfer or release.

No explicit synchronization or atomic operation appears. The final status update is a read-modify-write sequence, not an established atomic publication protocol. No preservation of globals, context storage, descriptor contents, or escaped temporary contents is assumed across unknown calls.

## Dependencies

| Opaque ID | Visible call-site contract |
|---|---|
| F-80c2f77a86cc | Receives context and writable four-byte token storage. Nonzero terminates processing. |
| F-b00cf3174ef8 | Receives context, the token slot’s value at argument preparation, and writable 20-byte workspace storage. Nonzero terminates processing. |
| F-f45584ca1b1e | Called once per target with context, target, and zero. Its four-byte result is forwarded to the corresponding population call. |
| F-dfe52f60212b | Called in selector order zero, then one. Each call receives a freshly obtained workspace-first-word descriptor argument after its preceding F-f45584ca1b1e call. No result is checked. |
| F-f7c5b7585b2d | Receives context, workspace, zero, and 256. Zero enables late token publication and status-bit setting. |

These call-site relationships do not establish internal validation, ownership, preservation, retention, exception behavior, or resource lifecycle.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not observed executions. Unless stated otherwise, accesses are valid and dependencies return normally.

### Early failures

If F-80c2f77a86cc returns a particular nonzero status, the function returns that status without any later call or explicit context or descriptor write.

If that dependency returns zero but F-b00cf3174ef8 returns nonzero, the second status is returned. No initialization, counter increment, population, submission, publication, or explicit cleanup follows.

Neither example constrains side effects inside the dependencies already called.

### Unsigned lookup wraparound

If the first base has the 32-bit pattern `0xFFFFFFF0` and the offset read is 32, the first displacement location is `0x0000000E`.

This is the defined unsigned location calculation. A valid signed 16-bit displacement read at that resulting address is still required. Subsequent signed target addition has its own range requirement.

### Signed displacement interpretation

Suppose each offset read produces 6. A displacement of −8 read at the first base plus four produces a first target eight below the first base, provided the signed addition is in range.

A displacement of 12 read at the original second base plus four produces a second target twelve above that base.

The first population call’s auxiliary address remains the original second base plus four; it is not the second target.

### Fresh offset and descriptor reads

Suppose the first offset read produces 6, and a first-entry dependency changes the field to 10. The second displacement lookup uses the original second base plus eight.

Suppose the first F-f45584ca1b1e call leaves the workspace’s first word holding address A, and the second leaves it holding address B. The population calls receive A and B respectively, provided the relevant reads are valid. Neither must match the descriptor address used during initialization.

### Counter write without persistence guarantees

Suppose the actual signed counter read, after acquisition and the descriptor +8 write, yields 255.

The function’s counter store writes 256. Under the represented low-byte narrowing, the subsequent descriptor +9 store writes `0x00`.

This does not guarantee that the context counter still contains 256 afterward. The descriptor write may overlap it, and later dependencies may modify it. A later submission failure causes no explicit restoration by this function.

### Late token publication

Suppose the value passed to F-b00cf3174ef8 is A. If the token slot contains B when read after successful submission, the function writes B to context offset +12, not a saved copy of A.

If the status word reads as `0xA4` after that token write, the following status store writes `0xA5`, and the function returns zero.

### Submission failure

If F-f7c5b7585b2d returns nonzero, that status is returned and both explicit publication writes are skipped. Earlier writes are not explicitly undone. Dependencies may nevertheless have changed any accessible fields, including the publication destinations.

## Uncertainties

The following remain documented_only:

- Application meanings of fields, selectors, constants, and descriptive object roles.
- Dependency behavior beyond the visible calls, arguments, result uses, and branches.
- Ownership, retention, allocation, release, and failure cleanup.
- Required alignment, byte order, accessible extents, and address-space conventions.
- Behavior of invalid memory accesses.
- Signed overflow in target additions, the auxiliary second-base-plus-four calculation, and other signed field-address expressions.
- Portable out-of-range signed narrowing semantics.
- Failure mechanisms other than normal dependency returns.
- Concurrency guarantees beyond the visible access order.

Unsigned 32-bit wraparound of the two displacement-location calculations is established and is not included in the signed-overflow uncertainty.

## Evidence

Established static evidence is the supplied function body and opaque dependency mapping. It supports the widths, offsets, call sequence, zero/nonzero branches, direct access order, unsigned displacement-location arithmetic, signed target arithmetic, and late reads of escaped temporary storage described above.

The descriptive roles assigned to data and operations are inferred intent, not established domain semantics. Dependency internals and platform behavior beyond the supplied expressions remain unresolved where identified.

No runtime tests have been performed. The acceptance examples are review conditions, not execution results.
