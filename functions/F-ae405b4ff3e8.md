<!-- Trusted provenance: F-ae405b4ff3e8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ae405b4ff3e8

## Purpose

F-ae405b4ff3e8 conditionally updates shared state and invokes an operation on an object obtained from F-b7af80e829ec. Its decision depends on a two-byte selector field in a supplied descriptor and the first one or two bytes of a referenced payload.

The established behavior is:

- Accept descriptor selectors equal to 1 or 2.
- Accept a first payload byte in the inclusive range 0–2.
- For an accepted payload, write a two-byte shared-state value and a one-byte shared-state value, then invoke an object operation with the first payload byte.
- Return a 32-bit status of 0 after that operation returns normally.
- Return a 32-bit status of 4 for either an unsupported descriptor selector or an unsupported first payload byte.

These two rejection cases differ observably. An unsupported descriptor selector avoids payload access and dependency invocation. An unsupported first payload byte is rejected only after F-b7af80e829ec has been called.

A possible interpretation is that the routine changes a mode while conditionally retaining or obtaining a related value. That interpretation is inferred intent, not an established meaning for the fields or methods.

## Interface

### Inputs

The function receives two arguments:

| Argument | Representation | Established use |
|---|---|---|
| First argument | Opaque 32-bit value | Not referenced by the supplied function body. |
| Second argument | Address represented by a 32-bit integer in the supplied representation | Identifies the descriptor whose fields are read. |

The descriptor has the following relevant layout:

| Byte offset | Width | Meaning established by use |
|---|---:|---|
| +8 | Pointer-sized; 32 bits in the supplied representation | Address of payload bytes. Read only for selector 1 or 2. |
| +12 | 16 bits | Selector, interpreted as a signed 16-bit value and compared with 1 and 2. |

The payload layout is:

| Byte offset | Width | Meaning established by use |
|---|---:|---|
| +0 | 8 bits, unsigned | Operation argument; only 0, 1, and 2 are accepted. |
| +1 | 8 bits, unsigned | Secondary state byte, read only for selector 2 after the first byte has passed validation. |

The selector is not established to be a payload length. Its accepted values happen to determine whether one or two payload bytes are accessed, but no independent length check occurs.

### Shared state

The routine reads a global pointer slot that identifies a shared-state region. Two fields within that region are relevant:

| Byte offset | Width | Established role |
|---|---:|---|
| +7 | 8 bits | Tested against zero when deciding whether to obtain or reuse a two-byte value; later overwritten with the secondary payload byte or zero. |
| +10 | 16 bits | Either read as a retained value or overwritten with the selected two-byte value. |

The routine captures a destination base from the global pointer slot after F-b7af80e829ec returns. That captured base determines both explicit writes performed by this function.

### Output

The return value is 32 bits:

- **0:** The accepted-input path reached the final indirect operation, and that operation returned normally.
- **4:** The descriptor selector was neither 1 nor 2, or the first payload byte was greater than 2.

A return of 0 does not independently establish that the indirect operation succeeded according to its own contract. This function does not inspect that operation’s return value.

### Preconditions

For normal completion, every address actually accessed on the selected path must be valid with suitable access permissions. In particular:

- The descriptor’s selector field must be readable.
- For selector 1 or 2, the descriptor’s payload pointer and first payload byte must be readable.
- For selector 2 with an accepted first byte, the second payload byte must be readable.
- An accepted first byte requires writable destination fields at the captured shared-state base.
- Any shared-state fields used for the retention decision must be readable.
- F-b7af80e829ec must supply an object whose required indirect operation entries are valid and callable.

There are no explicit null checks or recoverable handling for invalid addresses. Alignment requirements, fault behavior, and calling-convention obligations beyond the visible argument usage are not established.

## Behavior

### 1. Read and classify the descriptor selector

The first externally relevant access is a 16-bit read at descriptor offset +12.

If that value is neither 1 nor 2, the routine returns 4. On this path, it does not explicitly:

- Read the payload pointer.
- Read either payload byte.
- Invoke F-b7af80e829ec.
- Read the shared-state pointer slot.
- Write either shared-state field.
- Invoke either object operation.

The first argument does not affect this decision.

### 2. Capture the payload address and first byte

For selector 1 or 2, the routine reads the payload pointer at descriptor offset +8, then reads the first payload byte from that address.

Both the selector and first payload byte are captured before the dependency call. Later changes to the descriptor selector, descriptor payload pointer, or first payload byte do not cause those values to be reread.

