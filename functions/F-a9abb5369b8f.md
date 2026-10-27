<!-- Trusted provenance: F-a9abb5369b8f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a9abb5369b8f

## Purpose

F-a9abb5369b8f constructs a two-word result, conditionally allowing an indirectly selected operation to replace that result through a temporary four-word record.

When the receiver’s control word is zero, the result combines the receiver’s address representation with the supplied scalar. The input-record pointer is not dereferenced, and no indirect operation is invoked.

When the control word is nonzero, four words are copied from the input record into temporary storage. An operation selected through the receiver is then invoked with the receiver, the original scalar, and the temporary record. If that operation returns normally, the first two words of the temporary record become the result.

This describes the supplied static representation. The business meaning of the control word, record fields, and result is not established.

## Interface

### Inputs

The function has three inputs:

| Input | Role |
|---|---|
| Receiver | Address of an object-like storage region containing a dispatch-table reference and a control word. |
| Scalar | One 32-bit word, represented as a signed integer in the supplied declaration. |
| Input record | Address of four consecutive 32-bit words, read only on the enabled path. |

The supplied representation uses four-byte integer words and treats the receiver address as a four-byte component of the result. The contract below follows that representation. The original executable’s calling convention and pointer width are not independently established.

### Receiver layout

Two receiver locations participate directly:

| Byte offset | Width | Meaning established by use |
|---|---:|---|
| 0 | 4 bytes under the represented pointer model | Reference to a dispatch table, consulted only on the enabled path. |
| 172 | 4 bytes | Control word. Zero disables dispatch; every nonzero value enables it. |

The control word is at word index 43. Only the two listed locations are interpreted by this function; no meanings for intervening storage are established.

The enabled path selects a callable entry at byte offset 280 from the referenced dispatch table. Under a four-byte table-entry model, that is entry index 70. The byte offset is the directly supported property; the table’s broader organization is unknown.

### Input-record layout

The input record consists of these adjacent cells:

| Byte offset | Width | Use |
|---|---:|---|
| 0 | 4 bytes | Initial value of temporary cell 0. |
| 4 | 4 bytes | Initial value of temporary cell 1. |
| 8 | 4 bytes | Initial value of temporary cell 2. |
| 12 | 4 bytes | Initial value of temporary cell 3. |

All four cells are copied before the indirect invocation. Their semantic types are unknown. In particular, the pointer-like representation used for temporary cell 0 does not establish that its contents must identify valid memory: this function does not dereference the value stored in that cell.

### Output

On normal return, the function produces a 64-bit value composed of two 32-bit components:

- Bits 0–31 contain temporary cell 0.
- Bits 32–63 contain temporary cell 1.

This is bit concatenation, not signed arithmetic. A high component with its sign bit set remains the high 32-bit pattern; it is not sign-extended into a separately computed result.

Temporary cells 2 and 3 are not directly included in the return value.

## Behavior

### Common entry behavior

The function first establishes default result components:

- Temporary cell 0 receives the receiver’s address representation.
- Temporary cell 1 receives the supplied scalar.

It then reads the receiver’s control word at byte offset 172 and compares that word with zero.

The observed decision is based on that read. The function contains no later control-word check, retry, or revalidation.

### Disabled path

If the control word is zero:

1. The input record is not read.
2. The dispatch-table reference is not read for dispatch.
3. No indirect operation is invoked.
4. The function returns the default two-word result.

The low 32 bits are the receiver’s address representation, and the high 32 bits are the scalar’s bit pattern.

Temporary cells 2 and 3 have no established initialized values on this path. They are neither supplied to another operation nor used in the result, so their contents do not affect the specified normal return.

### Enabled path

If the control word is nonzero, the function replaces the temporary record’s contents with the four input-record words.

The represented order is:

1. Read the word at input offset 0 and place it in temporary cell 0.
2. Read the word at input offset 4 and place it in temporary cell 1.
3. Read the word at input offset 8 and place it in temporary cell 2.
4. Read the word at input offset 12 and place it in temporary cell 3.
5. Resolve the indirect operation through the receiver’s dispatch-table reference and the entry at table offset 280.
6. Invoke that operation.

The operation receives three arguments:

- The original receiver.
- The original scalar.
- The address of temporary cell 0, providing access to the temporary record.

The scalar argument is not replaced by input-record cell 1. That cell initializes the temporary record independently of the scalar passed to the operation.

The operation is not passed the original input-record pointer as its third argument. Consequently, writing the temporary record is not, by itself, a write to the original input record.

No return value from the indirect operation is used by this function. Its relevant output mechanism is modification of the temporary record, although other effects are not constrained by the supplied material.

### Completion after the indirect operation

If the indirect operation returns normally, the function constructs its result from the then-current contents of temporary cells 0 and 1.

Therefore:

- A modification to temporary cell 0 changes the low result component.
- A modification to temporary cell 1 changes the high result component.
- Modifications only to temporary cells 2 and 3 do not directly change the returned bits.
- If the operation leaves cells 0 and 1 unchanged, the result contains the first two copied input-record words.

The function does not re-copy the input record after the operation. A subsequent change to the original input record does not automatically update the temporary record.

The supplied representation establishes no significant ordering between the final reads of temporary cells 0 and 1. Both are used after the indirect operation has returned.

### Aliasing and concurrent access

No distinctness requirement between the receiver and input-record storage is demonstrated. If those regions overlap, the same ordered reads apply: the control word is inspected first, the four input words are copied on the enabled path, and dispatch resolution follows.

The temporary record provides a completed copy before the indirect invocation; it is not an atomic snapshot of externally mutable storage. No locking, synchronization, or atomic accesses are shown. Behavior under concurrent conflicting access cannot be established from this material.

