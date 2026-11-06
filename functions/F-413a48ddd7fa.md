<!-- Trusted provenance: F-413a48ddd7fa; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-413a48ddd7fa

## Purpose

F-413a48ddd7fa conditionally transfers an eight-byte representation of two dynamically obtained 32-bit values to an output object. Before deciding whether to perform that transfer, it obtains a record, populates its first four bytes from a conversion result, and invokes a predicate-like dependency on that record.

When the predicate result is nonzero, the function obtains two values through separate indirect calls, encodes each value most-significant byte first, and submits the resulting eight bytes in one output call. Any output result other than exactly eight causes a further call on the record with the numeric argument two.

These are established control-flow and data-flow properties. The application meaning of the record, the two values, the predicate, and the numeric state argument is unknown. In particular, interpreting the values as dimensions, identifiers, or addresses would go beyond the supplied evidence.

## Interface

The function has three inputs and no source-level return value:

| Input | Representation visible in the supplied source | Role |
|---|---|---|
| Context argument | One 32-bit value | Passed unchanged to F-603a53835f04. Its semantic type is unknown. |
| Source reference | Pointer to a 32-bit slot | Passed to F-5d7410a76c21 and subsequently dereferenced to locate an object used for two indirect calls. |
| Output object | Pointer to a 32-bit slot | Used to locate a dispatch table and passed explicitly as the receiver argument to the output operation. |

The supplied source uses a 32-bit pointer and dispatch representation. The contract describes that representation; it does not establish compatibility with a different pointer width or calling convention.

Relevant layouts are:

- **Returned record:** F-603a53835f04 returns an address used for two adjacent 16-bit writes, at byte offsets zero and two. No larger record layout is established.
- **Conversion result:** F-5d7410a76c21 receives temporary storage from which the function reads two adjacent 16-bit fields. Their semantic types and byte order are unknown.
- **Source object dispatch:** the current 32-bit value in the source-reference slot identifies an object whose first 32-bit word identifies a dispatch table. Operations are selected at byte offsets 40 and 36, in that order.
- **Output object dispatch:** the output object’s first 32-bit word identifies a dispatch table. The output operation is selected at byte offset 16.
- **Buffer holder:** temporary state initialized by F-9b3ee6b2bfa6 supplies a byte pointer and another 32-bit field. The byte pointer is used to write eight consecutive bytes. A separate descriptor containing that pointer and the additional field is then supplied to the output operation.

The complete buffer-holder layout, the meaning of its additional field, and any allocation strategy are not established.

For normal execution, all accessed slots, dispatch tables, selected call targets, record storage, and buffer storage must be valid for the operations described below. The function contains no explicit null checks or bounds checks establishing those conditions.

## Behavior

### Record acquisition and initialization

The function first calls F-603a53835f04 with the context argument followed by three zero-valued arguments. It retains the returned address as the record address.

It then calls F-5d7410a76c21 with:

1. The address of temporary conversion-result storage.
2. The source-reference pointer.
3. The numeric argument one.

After that call returns, the function performs these operations in order:

1. Reads the first 16-bit conversion-result field and writes it to record offset zero.
2. Reads the second 16-bit conversion-result field and writes it to record offset two.
3. Calls F-9b718979506d with the address of the conversion-result storage.
4. Calls F-ec61551795ee with the record address.

Thus, the record receives both fields before the conversion-result cleanup-like call, and that call completes before the predicate-like call begins. The function does not perform a single explicit 32-bit record assignment.

The stored record address remains the one obtained initially. The supplied body does not reacquire it after conversion or cleanup. Correct continuation therefore depends on that address remaining usable across those calls; this is not a guarantee supplied by the unknown dependencies.

### Gate on the predicate result

The 32-bit result of F-ec61551795ee is compared with zero.

- If it is zero, the function returns without creating the eight-byte buffer, invoking either source dispatch operation, invoking the output operation, or calling F-7f008c1060db.
- If it is nonzero, including a negative value under the source’s signed interpretation, the function continues with buffer preparation and output.