The captured payload address remains relevant: selector 2 may require a subsequent read at payload offset +1.

### 3. Obtain the object and capture the destination base

The routine invokes F-b7af80e829ec with no explicit arguments and retains its returned object pointer.

After that call returns, it reads the global shared-state pointer slot and captures the resulting base as the destination for later writes.

This ordering is significant:

- F-b7af80e829ec is called even if the already-read first payload byte will be rejected.
- Changes made by that dependency to the global pointer slot occur before the destination base is captured.
- Changes made by that dependency to the second payload byte can affect subsequent behavior.
- Changes made by that dependency to the first payload byte do not alter the captured operation argument.

No preservation of global state, descriptor storage, payload storage, or object state by F-b7af80e829ec is assumed.

### 4. Validate the first payload byte

The routine accepts the captured byte only when it is less than 3 under an unsigned comparison.

If it is in the range 3–255, the routine returns 4. It has already called F-b7af80e829ec and captured the shared-state destination base, but it does not explicitly read the second payload byte, update the shared-state fields, or invoke either object operation.

Dependency side effects are not undone.

### 5. Select the secondary byte and two-byte value

For an accepted first byte, the initial selected secondary byte and selected two-byte value are both zero.

For **selector 1**, those zero values remain selected. The second payload byte is not read, and neither the retained shared-state value nor the query operation is needed.

For **selector 2**, the routine reads payload offset +1 after F-b7af80e829ec has returned.

- If that byte is zero, both selected values remain zero. The routine does not consult the existing shared-state fields to obtain a retained value.
- If that byte is nonzero, the routine checks the one-byte field at shared-state offset +7 through the global pointer slot:
  - If the existing field is zero, it invokes the object’s query-like operation and uses the low 16 bits of its returned value.
  - If the existing field is nonzero, it reads and uses the 16-bit field at shared-state offset +10.

The secondary byte is not normalized to a Boolean. Any nonzero byte, including 255, is retained as that exact byte for the later write.

The pointer accesses used for the retention decision are distinct from the previously captured destination base. With stable global state, they identify the same region. The routine provides no synchronization guarantee if external activity changes the pointer between accesses.

### 6. Write shared state

The routine writes the selected values to the captured destination base in this order:

1. Write the selected 16-bit value at offset +10.
2. Write the selected 8-bit secondary value at offset +7.

The established widths are two bytes and one byte respectively. No other explicit shared-state writes appear in the function.

The destination remains the base captured after F-b7af80e829ec returned. In particular, if the query-like operation changes the global shared-state pointer slot, these writes still target the earlier captured region.

### 7. Invoke the final operation and return

After both writes, the routine invokes the final object operation, passing:

- The retained object pointer as the receiver.
- The captured first payload byte as the operation argument.

The operation argument is therefore 0, 1, or 2.

The final operation entry is obtained from the object’s dispatch table at the time of this invocation. If an earlier query-like operation changed the object’s dispatch table, the final lookup uses the then-current table.

The final operation’s return value is ignored. If the call returns normally, F-ae405b4ff3e8 returns 0.

## Side effects

### Explicit effects

For accepted inputs, the function explicitly overwrites the two shared-state fields before invoking the final object operation.

For rejected inputs, the function performs no explicit writes to those fields. However, rejection of an unsupported first payload byte still permits effects from F-b7af80e829ec.

The function contains no explicit allocation, deallocation, reference-count adjustment, or ownership transfer.

### Dependency effects and ownership

The absence of explicit memory-management operations does not establish that the dependencies allocate nothing or release nothing. Their ownership rules are unknown.

The function relies on the relevant object, payload address, and captured destination remaining usable across intervening calls. It does not visibly acquire ownership, pin storage, or verify continued validity.

Any dependency may have additional effects not described by this body. Such effects can include mutation of shared state or changes to the object’s dispatch table. They are not assumed away.

### Ordering and partial completion

The significant local order is:

1. Read selector.
2. If eligible, read payload pointer and first byte.
3. Call F-b7af80e829ec.
4. Capture the destination base.
5. Validate the captured first byte.
6. If required, read the second byte and obtain the selected two-byte value.
7. Write the two-byte field.
8. Write the one-byte field.
9. Invoke the final operation.
10. Return 0.

No rollback is visible. If an operation fails to return normally, earlier dependency effects or completed writes need not be reversed.

The two writes are not specified as an atomic pair. Synchronization, concurrent-reader behavior, and hardware memory-ordering guarantees are unknown.

