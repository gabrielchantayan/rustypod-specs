<!-- Trusted provenance: F-d4fdd2037c1c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d4fdd2037c1c

## Purpose

F-d4fdd2037c1c conditionally validates an integer position against an object, optionally notifies object-dispatched operations about a following range and a distinguished value, and invokes a final dependency for the selected position.

The established behavior is a sequence of reads, calls, branches, and return values. Interpreting the operation as deletion, truncation, selection, or cursor movement is not justified by the available evidence. In particular, the calls’ arguments suggest position-based processing, but their actual effects are unknown.

The function has three principal outcomes:

- Return −1 when ordinary validation reports zero.
- Return the selected position after the applicable dispatched calls and final dependency return.
- Enter a conditionally non-returning dependency following an initial status check.

An input of 2,147,483,647 has special treatment when an object field is positive: it selects one less than that field and bypasses ordinary validation.

## Interface

### Inputs

The recovered interface has two explicit inputs:

1. **Object reference.** A pointer to storage accessed as integer words and as an indirect-dispatch object.
2. **Requested position.** A signed integer, represented by the recovered interface as a 32-bit value.

The function returns a signed integer of the same represented width. Both −1 and 2,147,483,647 have explicit control-flow significance.

The following layout describes the object storage directly accessed by this function. It does not establish the object’s complete size or type.

| Byte offset | Access | Established use |
|---|---|---|
| 0 | Read a dispatch-table reference | Locate indirect call targets |
| 4 | Read a signed integer | Special-value selection and comparison with the following position |
| 16 | Read an integer and convert it to a character-sized value | Enable or disable the preliminary status check |

The field at offset 4 is called the **bound field** below. This is a descriptive label, not a guarantee that it is a count, length, capacity, or invariant upper limit.

The field at offset 16 is called the **check field**. The condition uses its character-converted value, not an established full-word nonzero test.

The observed table-relative call slots are:

| Table byte offset | Explicit arguments | Result used here |
|---|---|---|
| 0x40 | Object reference; original requested position | A reference whose first integer word is read |
| 0xB8 | Object reference; selected position plus one; bound field minus one; all-one 32-bit value; four additional opaque values | No result used |
| 0xBC | Object reference; all-one 32-bit value | No result used |

The four trailing values supplied to slot 0xB8 are not initialized or explained within this function. Their presence in the recovered call does not establish their intended types or semantic meanings.

### Preconditions and validity requirements

No explicit null, bounds, dispatch-target, or returned-reference checks are shown.

For any executed path:

- The object storage actually read must be accessible.
- Each dispatch-table reference and slot actually used must identify a callable target under the applicable calling convention.
- If slot 0x40 is called and returns, its returned reference must permit reading its first integer word.
- The object must remain suitable for all subsequent accesses that occur.

These are requirements for defined execution, not validations performed by F-d4fdd2037c1c. An invalid reference is not visibly converted into a −1 result.

Pointer width, alignment requirements, endianness, and the precise calling convention are not independently established by the supplied evidence.

## Behavior

### 1. Preliminary conditional status check

The function first reads the check field and converts it to a character-sized value.

If that converted value is zero, the preliminary dispatched call is skipped. Execution proceeds to position selection or validation.

If the converted value is nonzero:

1. The function obtains the object’s dispatch target at table offset 0x40.
2. It calls that target with the object reference and the original requested position.
3. After the call returns, it reads the first integer word through the returned reference.
4. If that word is nonzero, it calls F-feee9156a257.

The slot-0x40 call occurs before special-value handling. Therefore, when the requested position is 2,147,483,647, this call receives that original value, not a later derived position.

A zero first word permits execution to continue. A nonzero first word selects F-feee9156a257. The supplied annotation identifies this dependency as non-returning; its termination mechanism and externally observable effects are not established.

The slot-0x40 call may change the object or external state. Later behavior must be determined from the later reads, not from an assumed snapshot taken before this call.

### 2. Select a position or perform ordinary validation

After the preliminary stage, the function considers the requested position and the bound field.

If both of the following hold:

- The requested position equals 2,147,483,647.
- The bound field is greater than zero.

then the selected position becomes the bound field minus one. F-402eef64af83 is not called on this branch.

