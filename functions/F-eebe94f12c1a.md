<!-- Trusted provenance: F-eebe94f12c1a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-eebe94f12c1a

## Purpose

F-eebe94f12c1a conditionally emits one 32-bit value from a stateful, stack-like traversal and advances the associated traversal state.

The established behavior is narrower than a complete traversal abstraction:

- A null output address prevents all state inspection and returns zero.
- A nonnull output address with a zero word at state offset 4 also returns zero.
- Otherwise, the function writes one output word, updates a signed traversal position, and either prepares a deeper traversal state or removes exhausted states. It then returns one, provided the invoked dependencies return normally and the required memory accesses are valid.

The arrangement suggests a depth-first traversal over objects with child-like entries. That interpretation is **inferred intent**, not an established guarantee about the objects, dependency implementations, or overall traversal order.

A successful call can exhaust the traversal. In that case it still returns one because it has already written an output. Exhaustion is observed by a later call through the zero-word entry check.

## Interface

### Arguments and result

The interface has four arguments and a 32-bit result.

| Argument | Established use |
|---|---|
| State address | Addresses the primary state and an embedded secondary structure at byte offset 16. |
| Output address | Optional address of a writable 32-bit output word. Its nullness controls whether any state is inspected. |
| Opaque third argument | Not read, passed onward, or otherwise used by the visible function. |
| Fourth argument | A pointer-shaped value copied into a local slot on the active path. Its incoming value is never dereferenced or passed to a dependency by the visible function. |

The result is exactly the 32-bit value zero or one on normal return. It is not an error code returned by a dependency.

### Widths and layout

The supplied representation uses 32-bit integers and pointer-bearing words. Offsets below are byte offsets.

The state exposes at least:

- **Offset 4:** a 32-bit word tested against zero to determine whether active state exists.
- **Offset 16:** the beginning of an embedded secondary structure used by position-access, removal, and indirect-call operations.

An active object exposes at least:

- **Offset 4:** an embedded receiver whose first word supplies a dispatch-table address.
- **Offset 8:** a 32-bit signed bound used in traversal-position comparisons.

The receiver’s dispatch table has a callable entry at **offset 64**. Calling it with that receiver produces an address from which the function reads one 32-bit word.

The secondary structure at state offset 16 has a dispatch-table address in its first word. Its callable entry at **offset 28** is used to submit the address of a local pointer slot.

These are access requirements, not proof of complete structure definitions. Fields not accessed here, dispatch signatures beyond the observed arguments, and allocation provenance remain unknown.

### Preconditions for defined completion

When the output address is nonnull, the state must permit the initial 32-bit read at offset 4. If that word is nonzero, further requirements apply:

- The output address must permit a 32-bit write.
- Accessor dependencies must return addresses valid for the subsequent word accesses.
- Active-object addresses, embedded receivers, dispatch tables, and invoked entries must be valid.
- Position storage must permit both reading and writing.
- An allocation used on the expansion path must return storage permitting a 32-bit write.
- The dependencies and indirect calls must return normally for this function to reach its ordinary result.

The visible function does not enforce these requirements. It does not check returned pointers, allocation failure, or dispatch validity.

No non-aliasing guarantee is established. Overlap among output storage, state, position storage, object fields, or dependency-returned storage can change later reads and control flow.

## Behavior

### 1. Entry gating

The output address is examined first.

If it is null, the function returns zero without reading the state. Thus, on this path, even the word at state offset 4 is not accessed.

If the output address is nonnull, the function reads the word at state offset 4:

- A zero word causes an immediate zero result, without writing the output.
- A nonzero word enters the active path.

The test is for equality with zero. The visible behavior does not require that the word be a positive count.

### 2. Acquire the active object and its position

On the active path, the fourth argument’s value is copied into a local pointer slot. This does not read memory through that argument.

The function then performs these operations in order:

1. Calls F-82c63e47892d with the state address and the 32-bit value `0x7FFFFFFF`.
2. Reads one word through the returned address and retains it as the active-object address.
3. Calls F-c0e14e509280 with the address at state offset 16 and the same value `0x7FFFFFFF`.
4. Retains the returned address as writable signed-position storage.
5. Establishes the receiver address as active-object offset 4.

The selector value is established; its meaning is not. It must not be described as an index, timeout, or capacity without evidence from the dependencies.

### 3. Write the output

The function reads the current signed position.

If the position equals negative one, it writes the retained active-object address directly to the output word.

For every other position value, it:

1. Reads the receiver’s dispatch-table address.
2. Calls the entry at dispatch-table offset 64 with the receiver address.
3. Reads one word through the returned address.
4. Writes that word to the output.

Consequently, the negative-one case emits the active object itself; other cases emit a value selected by the receiver operation. The visible function does not pass the position explicitly to that operation.

There is exactly one explicit output-word write on a normally completing active path. Dependency side effects may cause additional writes, including writes to aliased storage.

### 4. Increment the position and choose advancement

After writing the output, the function reads the current word from the retained position address, adds one, and writes it back there.

This read occurs after the output write and after any output-selection indirect call. It is therefore not necessarily a reuse of the value that selected the output branch. An aliasing output write or an indirect-call side effect may have changed the position.

The function then reads the updated signed position and compares it with the signed word at retained active-object offset 8.

- If the position is **strictly less than** that bound, it takes the candidate-expansion path.
- Otherwise, it takes the removal path.

The active-object address used here is the one retained before output selection. It is not reacquired after intervening calls. Its bound field, however, is read at this later comparison.

### 5. Candidate expansion

When the updated position is less than the active-object bound, the function invokes the receiver’s offset-64 dispatch entry again. This is a fresh dispatch and call, not reuse of the earlier result.

It then:

1. Reads a word through the indirect call’s returned address.
2. Treats that word as an object address.
3. Reads the signed word at that object’s offset 8.
4. Tests whether that word is strictly greater than zero.

If the tested word is zero or negative, no expansion dependency or allocation is invoked. The function proceeds to return one.

If the tested word is positive, the following operations occur in order:

1. Call F-dfabedd4a3d1 with the state address.
2. Call F-8157a180d901 with the allocation-size argument four.
3. Replace the local pointer slot with the allocation result.
4. Write the 32-bit pattern `0xFFFFFFFF` into the returned storage.
5. Read the dispatch-table address from the secondary structure at state offset 16.
6. Invoke its offset-28 entry with the secondary-structure address and the address of the local pointer slot.

The stored pattern represents negative one when interpreted as a signed 32-bit integer. It is consistent with the initial-position sentinel used during output selection.

The visible function does not explicitly free this allocation. Whether the indirect operation copies the pointer, takes ownership, modifies the local slot, or retains anything is unknown.

### 6. Removal and ancestor inspection

If the updated position is greater than or equal to the active-object bound, the function begins a removal loop. The first removal pair is unconditional once this branch is selected.

Each iteration performs:

1. Call F-ac0b2c25a1a2 with the state address and `0x7FFFFFFF`.
2. Call F-519fa94838b5 with the secondary-structure address and `0x7FFFFFFF`.
3. Read the state word at offset 4.
4. If that word is zero, stop the loop.
5. Otherwise, reacquire an active-object address through F-82c63e47892d and a position address through F-c0e14e509280, in that order.
6. Read the reacquired position and the signed bound at the reacquired object’s offset 8.
7. Repeat the removal loop if the position is **less than or equal to** the bound.

If the reacquired position is greater than the bound, removal stops.

The loop’s comparison is deliberately distinct from the earlier advancement comparison:

- Initial removal is selected when updated position **≥ bound**.
- Further removal occurs when reacquired position **≤ bound**.

The visible evidence does not justify reversing either comparison to fit a conventional traversal model.

Once this loop finishes, the function returns one. It does not emit another output or attempt candidate expansion for the surviving state during that call.

## Side effects

### Observable ordering

On an active call, the output write precedes the position increment. Both precede any expansion preparation, allocation, or removal.

During removal, the primary removal dependency runs before the secondary removal dependency. The state’s offset-4 word is inspected only after both return.

During expansion, F-dfabedd4a3d1 runs before allocation. The allocated word is initialized before the secondary indirect operation receives the local pointer-slot address.

These orderings matter when dependencies inspect shared state or when accessed storage aliases.

### Memory and ownership

The function explicitly writes to:

- The output word.
- The retained position word.
- The four-byte allocation, if expansion is taken.
- Its own local pointer slot.

Additional memory and global-state changes may occur inside dependencies or indirect calls. No preservation guarantee for globals or shared state is established.

The returned output is a 32-bit address-like value. This function does not copy the referenced object and does not explicitly establish ownership transfer, lifetime extension, or permission to release it.

The local pointer-slot address submitted during expansion has only call-local storage duration. Whether the receiving operation uses it synchronously or retains it is undocumented.

### Errors and arithmetic