## Dependencies

### F-b7af80e829ec

- Called once for selector 1 or 2, regardless of whether the first payload byte is accepted.
- Receives no explicit arguments.
- Supplies the object pointer used for later indirect dispatch.
- Called before the shared-state destination base is captured.
- Ownership, lifetime guarantees, side effects, and abnormal-completion behavior are unknown.

### F-indirect-50

This document-local opaque ID identifies the query-like indirect dependency selected through the object’s dispatch-table entry at byte offset +80.

- Receives the object pointer.
- Called only when the selector is 2, the first payload byte is accepted, the second payload byte is nonzero, and the consulted shared-state byte at +7 is zero.
- Supplies a value used as a 16-bit bit pattern.
- Is not assumed to preserve globals, storage, or the object’s dispatch table.

### F-indirect-4c

This document-local opaque ID identifies the final indirect dependency selected through the object’s dispatch-table entry at byte offset +76.

- Receives the object pointer and the accepted first payload byte.
- Called after the two explicit shared-state writes.
- Its return value does not affect the wrapper’s status.
- Its semantic meaning, failure signaling, ownership behavior, and side effects are unknown.

These two document-local IDs identify call sites, not uniquely resolved implementation targets.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests. Unless stated otherwise, assume valid storage, stable external state, and dependencies that return normally.

1. **Unsupported selector**

   Given a selector of 0, the function returns 4 without reading a payload pointer or calling F-b7af80e829ec. The payload pointer need not identify readable storage on this path.

2. **Unsupported first byte**

   Given selector 1 and a first payload byte of 3, the function calls F-b7af80e829ec, captures the shared-state base, and returns 4. It does not explicitly update the shared-state fields or invoke F-indirect-4c.

3. **Selector 1 clears both selected fields**

   Given selector 1 and a first byte of 2, the function writes zero to the 16-bit field, then zero to the 8-bit field, invokes F-indirect-4c with argument 2, and returns 0. No second payload byte is required.

4. **Selector 2 with a zero secondary byte**

   Given selector 2 and payload bytes 0 and 0, the function selects zero for both fields without obtaining a retained value or invoking F-indirect-50. It performs the ordered writes, invokes F-indirect-4c with argument 0, and returns 0.

5. **Selector 2 obtains a new value**

   Given selector 2, payload bytes 1 and 9, and an existing shared-state byte of zero, the function invokes F-indirect-50. If its return contributes the bit pattern 0xABCD in its low 16 bits, the function writes 0xABCD at +10, then 9 at +7, before invoking F-indirect-4c with argument 1.

6. **Selector 2 retains an existing value**

   Given selector 2, payload bytes 2 and 255, an existing shared-state byte of 7, and an existing two-byte value of 0x1234, the function does not invoke F-indirect-50. It writes 0x1234 and then 255, invokes F-indirect-4c with argument 2, and returns 0.

7. **Dependency mutation of the payload**

   Given selector 2 and an initially accepted first byte, if F-b7af80e829ec changes the first byte to 255 and the second byte to zero, the captured first byte remains the final operation argument. The subsequently read zero second byte selects zero for both shared-state fields.

8. **Query changes the global destination pointer**

   If F-indirect-50 changes the global pointer slot to identify another region, the explicit writes still target the region captured before that query. This criterion assumes the captured region remains writable.

## Uncertainties

The following are **documented_only**:

- Application-level meanings of the selector, payload bytes, shared-state fields, and status values.
- The semantic role and validity requirements of the unused first argument outside this function.
- Ownership and lifetime contracts for all referenced storage.
- Actual targets and contracts of the indirect dependencies.
- Endianness, alignment constraints, and environmental fault handling.
- Thread safety, reentrancy, and synchronization requirements.
- Whether dependencies preserve or modify globals, release memory, or report errors through mechanisms not inspected here.

The visible behavior does not justify replacing either rejection status with a more specific error meaning. It also does not justify treating a normal return of 0 as proof of success beyond completion of this function’s observed control path.

## Evidence

Established static evidence consists of the supplied function body and the supplied mapping for F-b7af80e829ec. It supports the field widths and offsets, branch conditions, dependency-call placement, destination capture, explicit write order, and literal return values described above.

The mode-change interpretation and the labels “query-like” and “retained value” are explanatory inferences, not proven domain semantics.

No runtime tests have been performed. The acceptance examples specify conditional observable behavior; they are not reports of measured execution.