The positive test and the value used to form the selected position are represented as separate accesses. The evidence does not establish an atomic snapshot or synchronization with concurrent mutation.

In every other case, the function calls F-402eef64af83 with the object reference and the original requested position.

- If that dependency returns zero, F-d4fdd2037c1c immediately returns −1.
- If it returns any nonzero integer, execution continues with the requested position unchanged as the selected position.

The dependency’s return is used only as a zero/nonzero decision. It is not substituted for the position. A negative nonzero return therefore permits continuation just as a positive return does.

The special input does not universally mean “use the final position.” If the bound field is zero or negative, the function passes 2,147,483,647 to ordinary validation.

### 3. Treat selected position −1 as a downstream no-op

If the selected position equals −1, the function skips:

- The optional slot-0xB8 call.
- The slot-0xBC call.
- F-d6032bc0598f.

It then returns −1.

This path is distinct from validation failure. The same returned value can mean either:

- F-402eef64af83 rejected the request by returning zero; or
- F-402eef64af83 returned nonzero for a requested position of −1, after which the downstream work was skipped.

The preliminary status check, and ordinary validation where applicable, can already have produced side effects before either return.

### 4. Optionally invoke the following-range dispatch

For a selected position other than −1, the function compares the selected position plus one with the bound field.

If the former is strictly less than the latter, it invokes slot 0xB8 with:

1. The object reference.
2. The selected position plus one.
3. The bound field minus one.
4. The 32-bit bit pattern 0xFFFFFFFF.
5. Four additional opaque values.

Equality does not invoke slot 0xB8. Thus, for ordinary representable arithmetic and a stable bound, a position immediately preceding the bound does not cause this range call.

The bound read for the comparison and the bound used to form the call argument are not an established single snapshot. The exact ordering of reads within the call expression is also not established.

The all-one argument has the signed 32-bit interpretation −1, but its semantic role is unknown. It must not be described as a deletion marker, invalid index, or command code without additional evidence.

### 5. Invoke the remaining operations in order

Whether or not slot 0xB8 was called, a selected position other than −1 then causes:

1. A call through slot 0xBC, with the object reference and the all-one 32-bit value.
2. A call to F-d6032bc0598f, with the object reference and selected position.
3. Return of the selected position.

The slot-0xB8 call, if taken, completes before the slot-0xBC call. The slot-0xBC call completes before F-d6032bc0598f.

The dispatch table is consulted for the later operation rather than being established as a table cached before earlier calls. A dependency that changes object dispatch state can therefore affect a subsequent dispatch.

Neither dispatched call’s return value is inspected. F-d6032bc0598f also supplies no value used to determine the function’s result. If any dependency fails to return normally, the final return is not reached.

### Arithmetic boundaries

The recovered interface uses signed integer comparisons and arithmetic involving one-step increments and decrements.

For a stable, positive 32-bit bound, selecting bound minus one is representable. Other paths are not equivalently protected:

- Ordinary validation may return nonzero for 2,147,483,647 when the bound is nonpositive. The subsequent selected-position increment then exceeds the signed 32-bit range.
- A bound of −2,147,483,648 would make bound minus one nonrepresentable if that subtraction were reached.
- A successful validation result does not itself establish that later arithmetic is safe.

The evidence does not resolve signed-overflow behavior. A behavioral contract must not silently choose wrapping, trapping, saturation, or rejection for these cases.

## Side effects

The function’s visible body performs reads and dependency calls. It shows no direct object-field assignment, allocation, memory release, or buffer copy.

That does not make it side-effect free. Its dependencies may modify the object, dispatch table, other memory, global state, or external resources. Their preservation properties are not supplied.

The established cross-call order is:

1. Optional slot-0x40 call.
2. Conditional F-feee9156a257 call.
3. F-402eef64af83 call, unless the positive-bound special branch applies.
4. Optional slot-0xB8 call.
5. Slot-0xBC call.
6. F-d6032bc0598f call.

Steps 4–6 are omitted on validation rejection or when the selected position is −1.

There is no visible rollback. Side effects from an earlier call are not visibly undone if a later call terminates, faults, or otherwise fails to return.