The zero-result path does not undo the two record writes. The conversion-result cleanup-like call has already occurred on either path.

### Buffer preparation and value acquisition

For the nonzero path, the function calls F-9b3ee6b2bfa6 with temporary buffer-holder storage and the numeric arguments eight and three.

It next invokes the source operation selected at dispatch-table byte offset 40 and retains its result as the first 32-bit payload value. It then invokes the operation selected at byte offset 36 and retains its result as the second 32-bit payload value.

The call order is significant: offset 40 precedes offset 36, even though offset 36 is the lower table offset.

Each indirect call resolves its target from the source-reference slot at that point in execution. The second resolution is not visibly cached from the first. Consequently, if the first operation changes the slot, object, or dispatch table, the second operation follows the state then visible through that slot.

No explicit argument list is shown for either source operation. The evidence does not establish an implicit receiver convention, additional arguments, or the operations’ semantic identities.

### Eight-byte encoding

Both source operations finish before the first explicit payload-byte write.

Let the first retained 32-bit value be A and the second be B. The function writes:

| Buffer offset | Byte |
|---|---|
| 0 | Bits 31–24 of A |
| 1 | Bits 23–16 of A |
| 2 | Bits 15–8 of A |
| 3 | Bits 7–0 of A |
| 4 | Bits 31–24 of B |
| 5 | Bits 23–16 of B |
| 6 | Bits 15–8 of B |
| 7 | Bits 7–0 of B |

The writes occur in increasing offset order, zero through seven. This defines a most-significant-byte-first representation for each value, independent of the target’s native memory byte order.

The shifts operate on unsigned 32-bit representations. A high sign bit does not cause sign extension into the extracted bytes. Conversion to the byte-sized destination retains the low eight bits of each shifted value. There is no payload-length calculation, addition of the values, or variable-length encoding.

After the eight writes, the function copies the buffer pointer and its accompanying 32-bit field into a separate descriptor. It passes the address of that descriptor to the output operation.

### Output result and finalization

The output operation is invoked through the output object’s dispatch-table entry at byte offset 16. Its explicit arguments are:

1. The output-object pointer.
2. The requested count, eight.
3. The descriptor address.

The returned 32-bit integer is compared with exactly eight:

- A result of eight causes no explicit record-state call.
- Every other result causes F-7f008c1060db to be called with the record address and the numeric argument two.

This includes zero, positive results below eight, positive results above eight, and negative results. There is no retry, remainder transfer, alternate output path, or rollback in the supplied body.

After the optional record-state call, F-ffa7163332aa is invoked on the original buffer-holder storage. The function then returns without a source-level result.

This finalization order is established only for calls that return normally. The supplied source does not establish exception handling or guaranteed cleanup following a non-returning dependency, invalid access, or other abnormal transfer of control.

## Side effects

The function explicitly changes externally reachable state in these ways:

- Writes two 16-bit fields to the returned record before evaluating the gate.
- On the nonzero-gate path, writes exactly eight consecutive payload bytes to buffer storage.
- Invokes an output operation once with a requested count of eight.
- On an output result other than eight, invokes a record operation with argument two.

The dependencies and indirect operations may have additional effects. Nothing in the supplied body establishes that they preserve global state, source-reference contents, object state, dispatch tables, or aliased memory.

Ownership must not be inferred from names or placement alone:

- F-9b718979506d is called on the temporary conversion result, but its exact resource effects are unknown.
- F-ffa7163332aa is called on the buffer holder, not on the separate descriptor passed to output.
- No explicit record-release operation appears.
- No explicit source-object or output-object release appears.
- Passing the descriptor to output does not establish that the output operation takes ownership of either the descriptor or the byte storage.

The descriptor is temporary storage used during this invocation. Any dependency that retains its address would require lifetime behavior not established here. Whether output copies, consumes, references, or retains the payload is documented_only.

Aliasing is not excluded by the visible interface. If caller-supplied references or dependency-returned storage overlap, observable results depend on the stated ordering and on the unknown dependencies’ behavior. No non-aliasing guarantee is established.

