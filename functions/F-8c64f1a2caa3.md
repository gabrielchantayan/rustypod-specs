<!-- Trusted provenance: F-8c64f1a2caa3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8c64f1a2caa3

## Purpose

F-8c64f1a2caa3 conditionally updates an object’s status and payload fields, using a dependency’s result to decide whether to process a supplied or replacement record. On the processing path, it also decrements a field in that record and may invoke indirect dispatch and a terminal dependency.

The established behavior is a sequence of calls, field accesses, and conditional state changes. Interpreting the record as a reference-counted resource is plausible but not established. In particular, neither ownership transfer nor the effects of the terminal dependency can be determined from the supplied body alone.

The result is not necessarily Boolean: a nonzero result from the decision dependency can be returned unchanged.

## Interface

### Inputs

The visible interface has four argument positions:

| Position | Role established by the supplied body |
|---|---|
| 1 | Object address. Forwarded to dependencies and used as the base of byte-offset field accesses. |
| 2 | Opaque 32-bit value. Forwarded unchanged to F-31e285174b08. |
| 3 | Opaque 32-bit value. Not read or forwarded by this function. |
| 4 | Nullable pointer to a record. Initially copied into a local pointer slot; that slot is subsequently exposed to F-1e7091dc86a0 for possible replacement. |

The third argument has no direct effect in the visible body. That does not establish a broader calling convention or permission to omit the argument.

### Output

The ordinary return is a 32-bit integer:

- Zero if F-31e285174b08 returns zero.
- Zero if that dependency returns nonzero but the selected record is null or its first word is zero.
- Otherwise, the original nonzero result from F-31e285174b08.

Some paths reach F-feee9156a257, which is annotated as not returning. Such a path has no ordinary return under that annotation.

### Object layout used directly

Offsets are byte offsets from the first argument.

| Offset | Width | Direct use |
|---|---:|---|
| 0x8a4 | 1 byte | Status value written as zero or one. |
| 0x8a8 | 4 bytes | Selected record’s first word, or zero when the selected record is null. |

The function does not directly read either object field before writing it. It does not directly write the three bytes between these fields. Dependencies may have additional effects.

### Record layout used directly

The visible representation is consistent with 32-bit words and pointers:

| Offset | Width | Direct use |
|---|---:|---|
| 0 | 4 bytes | Payload copied to the object; later read again as an address on one conditional path. |
| 4 | 4 bytes | Read, decremented by one, and written back. |

A nonzero record pointer must permit the accesses required by the path taken. Null is explicitly handled without dereferencing it.

If indirect dispatch is reached, the first record word is interpreted as a pointer to an entity whose first word supplies another address. A callable entry is then loaded at byte offset 4 from that latter address. These interpretations are conditional; the initial payload transfer alone does not dereference the payload as an address.

### Preconditions

No explicit bounds, alignment, object-type, or callable-address validation appears in this body. Defined execution therefore requires valid storage and addresses for every access actually reached.

The representation strongly indicates a 32-bit address model. Exact ABI details, byte order, and the behavior of invalid addresses are not established by the supplied material.

## Behavior

### Entry and decision

The incoming record pointer is first copied into a local pointer slot. F-8d02f1ed0015 is then called without explicit arguments.

Only after that call completes does the function call F-31e285174b08 with the object address and the opaque second argument. Its result determines the main branch and is retained as the candidate return value.

F-8d02f1ed0015’s effects are unknown. The contract does not assume that it preserves globals, object storage, record storage, or other externally reachable state.

### Zero decision result

If F-31e285174b08 returns zero:

1. The function writes a single zero byte at object offset 0x8a4.
2. It returns zero.

On this branch, the function itself does not:

- Call F-1e7091dc86a0.
- Read the record’s first or second word.
- Write object offset 0x8a8.
- Decrement the record field.
- Invoke either conditional dispatch dependency.

Consequently, the object payload field is not directly cleared when the decision result is zero. Any value present there after the decision call remains untouched by this function’s own writes.

### Nonzero decision result and record selection

If F-31e285174b08 returns nonzero, F-1e7091dc86a0 is called with:

- The address of the local record-pointer slot.
- The object address.

This dependency can replace the pointer used by the remaining visible processing. The selected pointer need not equal the fourth argument.

After the dependency returns:

- A null selected pointer produces a selected payload value of zero.
- A nonnull selected pointer causes a four-byte read of its first word.

The selected payload value is then written as four bytes to object offset 0x8a8. This payload write precedes the subsequent status write.

### Zero selected payload

If the selected payload value is zero:

1. The candidate return value is changed to zero.
2. The function writes a zero byte at object offset 0x8a4.
3. F-unmapped-01 is called with the object address.
4. If that dependency returns, record-field processing follows.

The dependency is called for both a null selected record and a nonnull record whose first word is zero. It is not called merely because the initial decision result was zero.

Because F-unmapped-01 runs after the payload and status stores, it can observe those stores if they address ordinary accessible storage. Its effects are otherwise unknown. The function does not restore either field afterward.

### Nonzero selected payload

If the selected payload value is nonzero, the function writes a one byte at object offset 0x8a4 and retains the original nonzero decision result.

There is no conversion of that result to one. For example, a decision result of 7 remains 7 on an ordinary return from this path.

F-unmapped-01 is not called on this path.

### Record-field decrement

After the payload-dependent handling, the function checks the local record pointer again.

If it is null, there is no decrement or indirect dispatch.

If it is nonnull:

1. The four-byte field at record offset 4 is read.
2. One is subtracted.
3. The result is written back to that field.
4. The decremented result is tested for zero.

The write occurs before any resulting indirect call or terminal dependency call.

For values where the subtraction is defined, the zero-result branch is taken exactly when the prior field value was one. A prior zero value is not treated as “already released”: it is decremented to negative one under ordinary signed 32-bit arithmetic and does not take the zero-result branch.

There is no visible saturation, underflow guard, or validation that the field was positive.

### Zero-result dispatch path

If the decremented field is nonzero, the function proceeds toward its ordinary return without the dispatch described below.

If the decremented field is zero, the function rereads the selected record’s first word. This is a separate read from the earlier payload read; the value copied to the object is not reused for this decision.

When this reread value is nonzero, the function:

1. Treats it as an address.
2. Reads an address from the beginning of the referenced entity.
3. Reads a callable entry at byte offset 4 from that address.
4. Calls the entry, identified here as F-indirect-01.

No explicit arguments are shown for this indirect call. That is insufficient to establish any implicit receiver, register context, or complete ABI-level argument list.

After the indirect call returns—or immediately if the reread first word was zero—the function checks the local record pointer again. If it is still nonnull, it calls F-feee9156a257 without explicit arguments.

F-feee9156a257 is annotated as not returning. If that annotation is accurate, execution does not reach the ordinary return after this call.

The final pointer check must not be collapsed into an unconditional call: the pointer-slot address was exposed earlier, and an intervening dependency’s effects are not known. Conversely, the supplied body does not establish that any dependency actually changes the slot at that point.

## Side effects

### Direct writes and their ordering

The directly visible writes are:

- A zero byte at object offset 0x8a4 on the zero-decision path.
- On the nonzero-decision path, a four-byte payload write at offset 0x8a8, followed by a status-byte write at offset 0x8a4.
- A four-byte decrement write at selected-record offset 4 when the record pointer is nonnull at that stage.

The zero-payload dependency is called after both object writes and before record-field processing. Indirect dispatch, when reached, occurs after the decrement write.

These are the access-order relationships established by the supplied representation. No atomicity, synchronization, memory-barrier, or thread-safety guarantee is established.

### Ownership and lifetime

A decrement followed by conditional dispatch is consistent with reference-count release logic. That interpretation is not an ownership contract.

In particular, the evidence does not establish:

- Whether the incoming record is borrowed, transferred, or retained.
- Whether F-1e7091dc86a0 acquires or releases any resource.
- Whether F-unmapped-01 changes ownership.
- Whether F-indirect-01 destroys an entity.
- Whether F-feee9156a257 frees memory, terminates execution, or performs another operation.
- Whether any dependency preserves globals or accessible memory.

The local pointer slot has automatic lifetime within this invocation. Its address is exposed to F-1e7091dc86a0; permitted retention or later use of that address is not specified.

### Errors and partial state

There is no visible rollback or structured error reporting. Zero can result from either a zero decision result or a zero selected payload.

The function can have already written object fields and decremented the record field before reaching indirect dispatch or F-feee9156a257. Those earlier writes are not undone by the visible body.

