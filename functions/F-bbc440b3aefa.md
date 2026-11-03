<!-- Trusted provenance: F-bbc440b3aefa; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bbc440b3aefa

## Purpose

F-bbc440b3aefa updates a buffer-bearing object with a requested logical length and, when a source address is supplied, invokes an external transfer operation and writes a trailing zero byte.

The established behavior is a storage-selection and state-update protocol, not a fully established string-copy contract. The routine:

- Optionally obtains a length from a dependency.
- Requests replacement storage when the recorded length is smaller than the effective length or the stored buffer address is zero.
- Returns failure when that storage request returns zero.
- Otherwise records the effective length.
- When the source address is nonzero, invokes a transfer dependency and subsequently writes one zero byte at the effective-length offset.

Interpreting the object as a mutable string is inferred intent. The identities and implementations of the length, allocation, replacement, and transfer operations are not established here.

## Interface

### Inputs

The routine takes three arguments:

| Argument | Representation | Meaning established by use |
|---|---|---|
| Object address | Address of a structure containing 32-bit fields | Supplies the recorded length and stored buffer address; receives updates. |
| Source address | 32-bit address-valued argument | Zero disables the transfer and trailing-zero write. Nonzero is passed to the length dependency when automatic length selection is requested and to the transfer dependency on the successful path. |
| Requested length | Signed 32-bit integer | A negative value selects automatic length acquisition. A nonnegative value is used directly. |

The relevant object layout is:

| Byte offset | Width | Role |
|---|---|---|
| 0 | 32 bits | Signed recorded length, used in the storage-size comparison and overwritten on the successful path. |
| 4 | 32 bits | No direct access by this routine. |
| 8 | 32 bits | Buffer address, read and sometimes replaced. |

The source representation uses integer-valued addresses and 32-bit object fields. This contract describes that represented interface; it does not establish a portable host-language ABI or a wider-address variant.

### Output

The routine returns a 32-bit value:

- **0:** automatic length was requested with a zero source address, or a storage dependency returned zero.
- **1:** the routine reached the end after its state update and any requested transfer and trailing-zero write.

Return value 1 is not independent proof that the external transfer copied the intended bytes. The transfer dependency’s result, if any, is not inspected.

Faults, nonreturning dependency calls, and invalid-memory behavior are outside these two normal return outcomes.

### Preconditions

For a defined, normally completing invocation:

- The object must be readable and writable at every field access actually reached.
- Addresses supplied to dependencies must satisfy those dependencies’ contracts.
- Any reached trailing-zero write must target writable memory.
- The effective length and its increment must be meaningful for the selected storage dependency.
- Any transfer count must be valid under the transfer dependency’s count interpretation.

The initial zero-source, negative-request case is exceptional: it returns before directly accessing the object or calling a dependency. Consequently, that path does not itself require a dereferenceable object address.

## Behavior

### 1. Effective-length selection

When the requested length is nonnegative, it becomes the effective length without a length dependency call.

When the requested length is negative:

1. A zero source address causes an immediate return of 0.
2. Otherwise, F-1cb585febd84 is called with the source address.
3. Its returned value becomes the effective length.

There is no second sign check or validation after F-1cb585febd84 returns. A negative result therefore continues into the comparison, possible storage request, length-field update, and possible transfer path.

No direct object-field read occurs before this optional dependency call. If that dependency changes the object through an alias or shared state, subsequent object reads observe the state then present.

### 2. Storage-selection condition

The routine first reads the object’s recorded length and compares it with the effective length using signed ordering.

A storage request is required if either:

- The recorded length is less than the effective length; or
- The stored buffer address is zero.

The second condition is short-circuited: when the length comparison already requires storage, the buffer need not be read to evaluate that condition. The buffer address is nevertheless read inside the storage-request path to choose the dependency.

A sufficiently large recorded length does not avoid a storage request when the buffer address is zero.

Conversely, a nonzero buffer address and a recorded length greater than or equal to the effective length avoid a storage request. The routine performs no independent validation that the address actually refers to enough memory.

### 3. Storage request and returned-address handling

On entering the storage-request path, the routine saves the current buffer address.

