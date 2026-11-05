<!-- Trusted provenance: F-5bf0cc325ea6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5bf0cc325ea6

## Purpose

F-5bf0cc325ea6 selects a record, initializes two one-byte fields according to a mode argument, invokes an operation using a value read from a caller-supplied object, and records the operation’s result.

The established behavior distinguishes three outcomes:

- No record is selected: return −1 without performing the subsequent record initialization or operation.
- The operation returns a positive signed result: retain that result in the selected record and return 0.
- The operation returns zero or a negative signed result: leave the record’s result field at zero. Return −1 for a zero result; for a negative result, return its character-width conversion widened back to the return type.

The record’s purpose, the meaning of its flags, and the underlying operation’s domain are not established. Interpreting this routine as acquiring or opening a resource is inferred intent, not part of the established contract.

## Interface

The interface has six positional arguments and returns an integer. The supplied representation uses 32-bit integers and four-byte opaque values, with byte-sized record flags. These widths describe the represented interface; the source alone does not independently establish the executable ABI.

| Position | Contract |
|---|---|
| First | A 32-bit address-bearing value identifying an object. When selection succeeds, a four-byte value is read at byte offset 52 from this address and passed to F-04c06cd369a6. |
| Second | An opaque four-byte value forwarded unchanged to F-04c06cd369a6. This routine does not directly dereference it. |
| Third | A signed 32-bit mode value. Exact values 1 and 3 control two record flags. The original value is also forwarded unchanged to F-04c06cd369a6. |
| Fourth | A signed 32-bit option value. Nonzero values are forwarded unchanged. Zero requests a default selected through F-f1edefb7c207. |
| Fifth | An opaque four-byte argument not read or forwarded by the visible body. |
| Sixth | An opaque four-byte value forwarded unchanged as the final argument to F-04c06cd369a6. This routine does not directly dereference it. |

The return value is represented as a signed 32-bit integer. It is not the positive result returned by F-04c06cd369a6; that value is stored in the selected record instead.

### Record layout

A shared global value supplies the base address of a record array. The selected record address is calculated using a stride of 24 bytes.

Only these fields are directly accessed:

| Byte offset within record | Width | Access |
|---|---:|---|
| 12 | 4 bytes | Written with the operation result, then overwritten with zero if that result is nonpositive |
| 17 | 1 byte | Cleared, then set to 1 only when the third argument equals 1 |
| 18 | 1 byte | Cleared, then set to 1 only when the third argument equals 3 |

No other record fields are directly initialized by this routine.

### Preconditions for defined execution

If selection succeeds, the shared base and selected index must identify writable storage for the listed fields. The first argument must identify readable storage for the four-byte access at offset 52 when that access occurs.

The implementation supplies no visible bounds, alignment, overflow, or pointer-validity checks. The required validity of the remaining values depends on the callees and is unknown.

## Behavior

### 1. Select a record

The routine first calls F-36a285714182 with no explicit arguments.

If its result is exactly −1:

- Return −1.
- Do not directly read the record-array base.
- Do not directly read the object addressed by the first argument.
- Do not initialize record flags.
- Do not call either F-f1edefb7c207 or F-04c06cd369a6.

These exclusions concern this routine’s visible actions. F-36a285714182 may itself have side effects.

Every selection result other than −1 proceeds to address calculation. There is no visible rejection of other negative values or excessively large values.

The selected address is formed from the shared base plus the selection result multiplied by 24. The base is obtained after F-36a285714182 returns. Accordingly, a change to that global made during selection can affect which address is used.

Multiplication and addition have no visible range checks. Their behavior outside the valid address-calculation range is not established; a portable wraparound guarantee must not be inferred from the representation.

### 2. Initialize mode flags

At the selected record address, the routine performs these byte writes in order:

1. Write zero at offset 17.
2. Write zero at offset 18.
3. If the third argument equals 1, write one at offset 17.
4. Otherwise, if the third argument equals 3, write one at offset 18.

Immediately after these writes, the directly established flag values are:

