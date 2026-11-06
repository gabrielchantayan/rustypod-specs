<!-- Trusted provenance: F-3ab3f882cc2a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3ab3f882cc2a

## Purpose

F-3ab3f882cc2a performs an ordered sequence of dependency calls, obtains a working object, invokes a dependency on an embedded field, and conditionally consumes one indexed record from that object. Consuming a record decrements a 32-bit count and, depending on a one-byte selector, copies either two bytes or one byte into current-state fields. Every execution of the nonzero-count branch also writes a one-byte state marker.

Afterward, the function examines an associated pointer and its first 32-bit word. If both are nonzero, it invokes an indirect external target.

This describes the behavior established by the supplied static representation. Interpretations such as “restore previous state,” “pop a history entry,” or “notify an observer” are plausible intent, not established semantics.

## Interface

### Inputs and output

The function has no explicit parameters and no explicit return value.

Its effective inputs are:

- Values returned by dependencies.
- Memory reachable through the working-object address returned by the final call to F-bb50a072e902.
- The external indirect-call target used at the end.
- Any ambient state read by dependencies.

The function does not report success or failure through a returned status. Dependency results are used or discarded as described below.

### Widths and working-object layout

The supplied representation uses 32-bit scalar values for the initial dependency result, its transformed result, the working-object address, the count, and the first word reached through the associated pointer. Individual state fields and record components are one byte wide.

Offsets below are byte offsets from the working-object base. Field names are descriptive labels for this contract, not recovered type declarations.

| Offset | Width | Established use |
|---|---:|---|
| +0x24 | 1 byte | Written with the value 1 when the count is nonzero. |
| +0x28 | Pointer-sized field in the apparent 32-bit model | Its address is passed to F-338d16f9ce72. Its contents are subsequently read as an associated pointer. |
| +0x54 | 1 byte | Receives the selected record’s first byte when the selector is zero. |
| +0x55 | 1 byte | Receives the selected record’s second byte when the selector is zero. |
| +0x56 | 1 byte | Receives the selected record’s third byte when the selector is one. |
| +0x5c | 32 bits | Tested as a count, then reread, decremented, and stored when initially nonzero. |
| +0x60 onward | Three-byte record stride | Source area for the indexed byte transfers. No extent is supplied. |
| +0x90 | 1 byte | Selector controlling which record components are copied. |

The apparent pointer width is supported by the supplied 32-bit address and pointer representation. A complete platform ABI or independently verified object declaration is not provided.

### Preconditions

For defined, ordinary completion:

- Every invoked dependency must be callable with the indicated arguments.
- The final working-object address must support the accesses actually reached.
- Every reached indexed source address must be readable.
- Every reached destination field must be writable.
- A nonnull associated pointer must support a 32-bit read at its start.
- If the indirect-call condition is satisfied, the external target must be valid under its actual calling convention.

The function performs no explicit checks that establish these conditions. In particular, it does not check the working-object address for null, validate record bounds, or validate the indirect target.

## Behavior

### 1. Ordered dependency sequence

The function begins with the following calls, in this order:

1. Call F-bb50a072e902 with the scalar argument zero.
2. Pass that call’s 32-bit result to F-5b4ae75440ab as its first argument, with the scalar value one as its second argument.
3. Retain the 32-bit result of F-5b4ae75440ab.
4. Call F-84055b679da8 with no explicit arguments.
5. Call F-bb50a072e902 again with zero. Discard this call’s result.
6. Call F-5032ac6bc151 with no explicit arguments.
7. Call F-ae23f27a31f9 with the retained result from step 3.
8. Call F-11f610542ac8 with that same retained value.
9. Call F-bb50a072e902 a third time with zero. Use this result as the working-object address.

The three results of F-bb50a072e902 are not established to be equal. The second call remains part of the behavior even though its return value is unused: its side effects, if any, are unknown.

The retained value is passed to two dependencies. This alone does not establish ownership transfer, borrowing, reference counting, or validity after either call.

### 2. Process the embedded field

The function calls F-338d16f9ce72 with the address at working-object offset +0x28.

This is the address of the field, not its contents. The function reads the associated pointer stored there only later, after F-338d16f9ce72 and the conditional count processing.

Consequently, the pointer examined near the end may differ from the field’s value before this dependency call. No preservation of the object or other memory by this dependency is established.

### 3. Test and decrement the count

The function reads the 32-bit count at +0x5c and tests it against zero.

If that read is zero, the function skips the decrement, selector handling, indexed byte transfers, and marker write. It proceeds directly to the associated-pointer check.

If that read is nonzero, it:

1. Reads the count again.
2. Subtracts one.
3. Stores the result back to +0x5c.
4. Uses that newly calculated result as the record index for any subsequent byte transfer.

The supplied representation therefore distinguishes the initial test from the read used for subtraction. Their equality depends on a stable-memory assumption; the contract does not collapse them into an atomic operation.

There is no positive-count check. A negative signed count is nonzero and enters the branch. There is also no explicit lower-bound, upper-bound, or arithmetic-overflow check.

### 4. Select and copy record components

The indexed record begins at an offset of +0x60 plus three times the decremented count.

After storing the decremented count, the function reads the one-byte selector at +0x90.

- **Selector zero:** Read the indexed record’s first byte and write it to +0x54. Then read the record’s second byte and write it to +0x55. This branch does not directly write +0x56.
- **Initial selector read nonzero:** Read the selector again and compare this second reading with one.
  - If the second reading is one, read the indexed record’s third byte and write it to +0x56.
  - Otherwise, perform no indexed byte transfer.

The zero-selector branch uses two ordered read/write pairs, not an established snapshot of both source bytes. If unusual index values cause source and destination overlap, the first write can matter to a later read. No independent source buffer is created.