The returned reference from slot 0x40 is read but is not visibly released or retained. Its ownership and lifetime contract are unknown. No dependency may be assumed to preserve globals, retain references, or release memory merely because such behavior would fit a plausible interpretation.

## Dependencies

| Dependency | Established contract at this call site | Unknowns |
|---|---|---|
| F-402eef64af83 | Receives object and original requested position. Zero causes immediate −1 return; any nonzero result permits continuation. | Validation rules, mutations, resource behavior, failure mechanisms |
| F-feee9156a257 | Called when the preliminary returned word is nonzero. No explicit arguments are shown. Annotated non-returning. | Termination mechanism, diagnostics, cleanup, implicit machine-state inputs |
| F-d6032bc0598f | Receives object and selected position after slot 0xBC returns. Its result is not used. | Actual operation, mutations, ownership effects, failure behavior |

The three dispatched operations have no supplied opaque function IDs. They are identified only by their table slots. Assigning them the identity or semantics of a named dependency would be unsupported.

## Acceptance examples

These are conditional static expectations, not executed tests. Unless stated otherwise, assume valid references, stable fields, representable arithmetic, and normal return from dependencies.

1. **Ordinary validation rejects.**  
   The check field converts to zero, and F-402eef64af83 returns zero for position 3. Expected: return −1; no calls through slots 0xB8 or 0xBC; no call to F-d6032bc0598f.

2. **Positive-bound special selection.**  
   The check field converts to zero, the request is 2,147,483,647, and the bound is 5. Expected: skip F-402eef64af83; select 4; skip slot 0xB8 because 5 is not less than 5; call slot 0xBC, then F-d6032bc0598f with position 4; return 4.

3. **Special request with zero bound.**  
   The request is 2,147,483,647 and the bound is zero. Expected: call F-402eef64af83 with the original request. If it returns zero, return −1. If it returns nonzero, later increment overflow is unresolved rather than assigned a guessed result.

4. **Following range exists.**  
   The request is 2, the bound remains 6, and validation returns nonzero. Expected: slot 0xB8 receives range arguments 3 and 5, followed by the all-one value and opaque trailing values. Then slot 0xBC and F-d6032bc0598f execute; return 2.

5. **Following range is empty at equality.**  
   The request is 5, the bound remains 6, and validation returns nonzero. Expected: no slot-0xB8 call; slot 0xBC and F-d6032bc0598f still execute; return 5.

6. **Accepted −1 request.**  
   Validation returns nonzero for −1. Expected: return −1 without any of the three downstream operations. This is not evidence that validation rejected the request.

7. **Negative nonzero validation result.**  
   Validation returns −7 for position 2 with bound 6. Expected: continuation, not rejection; the validator’s −7 is not the final returned position.

8. **Preliminary status is nonzero.**  
   The check field converts to nonzero and slot 0x40 returns a valid reference containing a nonzero first word. Expected: call F-feee9156a257 before validation or special selection. Under its supplied non-return annotation, no later stage is reached.

9. **Preliminary dispatch changes the bound.**  
   Slot 0x40 returns a reference containing zero but changes the bound before returning. Expected: subsequent selection uses later bound reads; the pre-call bound is not preserved by this function.

## Uncertainties

The following remain **documented_only**:

- Application-level meaning of the object, bound, positions, and dispatched operations.
- Complete object layout, pointer representation, alignment, and calling convention.
- Exact character-conversion rules. Under an eight-bit truncating interpretation, only the low eight bits affect the preliminary condition; a full-word nonzero test is not equivalent.
- Types, origins, and significance of the four opaque trailing values at slot 0xB8.
- Signed-overflow behavior and invalid-memory-access behavior.
- Dependency ownership, mutation, global-state preservation, and failure contracts.
- Whether shared-state mutation can occur concurrently and whether any external synchronization exists.
- Whether the non-return annotation fully characterizes F-feee9156a257 in the original executable.

## Evidence

Established static evidence consists of the supplied recovered function body, its non-return annotation, and the supplied mapping to opaque dependency IDs. These support the branch conditions, explicit arguments, field and dispatch offsets, call order, and normal return-value rules described above.

Position-based or range-based application intent is inference only. No specific container operation is established.

No runtime tests have been performed. The acceptance examples are conditional predictions from static evidence, not observed executions.