## Dependencies

| Opaque ID or operation | Established use | Unestablished semantics |
|---|---|---|
| F-603a53835f04 | Receives the context and three zeros; returns the record address. | Whether it allocates, looks up, borrows, or otherwise obtains the record. |
| F-5d7410a76c21 | Receives conversion-result storage, the source reference, and one. Supplies fields subsequently read as two 16-bit values. | Field meanings, full result layout, ownership, and other side effects. |
| F-9b718979506d | Receives the conversion-result address after both record writes. | Whether it releases memory, adjusts references, or performs another operation. |
| F-ec61551795ee | Receives the record; its zero/nonzero result controls output. | Whether the result represents validity, readiness, status, or another condition. |
| F-9b3ee6b2bfa6 | Receives buffer-holder storage, eight, and three before payload construction. | Allocation policy, failure behavior, and meaning of three. |
| F-7f008c1060db | Receives the record and two when the output result differs from eight. | Meaning of two and the precise state transition. |
| F-ffa7163332aa | Receives the original buffer holder after output-result handling. | Exact destruction, release, or bookkeeping behavior. |
| Source dispatch operations | Selected at offsets 40 and 36; yield the first and second 32-bit payload values. | Opaque IDs, receiver convention, meanings, and side effects. |
| Output dispatch operation | Selected at offset 16; receives the output object, eight, and the descriptor address. | Opaque ID, transfer guarantees, ownership, and interpretation of its result beyond this function’s comparison. |

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests. Each assumes valid storage and normally returning dependencies.

### Zero gate suppresses output

Suppose conversion produces 16-bit fields `0x1234` and `0xABCD`, and F-ec61551795ee returns zero.

Expected behavior:

- Record offsets zero and two receive those respective 16-bit values.
- F-9b718979506d runs after both writes and before the gate call.
- No buffer construction, source dispatch, output dispatch, record-state call, or buffer finalization occurs.

The record’s four individual memory bytes are not specified here because the target’s native byte order is not established.

### Exact-count output

Suppose the gate is nonzero, the first source operation returns `0x12345678`, and the second returns `0x90ABCDEF`.

The payload is `12 34 56 78 90 AB CD EF`.

If output returns eight, F-7f008c1060db is not called. F-ffa7163332aa follows the output call.

### Non-eight output result

With the same payload, suppose output returns seven, zero, nine, or negative one.

For each case, F-7f008c1060db receives the record and two before buffer finalization. The function does not issue another output call.

### High-bit values

Suppose A is `0x80000001` and B is `0xFFFFFFFF`.

The payload is `80 00 00 01 FF FF FF FF`. A signed interpretation of either returned value does not alter that byte sequence.

### Changed source reference between calls

Suppose the first source operation changes the source-reference slot to identify another valid object.

The second operation is resolved through the updated slot and its then-current dispatch table. Requiring both operations to target the original object would contradict the visible lookup sequence.

## Uncertainties

The following remain documented_only:

- The application purpose and field meanings of the record.
- The complete object, conversion-result, buffer-holder, and descriptor layouts.
- Native byte order for the two 16-bit record stores.
- Dependency ownership, allocation, release, and global-state effects.
- Source-operation calling conventions and semantic return types.
- Output transfer semantics, including whether a returned count reflects actual bytes consumed.
- Failure behavior for invalid addresses, allocation failure, dependency failure, exceptions, or nonlocal exits.
- Thread safety, synchronization, and reentrancy guarantees.

Calling the gate a validity check, the argument two an error code, or the finalization calls destructors would be inferred intent rather than established behavior.

## Evidence

This contract is based solely on the supplied static source representation and dependency-ID mapping.

Established evidence covers call order, explicit arguments, dispatch offsets, access widths, branch conditions, byte extraction, output-result comparison, and normal-path finalization. It does not establish the internals of any dependency or indirect operation.

No runtime tests have been performed. The acceptance examples describe conditional obligations, not observed execution results.
