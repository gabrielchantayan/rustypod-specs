<!-- Trusted provenance: F-2bce03db826c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2bce03db826c

## Purpose

F-2bce03db826c advances a compact allocation-like state by one fixed-size, 48-byte slot. It has two distinct paths:

- If a stored linked-entry pointer is nonzero, replace that pointer with the entry’s first word and return.
- Otherwise, advance the sequential cursor, first obtaining a new block when the cursor equals its stored end.

The apparent intent is to consume either a reusable entry or the next unused slot in a block. This interpretation is inferred from the layout and operations. The established behavior is the sequence of memory accesses, dependency calls, and state updates described below. The function does not return a slot address.

## Interface

### Inputs

The function accepts two arguments:

1. **State address.** A pointer to a writable structure containing at least four consecutive 32-bit words.
2. **Mode value.** A 32-bit integer tested only for equality with zero. All nonzero values select the same behavior.

The recovered representation uses 32-bit integers and 32-bit address-bearing words. This contract does not imply compatibility with native 64-bit pointers.

### State layout

The following names describe the observed roles of the four words; they are not additional fields.

| Byte offset | Width | Role |
|---|---:|---|
| 0 | 32 bits | Address of the current block descriptor, or zero |
| 4 | 32 bits | Address of a linked reusable entry, or zero |
| 8 | 32 bits | Sequential cursor |
| 12 | 32 bits | Sequential end address |

A block descriptor is treated as three consecutive 32-bit words:

| Byte offset | Width | Role |
|---|---:|---|
| 0 | 32 bits | Previous block-descriptor address |
| 4 | 32 bits | Capacity in 48-byte slots |
| 8 | 32 bits | Payload-region address |

A linked reusable entry must expose a readable 32-bit word at its starting address. That word becomes the new linked-entry pointer. No further entry layout is established.

### Output

There is no explicit return value. Results are communicated through state and memory writes.

### Preconditions

For defined, ordinary operation:

- The state address must identify readable and writable storage for all four state words.
- A nonzero linked-entry pointer must identify a readable 32-bit successor word.
- When an existing descriptor is consulted for capacity, its capacity word must be readable.
- Dependency results used as descriptor storage must permit the three descriptor writes.
- Address calculations and the resulting cursor and end values must be meaningful in the target’s 32-bit address model.

The source performs no checks enforcing these conditions.

The straightforward block-management interpretation also requires that state, descriptor storage, and payload storage not overlap in ways that change subsequent reads. The source contains no alias validation. Where overlap exists, the ordered accesses—not the descriptive field roles—govern the result.

## Behavior

### 1. Linked-entry path takes precedence

The function first tests whether the linked-entry pointer at state offset 4 is nonzero.

If nonzero:

1. Read the first 32-bit word at the address held in that state field.
2. Write the resulting word to state offset 4.
3. Return immediately.

This path does not inspect the mode value, compare the cursor with the end, consult block capacity, or call F-871e64ddc6de.

A successor value of zero is accepted and stored. There is no separate empty-list repair or fallback to sequential allocation during this invocation.

The operation does not clear or otherwise modify the consumed entry. It also does not explicitly change the descriptor pointer, cursor, or end.

### 2. Sequential-space decision

If the linked-entry pointer is zero, the function compares the cursor at offset 8 with the end address at offset 12.

Only equality triggers a new-block operation.

If they differ, the function skips block acquisition and increases the cursor by 48 address units. It does not establish that the cursor is below the end or that at least 48 bytes remain.

Consequently, a cursor beyond the end still follows this advance-only path. A cursor less than the end by fewer than 48 bytes also advances without a bounds check. The apparent invariant that available space consists of whole 48-byte slots is a caller-maintained condition, not an enforced property.

### 3. Capacity selection when cursor equals end

When the cursor equals the end, a new capacity is selected before either dependency call.

#### Nonzero mode

For every nonzero mode value, the selected capacity is exactly one slot.

This applies even when an existing descriptor is present. Its capacity is not read for this choice.

#### Zero mode, no current descriptor

If the mode is zero and the descriptor pointer is zero, the selected capacity is 32 slots.

#### Zero mode, existing descriptor

If the mode is zero and the descriptor pointer is nonzero, read the descriptor’s capacity word as an unsigned 32-bit quantity, denoted here by \(c\).

Calculate two unsigned 32-bit candidates:

\[
A = (c + 32) \bmod 2^{32}
\]

\[
B = \left(c + \left\lfloor c/2 \right\rfloor
             + \left\lfloor c/8 \right\rfloor\right) \bmod 2^{32}
\]

The shifts underlying the divisions are unsigned right shifts. The selected capacity is the larger candidate under an unsigned comparison:

\[
n = \max_{\mathrm{unsigned}}(A,B)
\]

Equality selects the second candidate, but both then have the same numeric value.

For capacities that do not overflow either sum, this implements a minimum increment of 32 slots versus approximately 62.5% growth. That growth interpretation does not extend to wrapped arithmetic: the actual comparison is between the wrapped 32-bit candidates.

### 4. Dependency calls and descriptor construction

After choosing capacity \(n\), the function performs these operations in order:

1. Call F-871e64ddc6de with first argument 12 and second argument zero. Retain the returned address as the new descriptor address.
2. Call F-871e64ddc6de with first argument
   \[
   (48n) \bmod 2^{32}
   \]
   and second argument zero. Retain the returned value as the payload address.
3. Write the payload address to the new descriptor at offset 8.
4. Read the state’s current descriptor pointer and write it to the new descriptor at offset 0.
5. Write the selected capacity to the new descriptor at offset 4.
6. Write the new descriptor address to state offset 0.
7. Read the payload address back from the new descriptor at offset 8.
8. Write that address to the state cursor at offset 8.
9. Write that address plus the wrapped 32-bit payload-size product to the state end at offset 12.

Both dependency calls occur before the first explicit descriptor initialization write. In particular, the first returned region is not initialized before the second call.

The previous-descriptor value used for the link is read after both calls. It is not necessarily the descriptor address observed during capacity selection. If a dependency changes the state through some external mechanism, capacity can therefore derive from one descriptor while the newly linked predecessor is another.

Similarly, the payload address is re-read from descriptor storage after publishing the descriptor address into the state. Treating all these values as immutable cached inputs would lose behavior in the presence of aliasing or externally visible dependency effects.

### 5. Final cursor advance

After a new block has been installed, or immediately after finding unequal cursor and end values, the function increases the current cursor by 48.

For an ordinary successful new block with nonwrapping addresses:

- The descriptor’s payload address remains the base of the region.
- The state cursor becomes payload base plus 48.
- The state end becomes payload base plus \(48n\).

For a one-slot block, the final cursor therefore equals the end. The next invocation with no reusable entry will request another block.

The linked-entry field is not explicitly written on the sequential path.

## Side effects

### Direct memory effects

Depending on the path, the function directly performs one of the following:

- Replaces only the linked-entry pointer.
- Advances only the cursor.
- Initializes a descriptor, publishes it, replaces cursor and end, and then advances the cursor.

No payload bytes are explicitly initialized. Descriptor initialization consists of three separate 32-bit writes, in payload-address, previous-descriptor, then capacity order. Publication to the state occurs afterward.

These writes are not an atomic transaction. There is no synchronization or rollback in the supplied function.

### Allocation and ownership

The two calls to F-871e64ddc6de have an allocation-like shape, but the dependency’s actual contract is not provided.

The function retains the first result through the descriptor pointer and retains the second result through the descriptor’s payload-address field. Linking the previous descriptor appears intended to preserve access to earlier blocks.

No explicit release operation occurs. The function does not explicitly free a reusable entry, an old descriptor, or an old payload region. It also does not establish who owns returned regions, how long they remain valid, or how they must eventually be released.

Unknown callees must not be assumed to preserve globals, preserve this state, or avoid releasing other storage.

### Failure behavior

There is no explicit error return, null-result check, capacity-overflow check, or recovery path.

If a descriptor result is unusable, subsequent writes can access invalid storage. If a payload result is zero or otherwise unusable, that value is still recorded and used in cursor/end construction; this function does not dereference the payload itself on the shown path.

A failure or nonlocal exit during either dependency call can prevent later operations. An access failure during descriptor or state writes can leave partially applied changes. The exact exception, signal, or termination behavior is not established.

## Dependencies

### F-871e64ddc6de

This is the only explicit callee.

It is called exactly twice when all of the following hold:

- The linked-entry pointer is zero.
- The cursor equals the end.
- Execution proceeds through both calls without interruption.

The calls are ordered:

| Call | First argument | Second argument | Use of result |
|---|---:|---:|---|
| First | 12 | 0 | Address used for a three-word descriptor |
| Second | Wrapped 32-bit product of capacity and 48 | 0 | Value stored as the payload-region address |

The dependency’s formal parameter types, allocation guarantees, initialization policy, failure conventions, ownership rules, alignment guarantees, and global side effects are unknown. The displayed call arguments alone do not establish them.

No dependency call occurs on the linked-entry path or the unequal-cursor/end path.

## Acceptance examples

These examples are conditional static expectations, not reported runtime results. Unless stated otherwise, assume valid, nonoverlapping storage, normal dependency returns, no dependency mutation of the state, and nonwrapping address calculations.

### Reusable entry takes precedence

Given a nonzero linked-entry pointer whose first word contains another entry address:

- The linked-entry field becomes that successor address.
- Cursor and end are unchanged by this function.
- No dependency call occurs, even if cursor equals end.
- Zero and nonzero mode values produce the same result.

If the successor word is zero, the field becomes zero and the function returns without obtaining a block.

### Existing sequential space

Given a zero linked-entry pointer, cursor 4096, and end 4192:

- Cursor becomes 4144.
- End and descriptor pointer remain unchanged.
- No dependency call occurs.

If the cursor is instead 4240 with the same end, the function still advances it to 4288. It does not reject the already-out-of-range state.

### Initial ordinary block

Given zero mode, zero descriptor pointer, zero linked-entry pointer, and equal cursor and end:

- Selected capacity is 32.
- Dependency first arguments are 12, then 1536.
- The new descriptor records a zero predecessor.
- With payload base \(P\), final cursor is \(P+48\) and end is \(P+1536\).

### Forced single-slot block

Given any nonzero mode and an exhausted state with no reusable entry:

- Selected capacity is one.
- Dependency first arguments are 12, then 48.
- The prior descriptor, if any, is linked without consulting its capacity.
- Final cursor and end both equal payload base plus 48.

### Ordinary growth

For zero mode and an existing capacity of 32:

- The additive candidate is 64.
- The proportional candidate is 52.
- Selected capacity is 64.
- Payload-size argument is 3072.

For an existing capacity of 128:

- The additive candidate is 160.
- The proportional candidate is 208.
- Selected capacity is 208.
- Payload-size argument is 9984.

### Capacity and size wrapping

For zero mode and an existing unsigned capacity of hexadecimal 80000000:

- The additive candidate is hexadecimal 80000020.
- The proportional candidate is hexadecimal D0000000.
- The selected capacity is hexadecimal D0000000.
- Its 32-bit product with 48 is zero.

Accordingly, the second dependency receives zero as its first argument. If it returns normally with usable descriptor storage available, the installed end initially equals the payload value, while the final cursor is advanced by 48. There is no overflow rejection or forced minimum payload size.

## Uncertainties

### Established static evidence

The four-word state accesses, linked-entry precedence, equality-only exhaustion check, capacity rules, unsigned capacity arithmetic, two dependency calls, descriptor write order, and final 48-byte advance are directly supported by the supplied function representation.

### Inferred intent

The terms “reusable entry,” “block,” “capacity,” and “payload” describe a likely allocation-management design. They do not establish an external API promise that this function returns, initializes, or transfers ownership of a usable object.

### documented_only

The following remain unknown:

- The dependency’s implementation and complete contract.
- Whether callers guarantee slot-aligned cursor/end differences and bounded capacities.
- Whether callers depend on the consumed slot’s address through another interface.
- Whether descriptor and payload regions can overlap state or each other.
- Whether dependency calls can mutate this state or related memory.
- Synchronization requirements and permitted concurrent observers.
- Exact target behavior for invalid addresses and cursor arithmetic crossing the address range.

Unsigned capacity calculations and the capacity-times-48 product are explicitly 32-bit wrapping operations. The final cursor addition is presented through a signed integer expression in the recovered representation. Its precise overflow behavior cannot be promoted into a portable signed-arithmetic guarantee; ordinary acceptance examples avoid that boundary.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping. No dependency implementation, caller behavior, allocator specification, or runtime observations were supplied.

No runtime tests have been performed. The acceptance examples are arithmetic and control-flow consequences of the supplied representation, conditional on their stated memory and dependency assumptions.
