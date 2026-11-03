<!-- Trusted provenance: F-6b5b32cae8cb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6b5b32cae8cb

## Purpose

F-6b5b32cae8cb selects a position associated with an object, conditionally invokes object-dispatched operations, and returns the selected position.

A distinguished request can select the last counted position without validation. Other requests pass through a validation-like dependency. A separate entry flag diverts execution to a path annotated as nonreturning.

The established contract concerns selection, control flow, field reads, call arguments, and call ordering. The evidence does not establish that the function deletes, truncates, destroys, moves, or releases an element.

A normal result of −1 is ambiguous: it can indicate validation rejection or an accepted position for which subsequent operations are skipped.

## Interface

### Inputs and numeric representation

The function receives:

- **Object reference:** a pointer to a representation accessed through integer-sized slots.
- **Requested position:** a signed integer passed by value.

The supplied representation supports an apparent signed 32-bit interpretation for the requested position, selected position, count-like field, validation result, and function result. Under that interpretation:

- The distinguished request is **2,147,483,647**, represented as **0x7fffffff**.
- The skip position and rejection result are **−1**.
- The indirect calls receive a constant with bit pattern **0xffffffff**. That pattern represents −1 when interpreted as a signed 32-bit integer.

A separate target ABI and the indirect callees’ parameter declarations are unavailable. Consequently, the bit pattern passed to an indirect call is established more directly than its interpretation by that callee.

### Object representation

| Location | Established use |
|---|---|
| First integer-sized slot; apparent byte offset 0 | Supplies the dispatch-table address for indirect calls. |
| Second integer-sized slot; apparent byte offset 4 | Supplies a signed count-like value. |
| Fifth integer-sized slot; apparent byte offset 16 | Is converted to a character-sized type and compared with zero. |

The byte offsets above assume four-byte integer slots. Pointer representation, alignment, and portability to other layouts are not independently established.

The flag decision is specifically a comparison of the **character-converted fifth-slot value** with zero. It is not a direct comparison of the full integer slot with zero.

An eight-bit character width alone does **not** establish a low-order-byte test. If the character type is signed and the integer value is outside its representable range, conversion can have implementation-defined behavior rather than necessarily preserving the low eight bits. Without a target conversion contract, neither truncation nor equivalence to reading a particular object byte is guaranteed.

Two dispatch entries are used:

| Dispatch-table byte offset | Arguments supplied |
|---|---|
| **0xb8** | Object reference, selected position plus one, count-like value minus one, constant 0xffffffff. |
| **0xbc** | Object reference, constant 0xffffffff. |

These offsets distinguish the unresolved dependencies; they do not establish their semantic identities.

### Output

On normal completion, the function returns:

- **−1** if F-402eef64af83 returns zero; otherwise
- the selected position, including −1 if that position was accepted.

The results of the indirect calls and F-d6032bc0598f do not contribute to the returned value.

The flagged entry path has no established normal return.

### Preconditions

Defined execution requires:

- A valid object reference for every field access reached.
- A usable integer-to-character conversion contract for interpreting the flag decision.
- Valid dispatch state and callable entries for every indirect call reached.
- Satisfaction of reached dependencies’ unknown preconditions.
- Representable signed arithmetic for reached additions and subtractions, unless an external execution contract specifies otherwise.

No local null check, layout check, dispatch-entry check, or recovery mechanism is visible.

## Behavior

### Entry flag and diverted path

The function first reads the fifth integer-sized slot, converts its value to the character-sized type, and compares the converted value with zero.

If the converted value is nonzero:

1. F-1b24ca874f69 receives the object reference and the original requested position.
2. If it returns normally, F-feee9156a257 is invoked without explicit arguments.
3. F-feee9156a257 is annotated as nonreturning.

Under that annotation, no position selection, validation, indirect operation, or final helper follows on this path.

The evidence does not identify the path as an assertion, exception, abort, or ordinary error report. F-1b24ca874f69 may itself fail to return. Its effects are not constrained by the visible body.

If the converted value is zero, ordinary processing continues. If an out-of-range character conversion does not complete normally under the target’s rules, the evidence provides no additional handling.

### Position selection and validation

The distinguished request receives special treatment only when the count-like field is strictly positive.

For request **0x7fffffff**:

- The function reads the count-like field for the positivity test.
- If positive, it reads the field for the last-position calculation and selects that value minus one.
- F-402eef64af83 is not called on this shortcut.

The positivity test and selection are separate accesses in the supplied representation. No atomic-snapshot or concurrency guarantee is established.