There is no visible recovery path for a null accessor result, null allocation result, invalid receiver, or inaccessible output. A zero return means only that the entry gate rejected the call; it does not report these failures.

Position and bound comparisons are signed 32-bit comparisons. The increment is a 32-bit operation in the supplied representation. Behavior at signed maximum, including whether the execution environment wraps or has another overflow rule, is not established by this evidence. A portable contract must not silently assume wraparound.

## Dependencies

| Opaque ID or indirect operation | Visible role and limits |
|---|---|
| F-82c63e47892d | Receives the state and `0x7FFFFFFF`; returns an address whose first word is read as an active-object address. |
| F-c0e14e509280 | Receives the secondary-structure address and `0x7FFFFFFF`; returns position storage. |
| Receiver dispatch, offset 64 | Receives the active object’s embedded receiver; returns an address whose first word is read. Can be called twice in one invocation. |
| F-dfabedd4a3d1 | Receives the state on the positive-bound expansion path. Its internal transition is unknown. |
| F-8157a180d901 | Receives four; its result is used as storage for one 32-bit sentinel. Allocation and failure semantics are otherwise unknown. |
| Secondary dispatch, offset 28 | Receives the secondary structure and a pointer-slot address after sentinel initialization. |
| F-ac0b2c25a1a2 | First dependency in each removal iteration; receives the state and `0x7FFFFFFF`. |
| F-519fa94838b5 | Second dependency in each removal iteration; receives the secondary structure and `0x7FFFFFFF`. |

Labels such as “removal” and “allocation” describe visible usage or inferred intent. They do not establish undocumented internal behavior.

## Acceptance examples

These are conditional acceptance scenarios derived from static evidence, not executed tests. Unless stated otherwise, assume valid storage, normally returning dependencies, no relevant aliasing, and no intervening mutation of the named values.

1. **Null output:** With a null output address, the result is zero. There is no state read or dependency invocation, regardless of the state-address value.

2. **Inactive state:** With a nonnull output address and zero at state offset 4, the result is zero and the output word is unchanged by this function.

3. **Initial sentinel with zero bound:** With position negative one and active-object bound zero, the active-object address is written to the output. The position becomes zero, and the first removal pair runs. The function returns one even if that pair empties the state.

4. **Initial sentinel with a remaining candidate:** With position negative one and active-object bound two, the output is the active-object address and the updated position is zero. The receiver operation runs once for candidate inspection. If the selected object’s signed bound is zero, no expansion allocation occurs.

5. **Ordinary output plus expansion:** With initial position zero and active-object bound three, the first receiver operation supplies the output word. After the position becomes one, a second receiver operation supplies the candidate for expansion inspection. If that candidate’s signed bound is positive, preparation, four-byte allocation, sentinel initialization, and secondary submission occur in the stated order. The two receiver calls need not return the same address or word.

6. **Removal boundary:** After the first removal pair, suppose the state remains nonzero and the reacquired position equals its object’s bound. Another removal pair occurs. Equality does not stop removal.

7. **Removal termination with remaining state:** After a removal pair, suppose the state remains nonzero and the reacquired position is greater than its bound. Removal stops and the function returns one without another output write.

8. **Negative candidate bound:** On the candidate-expansion path, a selected bound of negative one does not trigger expansion. The positive test is signed and strict.

## Uncertainties

The following are **documented_only**:

- Complete object and state layouts, alignment requirements, and address-space constraints.
- The semantic meaning of `0x7FFFFFFF`.
- Dependency mutation of globals, positions, bounds, dispatch tables, or object lifetimes.
- Whether the receiver operation depends on implicit shared position state.
- Allocation failure behavior and ownership of submitted storage.
- Whether either removal dependency releases memory.
- Exceptional exits, nonlocal transfers, concurrency behavior, and synchronization.
- Signed-overflow behavior at the position increment.
- Any higher-level invariant explaining the differing advancement and removal comparisons.

A depth-first iterator is plausible intent. The established contract is the access sequence, comparisons, writes, calls, and normal-return values above, not a guarantee that every reachable object is visited exactly once.

## Evidence

This contract is based solely on static inspection of the supplied function representation and the supplied opaque dependency mapping.

Established observations include the null-output short circuit, state-offset gate, 32-bit word accesses, dispatch offsets, output-before-increment ordering, distinct signed comparisons, expansion call order, removal-loop order, and binary normal-return values.

No dependency bodies, runtime traces, allocation contracts, or external structure definitions were supplied. No runtime tests have been performed.