Invalid pointers, inaccessible storage, invalid dispatch entries, and arithmetic overflow are not handled explicitly.

## Dependencies

| Opaque ID | Established call relationship |
|---|---|
| F-8d02f1ed0015 | Called once before the decision dependency, without explicit arguments. |
| F-31e285174b08 | Receives the first and second inputs. Its result gates processing and supplies the candidate return value. |
| F-1e7091dc86a0 | Called only after a nonzero decision result. Receives the local pointer-slot address and object address. |
| F-unmapped-01 | Called with the object address after a zero selected payload has been stored and status cleared. |
| F-indirect-01 | Dynamically resolved through the record’s reread first word and two further address-based reads. Called only after the decrement produces zero and the first word is nonzero. |
| F-feee9156a257 | Called after a zero decrement result if the final local-pointer check is nonnull. Annotated as not returning. |

F-unmapped-01 and F-indirect-01 are document-local opaque labels, not identities supplied by the dependency map. Their implementations are unavailable.

## Acceptance examples

These examples are conditional behavioral criteria, not reports of executed tests. Unless noted otherwise, dependencies return normally and do not introduce additional mutations.

1. **Decision rejection preserves the payload field directly.**  
   If F-31e285174b08 returns zero, the function writes zero to the status byte and returns zero. It does not itself write the payload field or touch either record word.

2. **Null selection after a positive decision.**  
   If the decision result is 5 and F-1e7091dc86a0 leaves a null selected pointer, the function writes a zero payload, clears status, and calls F-unmapped-01. If the pointer remains null, no decrement occurs and the ordinary result is zero.

3. **Non-Boolean successful result.**  
   If the decision result is 7, the selected record has a nonzero first word, and its second word is 3, the function stores that first word, sets status to one, changes the second word to 2, and ordinarily returns 7.

4. **Zero payload with a nonnull record.**  
   If the selected record’s first word is zero and its second word is 2, the function stores zero, clears status, and calls F-unmapped-01. If that call leaves the record unchanged, the second word becomes 1 and the ordinary result is zero.

5. **Decrement to zero with a nonzero payload.**  
   If the selected record’s second word is 1 and its first word remains a valid nonzero dispatch address, the function writes zero to the second word before calling F-indirect-01. If the local pointer remains nonnull afterward, F-feee9156a257 is called; its nonreturn annotation precludes an ordinary return.

6. **Decrement to zero with a zero payload.**  
   If the payload is zero, F-unmapped-01 returns without changing the record, and the second word is 1, the function decrements that field to zero. It skips F-indirect-01 because the reread first word is zero, but still calls F-feee9156a257 if the local pointer remains nonnull.

7. **Replacement record controls later accesses.**  
   If F-1e7091dc86a0 replaces the incoming pointer with another valid record, payload selection and subsequent record-field processing use the replacement, subject to later dependency effects.

8. **Zero is not a protected decrement boundary.**  
   If the second word is zero when read, it is decremented rather than left unchanged. With ordinary signed 32-bit arithmetic, the resulting negative one does not trigger zero-result dispatch.

## Uncertainties

The following are **documented_only**:

- Exact ABI, byte order, alignment requirements, and machine-level pointer representation.
- Signed-overflow behavior when decrementing the minimum signed 32-bit value. The visible signed expression does not independently establish wrapping semantics.
- Whether the status byte means validity, availability, readiness, or another state.
- The semantic type of the payload and the meaning of the decremented field.
- All dependency effects beyond their visible invocation conditions and arguments.
- Whether F-feee9156a257’s nonreturn annotation is accurate.
- The indirect call’s complete calling convention and any implicit arguments.
- Aliasing restrictions among object storage, record storage, and dependency-accessible memory.

The acceptance examples assume nonproblematic aliasing. In the general case, earlier writes may affect later reads when their storage overlaps. No additional disjointness guarantee is evident.

## Evidence

Established static evidence consists solely of the supplied function body and dependency map. It supports the branch conditions, call order, field offsets, access widths, payload reread, decrement-before-dispatch ordering, and ordinary return-value selection described above.

Reference-counting and resource-release terminology describe inferred intent only. They do not establish ownership or destruction behavior.

No dependency implementations, runtime traces, or runtime tests were provided. No runtime tests have been performed.