For all other cases, including a distinguished request whose count test is zero or negative:

1. F-402eef64af83 receives the object reference and requested position.
2. A zero result causes an immediate return of −1.
3. A nonzero result accepts the requested position as the selected position.

For a request other than the distinguished value, the shortcut does not require a count read before validation. The validation result is used only as a zero/nonzero gate; it does not replace the selected position.

No additional local bounds check follows successful validation. In particular:

- Requests below −1 are not independently rejected.
- Positions at or above the count are not independently rejected.
- A distinguished request with a nonpositive count is not automatically converted to −1.
- Validation may alter state subsequently read by this function.

### Accepted position −1

After shortcut selection or successful validation, a selected position of −1 causes a normal return of −1 without either indirect call or F-d6032bc0598f.

A request of −1 does not bypass validation. It first reaches F-402eef64af83; only a nonzero validation result reaches this accepted-skip case.

The numeric return alone cannot distinguish this case from validation rejection.

### Conditional first indirect operation

For a selected position other than −1, the function compares:

**selected position + 1 < current count-like value**

The count is read after any validation call. If the comparison is false, the dispatch entry at **0xb8** is skipped.

If true, that entry receives:

1. The object reference.
2. The selected position plus one.
3. The count-like value read for the call, minus one.
4. The constant 0xffffffff.

The position-shaped arguments resemble endpoints following the selected position. This is inferred intent only. No inclusive-range interpretation, element unit, or operation such as movement or deletion is established.

The count used for the comparison is not guaranteed to be the value used for the call’s endpoint. The representation accesses the field again for that endpoint. Likewise, the relative evaluation order of dispatch resolution and argument field reads within the call expression is not established.

The guaranteed ordering is that the comparison determines whether the call is reached, and the call must return normally before the next operation proceeds.

### Second indirect operation and final helper

For every selected position other than −1, after the optional first indirect call returns:

1. The dispatch entry at **0xbc** receives the object reference and 0xffffffff.
2. If that call returns normally, F-d6032bc0598f receives the object reference and selected position.
3. If the helper returns normally, the function returns the selected position.

The second dispatch is resolved at its own call site. The function does not establish that it uses dispatch state captured before the first indirect call. A change to the object’s dispatch reference by the first operation can therefore affect the second lookup, provided subsequent accesses remain valid.

No later count or position validation is visible.

### Arithmetic boundaries

Under a stable signed 32-bit interpretation:

- A strictly positive count makes the shortcut’s subtraction by one representable.
- A selected position other than −1 still requires addition by one for the later comparison.
- If validation accepts **2,147,483,647**, that addition exceeds the signed 32-bit maximum. No wrapped comparison or subsequent call sequence is specified.
- A count of **−2,147,483,648** would make subtraction by one unrepresentable if used as a reached endpoint value.

The last case requires care: with a stable minimum count and a representable selected-position addition, the range comparison cannot be true, so the endpoint subtraction is not reached. The overflow risk remains if a separately observed endpoint count becomes the minimum after an earlier comparison succeeded. Whether such changes are possible under a valid external execution model is unknown.

Similarly, a positive count observation does not by itself constrain a later count read if external mutation is permitted. The function provides no snapshot mechanism.

## Side effects

The visible body performs no direct assignment to object fields or explicit global state. Its local position selection does not itself modify the object.

This does not make the function read-only:

- F-1b24ca874f69 may change state before the nonreturning dependency.
- F-402eef64af83 may change state even when rejecting a request.
- Either indirect dependency may change object fields, dispatch state, external resources, or globals.
- F-d6032bc0598f may change state before the normal return.

No dependency is assumed to preserve globals.

### Ownership and lifetime

The same object reference is passed to reached dependencies. No explicit allocation, object copy, retain, release, or deallocation appears in the visible body.

That absence does not establish ownership preservation by dependencies. Their ownership and release contracts remain unknown.

For normal completion, the object must remain usable for all later accesses and calls:

- Successful validation can be followed by count and dispatch reads.
- The first indirect operation can be followed by a fresh dispatch lookup.
- The second indirect operation is followed by passing the object to F-d6032bc0598f.

There is no visible protection against a dependency invalidating the object too early.

### Errors and interrupted execution

The only explicit local rejection result is −1 after zero validation.

No rollback is visible. If a dependency faults, throws, terminates, or otherwise fails to return normally, this function establishes no reversal of prior effects and no normal result for that execution.

## Dependencies