| Third argument | Offset 17 | Offset 18 |
|---|---:|---:|
| 1 | 1 | 0 |
| 3 | 0 | 1 |
| Any other value | 0 | 0 |

Mode handling uses exact equality, not bit testing. A value containing either numeric value as part of a larger bit pattern receives no special treatment.

The routine does not normalize the third argument before forwarding it. A mode other than 1 or 3 still reaches F-04c06cd369a6 unchanged.

The table describes the state before later calls. Those calls are not known to preserve the flags.

### 3. Resolve a zero option

If the fourth argument is nonzero, use it unchanged and do not call F-f1edefb7c207.

If the fourth argument is zero, call F-f1edefb7c207 with no explicit arguments:

- A zero result selects the numeric option 64.
- Any nonzero result selects the numeric option 512.

There is no distinction between positive and negative nonzero results from this dependency.

The flag writes precede this optional call. This routine performs no direct rollback of those writes if subsequent processing fails.

### 4. Invoke the underlying operation

After flag initialization and any default-option resolution, read four bytes at offset 52 from the address supplied as the first argument.

Call F-04c06cd369a6 with five arguments in this order:

1. The four-byte value just read from offset 52.
2. The original second argument.
3. The original third argument.
4. The original nonzero fourth argument, or the resolved default of 64 or 512.
5. The original sixth argument.

The fifth argument of F-5bf0cc325ea6 is not forwarded.

The object read occurs after the optional F-f1edefb7c207 call. If that dependency changes the object’s contents, the later read can observe the changed value. No preservation guarantee is established.

### 5. Store and classify the result

Treat the result of F-04c06cd369a6 as a signed 32-bit integer.

First, write that result into the four-byte field at offset 12 of the selected record. Then classify it:

- **Positive:** retain the stored result and return 0.
- **Zero:** write zero to offset 12 again and return −1.
- **Negative:** overwrite offset 12 with zero and return the result converted through a character-sized type, then widened to the integer return type.

The negative conversion is not equivalent to returning the original 32-bit result for all inputs. Its precise numeric outcome depends on character signedness and the target’s narrowing-conversion rules, which are not independently established by the supplied material.

Under a signed-eight-bit, low-byte interpretation, for example, −129 becomes 127 and −256 becomes 0. Thus a negative operation result can produce a nonnegative wrapper return. A caller cannot safely infer that every nonnegative return means the operation returned a positive result.

The initially selected record address is retained across the later calls. The routine does not visibly reload the shared base or repeat selection before writing the result field.

## Side effects

The direct visible effects are:

- One call to F-36a285714182 on every invocation.
- On successful selection, a read of the shared record-array base.
- Two mandatory flag-clearing writes, followed by at most one flag-setting write.
- An optional call to F-f1edefb7c207 when the fourth argument is zero.
- A four-byte read from the first argument’s object at offset 52.
- One call to F-04c06cd369a6.
- A four-byte result-field write, followed by a second four-byte write when the result is nonpositive.

For a negative operation result, the visible write sequence stores the negative value before replacing it with zero. For a zero result, it writes zero twice. These are distinct accesses in the supplied representation. Whether another observer can detect intermediate states depends on synchronization, memory properties, and generated execution behavior not supplied here.

No synchronization mechanism is shown. The routine does not establish atomic initialization of a record or safe concurrent access.

The routine contains no explicit allocation, deallocation, close, release, or ownership-transfer operation. This does not establish that the dependencies lack such effects. In particular:

- Selection may reserve or mutate a record.
- The underlying operation may acquire resources.
- Any dependency may change shared state or memory reachable through other means.
- No explicit release of the selected record is shown after an unsuccessful operation.

Ownership and cleanup obligations remain documented_only.

Aliasing is not prohibited or checked. If the first argument’s object overlaps the selected record or other shared state, earlier writes or callee effects may influence the later object read. The established contract is the access order, not an assumption that the storage regions are independent.

## Dependencies