- If the saved address is zero, it calls F-43da6b23bd44 with the effective length plus one and two zero arguments.
- If the saved address is nonzero, it calls F-8b998a07b5e1 with that address, the effective length plus one, and two zero arguments.

The extra unit is consistent with space for the trailing zero byte, but the unit and allocation semantics of these dependencies are not independently established.

After the selected dependency returns, the routine writes its returned address into the object’s buffer field.

If that returned address is zero, the routine then writes the previously saved buffer address back into the same field and returns 0. Thus the failure path includes two ordered writes to the buffer field: first zero, then the saved address.

That restoration is limited:

- It restores an address value, not necessarily live storage.
- It does not establish that the dependency retained the original allocation.
- It does not roll back dependency changes to memory, other object fields, globals, or ownership.
- It does not undo a dependency’s modification of the recorded-length field.

If the returned address is nonzero, processing continues using the object containing that stored result.

### 4. Logical-length update

After either bypassing storage acquisition or receiving a nonzero storage result, the routine writes the effective length to the object’s field at offset 0.

This write occurs before the optional transfer.

The field is used both as the earlier storage-comparison value and as the new logical length. There is no directly accessed, separately maintained capacity field. In particular, reducing the recorded length can cause a later larger request to seek replacement storage even if the underlying allocation was not physically reduced.

### 5. Optional transfer and termination

If the source address is zero, the routine does not invoke the transfer dependency and does not write a trailing zero. It returns 1 after the length update.

If the source address is nonzero:

1. The routine reads the current buffer address from the object.
2. It invokes F-unmapped-01 with that destination address, the source address, and the effective length.
3. After the call returns, it reads the object’s buffer address again.
4. It writes one zero-valued byte at that freshly read address plus the effective length.
5. It returns 1.

The second buffer read is significant. The trailing-zero write is not unconditionally directed to the destination address supplied to F-unmapped-01. If the dependency changes the object’s buffer field, the write uses the changed address.

Likewise, the routine does not reapply the recorded-length update after the transfer call. A dependency modification of that field may remain visible at return.

### 6. Integer and address boundaries

The routine contains no explicit bounds checks beyond the signed recorded-length comparison.

In particular:

- A storage request computes effective length plus one.
- At the maximum signed 32-bit effective length, the increment exceeds the signed 32-bit range.
- The supplied representation does not resolve whether that case follows machine-width wraparound or some other language-level arithmetic interpretation.
- A negative effective length returned by F-1cb585febd84 is not rejected.
- The transfer dependency’s interpretation of a negative count is unknown.
- The trailing-zero address is computed from the buffer address and effective length without an explicit overflow or range check.

A defined-behavior acceptance claim must therefore constrain these arithmetic operations and dependency arguments rather than assume safe handling at their boundaries.

## Side effects

### Direct object writes

The routine may directly:

- Store the storage dependency’s returned address at offset 8.
- Restore the previously saved address there after a zero result.
- Store the effective length at offset 0.

There is no direct write to the field at offset 4.

“No direct write” does not imply preservation against dependency effects or aliasing.

### Direct buffer write

When the source is nonzero and the transfer call returns, the routine directly writes one zero byte at the current buffer address plus the effective length.

With a zero source address, this direct buffer write is skipped even if storage was just acquired.

### Dependency effects and ownership

No explicit release operation appears in this routine.

The ownership effects of F-43da6b23bd44 and F-8b998a07b5e1 are unknown. In particular, the latter’s apparent replacement role does not establish whether it frees, retains, moves, or modifies the old storage.

No dependency is assumed to preserve globals, object fields reachable through aliases, or source memory. There is no established transactional or thread-safe guarantee.

If the object overlaps destination storage, the transfer or trailing-zero write may overwrite object fields. The routine provides no alias protection.

## Dependencies

| Opaque ID | Established invocation | Unestablished behavior |
|---|---|---|
| F-1cb585febd84 | Called with the nonzero source address when the requested length is negative. Its result is used as the effective signed length. | How length is determined; required termination; read bounds; error signaling; side effects. |
| F-43da6b23bd44 | Called when storage is required and the saved buffer address is zero. Receives effective length plus one, zero, zero. Its result is treated as an address, with zero indicating failure to this caller. | Allocation units, initialization, ownership, alignment, failure effects, and argument-width interpretation. |
| F-8b998a07b5e1 | Called when storage is required and the saved buffer address is nonzero. Receives saved address, effective length plus one, zero, zero. | Preservation of old bytes, old-storage lifetime, ownership transfer, and failure behavior. |
| F-unmapped-01 | Called with the current destination address, source address, and effective length when the source is nonzero. No result is inspected. | Whether it copies bytes, overlap rules, count interpretation, error behavior, and other side effects. |