For stable selector memory, the three ordinary cases are zero, one, and all other byte values. With changing selector memory, a nonzero first read followed by a zero second read does not enter the zero-selector branch; it performs no byte transfer.

Finally, for every execution of the nonzero-count branch, the function writes the byte value one to +0x24. This write occurs after any selected byte transfers and also occurs when the selector causes no transfer.

### 5. Examine the associated pointer and conditionally invoke the external target

After count processing, the function loads the pointer stored at +0x28.

- If the pointer is null, it does not dereference it and does not invoke the indirect target.
- If the pointer is nonnull, it reads the first 32-bit word at that pointer.
- If that word is zero, it does not invoke the indirect target.
- If that word is nonzero, it invokes an external indirect target and then returns, assuming the call returns normally.

The first word is a gate for the call. The supplied representation does not show that word being used as the call target. The indirect target comes from a separate external location.

No explicit arguments or returned value are shown for the indirect invocation. Its complete calling convention and target behavior remain unknown.

## Side effects

### Direct memory effects

The function directly writes:

- The decremented 32-bit count at +0x5c, when the tested count is nonzero.
- Either two bytes at +0x54 and +0x55, one byte at +0x56, or none of those bytes, depending on selector reads.
- The byte value one at +0x24 whenever it enters the nonzero-count branch.

It does not directly clear the marker when the count is zero. It also does not clear destination bytes that are not selected.

The sequence is not transactional. The count is written before any record-byte read. A failure during a later access can therefore occur after earlier state changes have already happened. No rollback is shown.

### Dependency effects and ownership

Dependencies and the indirect target may have additional observable effects. Those effects are not specified by this function’s body.

No explicit allocation, release, or destruction operation is identifiable from the direct memory operations. Dependency names and argument reuse do not establish whether dependencies allocate, release, retain, or transfer ownership.

No synchronization, locking, or atomic access is shown. This contract does not establish safe concurrent mutation of the object.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-bb50a072e902 | Called three times with zero. The first result is transformed, the second is discarded, and the third supplies the working-object address. |
| F-5b4ae75440ab | Receives the first lookup result and the value one; returns a retained 32-bit value. |
| F-84055b679da8 | Called without explicit arguments between transformation and the second lookup. |
| F-5032ac6bc151 | Called without explicit arguments after the second lookup. |
| F-ae23f27a31f9 | Receives the retained transformed value. |
| F-11f610542ac8 | Receives the same retained value after F-ae23f27a31f9. |
| F-338d16f9ce72 | Receives the address of the working object’s +0x28 field before direct count processing. |

The terminal indirect target has no supplied opaque function ID. It cannot be assigned the behavior of any mapped dependency.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Unless stated otherwise, assume valid memory, stable fields during direct processing, representable arithmetic, and dependencies that return normally without altering the scenario’s stated post-dependency state.

1. **Zero count, null associated pointer.**  
   If the count is zero after F-338d16f9ce72 returns and the associated pointer is null when later read, the function performs no direct count, destination-byte, or marker writes and makes no terminal indirect call. All preceding dependency calls still occur.

2. **Count two, selector zero.**  
   The stored count becomes one. The selected record’s first byte is read from +0x63 and written to +0x54; its second byte is then read from +0x64 and written to +0x55. The marker becomes one. The function does not directly write +0x56.

3. **Count one, selector one.**  
   The stored count becomes zero. The third byte of record zero, at +0x62, is copied to +0x56. The marker becomes one. The function does not directly write +0x54 or +0x55.

4. **Count three, selector two.**  
   The stored count becomes two. No record component is copied. The marker nevertheless becomes one. This case must not be treated as a rejected operation with no state change.

5. **Nonzero selector changes before its second read.**  
   If the first selector read is two and the second is one, the third-component transfer occurs. If the first read is two and the second is zero, no component is transferred.

6. **Zero count with an enabled associated object.**  
   A zero count does not prevent the terminal invocation. If the associated pointer is nonnull and its first 32-bit word is nonzero, the external target is invoked after the count branch is skipped.

7. **Negative count.**  
   A count of negative one enters the decrement branch and, if ordinary signed subtraction is representable, produces negative two. With a stable zero selector, the corresponding source offsets are +0x5a and +0x5b. The implementation shows no rejection of this case; whether those accesses are valid depends on the actual object and intended layout.

## Uncertainties

- **documented_only — Arithmetic boundaries:** The representation uses signed 32-bit arithmetic for decrementing and multiplying the index by three. Exact target behavior at signed overflow or address overflow is not independently established. No portable wraparound guarantee is asserted.
- **documented_only — Record capacity:** No array length or permitted count range is supplied. Three-byte stride does not establish storage capacity.
- **documented_only — Selector meaning:** Zero and one select different transfers; domain-level meanings are unknown.
- **documented_only — Marker meaning:** The write of one is established. Whether it denotes validity, dirtiness, pending work, or another state is unknown.
- **documented_only — Dependency semantics:** Global preservation, mutation, ownership effects, failure behavior, and reentrancy are unknown.
- **documented_only — Indirect invocation:** Target identity, full calling convention, implicit machine-state inputs, and effects are not supplied.
- **Inferred intent:** The decrement-before-index pattern is consistent with consuming a previous record. It does not establish stack invariants or prove that callers maintain a nonnegative count.

## Evidence

The supplied static representation establishes the explicit dependency order, direct field accesses, byte-transfer branches, decrement-before-index behavior, marker write, and terminal invocation gate.

It does not establish allocation sizes, dependency internals, ownership contracts, synchronization guarantees, or intended domain semantics. The terminal indirect transfer is represented as a call, but its full interface is unavailable.

No runtime tests have been performed. The acceptance examples are deductions under their stated conditions, not observations of execution.