## Side effects

### Effects directly established

The function directly:

- Writes its temporary result and record storage.
- Reads the receiver’s control word.
- On the enabled path, reads four input-record words.
- On the enabled path, reads dispatch information and invokes an indirect operation.

No direct write to the receiver or original input record is shown. No direct heap allocation, deallocation, resource acquisition, or resource release is shown.

These observations do not restrict the indirect operation. It may modify the receiver, access aliased storage, affect globals, allocate or release resources, or perform other externally visible work. None of those effects is established or excluded.

### Ownership and lifetime

No ownership transfer for the receiver or input record is explicitly performed.

The temporary record is local to this invocation and is supplied to the indirect operation for use during that invocation. No valid lifetime beyond the enclosing call is established. Whether the indirect operation retains its address is unknown; retained access after the temporary storage expires would not have a supported lifetime under this representation.

A word copied into or returned from cell 0 is not automatically acquired, released, validated, or dereferenced by this function. Any ownership meaning attached to that word must come from an external contract.

### Errors and abnormal completion

There is no explicit validation or error-result path.

Normal completion requires, as applicable:

- A receiver from which the control word can be read.
- On the enabled path, four readable input-record words.
- On the enabled path, readable dispatch metadata and a callable selected entry.
- An indirect invocation compatible with the represented argument arrangement.
- Normal return from that invocation.

The function does not define a fallback for invalid addresses, unreadable storage, an invalid callable entry, or failure inside the indirect operation. It contains no visible recovery or cleanup path. The resulting platform-level failure mechanism is unknown.

## Dependencies

### F-00000001 — unresolved indirect operation

This is a document-local opaque identifier for the operation selected from the receiver’s dispatch table at byte offset 280. It does not establish the identity of any particular implementation.

Its observed interface is:

- Original receiver as the first argument.
- Original 32-bit scalar as the second argument.
- Address of the four-word temporary record as the third argument.

Its return value, if any, is ignored.

Its full contract is **documented_only**. The available material does not establish:

- Which concrete operation is selected for any receiver.
- Whether it modifies any temporary cells.
- Whether it modifies receiver or global state.
- Whether it accesses or changes the original input record through another reference.
- Whether it acquires or releases memory.
- Whether it retains argument addresses.
- Whether it always returns normally.

No other called operation is shown.

## Acceptance examples

These examples are conditional consequences of the static representation, not runtime observations. They assume the represented four-byte word and address model and valid storage for each accessed location.

### Disabled dispatch ignores the input-record pointer

Given:

- Receiver address representation `0x12345678`.
- Scalar bit pattern `0x89ABCDEF`.
- Control word equal to zero.

The normal result is `0x89ABCDEF12345678`.

The input-record pointer need not identify readable storage for this path, because it is not dereferenced. The receiver must still support the control-word read.

### Enabled dispatch with an unchanged record

Given:

- A nonzero control word.
- Input-record words, in order: `0x11223344`, `0x55667788`, `0x99AABBCC`, `0xDDEEFF00`.
- A valid selected operation that returns normally without changing the temporary record.

The result is `0x5566778811223344`.

All four words are nevertheless read and supplied through the temporary record; the last two are not skipped merely because they are absent from the result.

### The original scalar remains a separate argument

Given:

- Scalar bit pattern `0x00000007`.
- Input-record cell 1 equal to `0x00000063`.
- Enabled dispatch.

The operation receives scalar `0x00000007` as its second argument, while temporary cell 1 initially contains `0x00000063`.

If the operation leaves that cell unchanged, the high result component is `0x00000063`.

### Indirect modification determines the result

Given enabled dispatch and an operation that sets:

- Temporary cell 0 to `0x00000009`.
- Temporary cell 1 to `0xFFFFFFFE`.

After normal return, the result is `0xFFFFFFFE00000009`, regardless of the original values copied into those two cells.

### Every nonzero control value enables dispatch

A control-word bit pattern of `0xFFFFFFFF` enables the same branch as `0x00000001`. The decision is zero versus nonzero, not positivity or an individual-bit test.

### Short input storage is insufficient when enabled

If only the first two input words are readable, normal completion is not established on the enabled path. The function also reads the words at offsets 8 and 12 before invoking the operation.

There is no specified shortened-record mode or error return for this case.

## Uncertainties

The following remain **documented_only**:

- **Native ABI:** The representation supports 32-bit words and four-byte address components, but does not independently prove the executable’s calling convention, alignment rules, or native return mechanism.
- **Record meaning:** The four fields may carry values, addresses, handles, or another encoding. Their domain semantics are unknown.
- **Control-word intent:** Its use as a zero/nonzero dispatch gate is established; its broader purpose is inferred rather than proven.
- **Dispatch identity:** The selected operation’s identity and complete contract are unavailable.
- **Ownership:** No resource ownership policy can be inferred from a field’s pointer-like representation.
- **Exceptional behavior:** Fault handling, unwinding, nonlocal exits, and external error conventions are not shown.
- **Memory layout of the result:** High and low bit components are established, but byte order when the result is stored in memory is not.
- **Concurrency:** No safe concurrent-access protocol is established.

## Evidence

The supplied static representation establishes:

- Default result initialization from the receiver and scalar.
- A zero/nonzero test of the receiver word at index 43.
- Four consecutive input-word copies only on the nonzero path.
- An indirect invocation selected at dispatch-table byte offset 280.
- Passing the original receiver, original scalar, and temporary-record address.
- A final 64-bit concatenation with temporary cell 1 above temporary cell 0.

Interpreting this as an optional result-transforming operation is inferred intent. The record’s semantic meaning and the dependency’s behavior are unknown.

No runtime tests have been performed.