| Opaque ID or unresolved entry | Visible role | Limits of established behavior |
|---|---|---|
| **F-1b24ca874f69** | Receives original inputs when the converted entry flag is nonzero. | Return behavior, state changes, and ownership effects unknown. |
| **F-feee9156a257** | Invoked after F-1b24ca874f69 returns. | Annotated nonreturning; mechanism and effects unknown. |
| **F-402eef64af83** | Gates requests not accepted through the shortcut. | Zero rejects; nonzero accepts. No stronger bounds or validity guarantee established. |
| **F-d6032bc0598f** | Receives object and selected position after the second indirect call. | Result unused; effects and lifetime requirements unknown. |
| **Unresolved entry 0xb8** | Conditional operation with two position-shaped arguments. | Function identity, parameter semantics, and effects unknown. |
| **Unresolved entry 0xbc** | Operation for every selected position other than −1. | Function identity, parameter semantics, and effects unknown. |

The unresolved entries have no supplied opaque function identities. Their table offsets are locators, not inferred names.

## Acceptance examples

These examples are conditional static deductions, not executed tests. Unless stated otherwise, the flag conversion produces zero, reached dependencies return normally, object and dispatch references remain valid, and the count remains stable.

1. **Distinguished request, count 4.**  
   Request 0x7fffffff selects position 3 without validation. The first indirect operation is skipped because 4 is not less than 4. Entry 0xbc receives the object and 0xffffffff; F-d6032bc0598f then receives position 3. The result is 3.

2. **Accepted interior position.**  
   With count 5, request 1, and nonzero validation, entry 0xb8 receives the object, positions 2 and 4, and 0xffffffff. Entry 0xbc follows, then F-d6032bc0598f receives position 1. The result is 1.

3. **Validation rejection with effects.**  
   If F-402eef64af83 changes state and returns zero, the result is −1. Neither indirect entry nor the final helper is reached. The validation effects are not rolled back.

4. **Accepted skip position.**  
   Request −1 reaches validation. If accepted, it returns −1 without either indirect operation or the final helper.

5. **Distinguished request, count zero.**  
   Validation receives 0x7fffffff. Rejection returns −1. Acceptance reaches an unrepresentable signed 32-bit addition; no wrapping behavior or normal result is specified.

6. **Validation changes the count.**  
   For request 1, suppose validation succeeds and sets the count to 3. If it remains 3, the first indirect operation receives position-shaped arguments 2 and 2. The earlier count, if any, does not govern that comparison.

7. **Accepted negative position other than −1.**  
   With count 3 and accepted request −2, the comparison succeeds. Entry 0xb8 receives position-shaped arguments −1 and 2. No local rule excludes them.

8. **Flag conversion produces nonzero.**  
   F-1b24ca874f69 receives the original request. If it returns, F-feee9156a257 follows. Ordinary position processing is not reached under the nonreturning annotation.

9. **Out-of-range flag value under a signed eight-bit character model.**  
   A fifth-slot integer value of 256 is outside that character range. Its low eight bits being zero does not establish which branch runs. If the target conversion produces zero, ordinary processing follows; if it produces nonzero, the diverted path follows. Any other permitted conversion outcome requires the target contract. No unconditional branch expectation is justified from character width alone.

## Uncertainties

The following remain **documented_only**:

- Exact integer, pointer, and character widths; alignment; endianness; and target ABI.
- Whether the character type is signed and how out-of-range conversions behave.
- Whether any target conversion rule preserves low-order bits.
- Object type and meanings of the count-like and flag-like fields.
- Indirect dependency identities, parameter types, and operation semantics.
- Dependency effects on globals, dispatch state, object lifetime, and ownership.
- Surrounding API meaning of a −1 result.
- The nonreturning mechanism and behavior if its annotation is violated.
- External invariants excluding invalid positions or arithmetic overflow.
- Concurrency, synchronization, and stability of repeated field observations.

Inferred intent is limited to last-position selection, validation-like gating, and a range-shaped operation. Those interpretations add no guarantees.

## Evidence

The supplied static representation establishes the flag conversion and comparison, distinguished constants, integer-slot accesses, branching conditions, dependency arguments, ordering between separate calls, and normal return selection.

The flag evidence establishes a comparison of a character-converted value with zero—not an unconditional low-order-byte test. A target-specific conversion contract would be needed for that stronger claim.

Dependency implementations, indirect-call contracts, and a separate ABI specification are unavailable. Nonreturning behavior is supplied by an annotation rather than runtime observation.

No runtime tests have been performed. All acceptance examples are conditional deductions from static evidence.