F-unmapped-01 is a document-local opaque identifier for the external dependency absent from the supplied dependency map. It does not assert a resolved identity.

## Acceptance examples

These examples describe conditional observations, not executed tests. Except where stated otherwise, assume valid addresses, representable arithmetic, returning dependencies, and no dependency mutation of the object.

### Automatic length with no source

Given a negative requested length and a zero source address:

- Return 0.
- Make no dependency calls.
- Perform no direct object reads or writes.
- Perform no buffer write.

### Explicit length with no source and reusable storage

Given recorded length 12, a nonzero buffer address, source address zero, and requested length 5:

- Make no storage or transfer call.
- Set the recorded length to 5.
- Leave the buffer field directly unchanged.
- Do not directly initialize or terminate buffer contents.
- Return 1.

### Zero-length transfer with existing storage

Given recorded length 0, a nonzero buffer address, a nonzero source address, and requested length 0:

- Do not request storage.
- Store recorded length 0.
- Invoke F-unmapped-01 with count 0.
- After its return, write zero at the current buffer address.
- Return 1.

The external operation’s zero-count behavior remains a dependency condition.

### Missing storage despite a large recorded length

Given recorded length 100, buffer address zero, and requested length 3:

- Invoke F-43da6b23bd44 with first argument 4 and two zero arguments.
- On a nonzero result, store that address and then record length 3.
- If the source is nonzero, invoke the transfer dependency and perform the trailing-zero write.
- If the source is zero, skip both operations.

### Replacement failure

Given recorded length 2, a nonzero saved buffer address, and requested length 7, suppose F-8b998a07b5e1 returns zero:

- Invoke it with the saved address, 8, zero, zero.
- Write zero to the buffer field, then restore the saved address.
- Return 0 before this routine’s length update or transfer.
- With no dependency mutation, the recorded length remains 2.

The saved address being restored does not prove its allocation remains valid.

### Successful transfer under additional dependency guarantees

Given recorded length 6, valid existing storage, requested length 3, and a nonzero source, additionally suppose F-unmapped-01 copies exactly three bytes and preserves the object:

- No storage request occurs.
- The recorded length becomes 3.
- The destination receives those three bytes.
- The next byte is explicitly set to zero.
- Return 1.

The copied-byte result depends on the stated external guarantee; the invocation and trailing-zero write do not.

### Transfer changes the destination field

Suppose F-unmapped-01 changes the object’s buffer address before returning:

- Its call receives the address read before the call.
- The subsequent zero-byte write uses the address present after the call.
- The routine does not restore the earlier destination.

## Uncertainties

The following are **documented_only**:

- Concrete dependency semantics, including whether the apparent transfer actually performs a byte copy.
- Ownership and lifetime of original or replacement storage.
- Global and aliased-state changes made by dependencies.
- Transfer overlap guarantees and negative-count interpretation.
- Arithmetic behavior when the length increment exceeds signed range.
- Address-overflow behavior and invalid-memory outcomes.
- Meaning of the untouched field at offset 4.
- Whether surrounding callers enforce nonnegative computed lengths, valid capacities, or nonaliasing.
- Concurrency, reentrancy, and synchronization guarantees.

The inferred string-oriented intent does not resolve any of these unknowns.

## Evidence

**Established static evidence:** the supplied routine shows the early-return condition, dependency argument structure, signed length comparison, short-circuited storage condition, buffer-result write and restoration, logical-length update, optional external call, post-call buffer reread, single-byte zero write, and normal return values.

**Inferred intent:** automatic string-length selection, allocation or replacement of character storage, and transfer followed by termination. These interpretations fit the call shapes but are not dependency contracts.

**Verification status:** no runtime tests have been performed. All acceptance examples are conditional consequences of the supplied static evidence, with additional dependency assumptions stated explicitly.