| Opaque function ID | Invocation and established use | Unknowns |
|---|---|---|
| F-36a285714182 | Called first, without explicit arguments. Exactly −1 aborts further processing; every other result selects an address using a 24-byte stride. | Valid result range, record reservation semantics, global changes, ownership, failure causes. |
| F-f1edefb7c207 | Called only after successful selection and flag initialization, and only when the fourth argument is zero. Zero selects 64; nonzero selects 512. | Meaning of the predicate, side effects, preservation of records and input objects. |
| F-04c06cd369a6 | Called once after successful selection, with the five arguments described above. Its signed result controls record storage and the wrapper return. | Argument meanings, result meanings beyond the sign-based handling, resource ownership, shared-state changes, error taxonomy. |

There is no supplied basis for assuming that any dependency preserves globals, preserves record contents, returns normally under every condition, or owns or releases memory.

## Acceptance examples

These examples are conditional static expectations, not executed tests. Unless stated otherwise, assume valid nonoverlapping storage, valid address arithmetic, normal callee returns, and no callee changes to the fields being discussed.

1. **Selection failure**  
   If F-36a285714182 returns −1, the routine returns −1. Neither later dependency is called, and no direct flag or result-field access occurs.

2. **Mode 1 with an explicit option**  
   If selection succeeds, the third argument is 1, the fourth argument is 7, and F-04c06cd369a6 returns 23:
   - The flags become 1 at offset 17 and 0 at offset 18.
   - F-f1edefb7c207 is not called.
   - The underlying operation receives option 7 and mode 1.
   - Offset 12 receives 23.
   - The routine returns 0.

3. **Mode 3 with a zero default predicate**  
   If the third argument is 3, the fourth is zero, and F-f1edefb7c207 returns zero, the flags become 0 and 1, and the underlying operation receives option 64.

4. **Other mode with a nonzero default predicate**  
   If the third argument is 9, the fourth is zero, and F-f1edefb7c207 returns −4, both flags are cleared. The underlying operation receives mode 9 and option 512.

5. **Zero operation result**  
   After successful selection, a zero result from F-04c06cd369a6 causes two zero writes to offset 12 and a return value of −1. The routine does not directly clear or restore the mode flags afterward.

6. **Negative operation result**  
   A result of −2 is first written at offset 12 and then replaced with zero. Under signed-eight-bit conversion with ordinary representable-value preservation, the routine returns −2.

7. **Narrowing can resemble success**  
   Under signed-eight-bit, low-byte conversion, an operation result of −256 leaves offset 12 at zero but makes the routine return 0. This example is conditional on that conversion model.

8. **Address retained across dependency calls**  
   If a later dependency changes the shared array base but leaves the originally selected storage valid, the result-field write still targets the previously calculated record address.

## Uncertainties

The following are documented_only:

- The executable ABI, byte order, alignment requirements, and concrete address model.
- The valid selection-index range and record-array capacity.
- Behavior when address arithmetic overflows or storage is invalid.
- Character signedness and out-of-range integer-to-character conversion.
- The semantic meanings of the flags, options 64 and 512, and positive operation results.
- Dependency side effects, ownership rules, and cleanup responsibilities.
- Concurrency guarantees and external visibility of intermediate writes.
- Whether a selected record remains reserved after the operation returns zero or a negative value.

Inferred intent is limited to record selection, mode marking, default-option choice, and recording an operation result. No stronger resource-management interpretation is required.

## Evidence

The established behavior comes solely from static inspection of the supplied function representation: its call conditions, argument forwarding, exact comparisons, field offsets and widths, access order, return handling, and absence of additional visible checks or cleanup.

The representation supports a 24-byte record stride, byte fields at offsets 17 and 18, a four-byte result field at offset 12, and a four-byte object read at offset 52. It also shows that only selection result −1 stops processing and that only positive operation results remain stored.

No runtime tests have been performed. No dependency implementation, ABI documentation, or independent execution evidence was supplied. Conditional conversion examples are not claims about verified target behavior.
