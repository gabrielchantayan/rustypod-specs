<!-- Trusted provenance: F-d0a663bcd0d2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d0a663bcd0d2

## Purpose

F-d0a663bcd0d2 conditionally delegates an operation on one of four fixed-stride records and then adjusts a shared accounting field by the change in a record-local field.

The visible behavior has three gates:

1. The record selector must be less than four.
2. A record-specific dependency must return a nonzero value.
3. A 32-bit unsigned interpretation of a computed quantity must be less than 102,401.

The function returns either a locally selected status or the 32-bit result of the delegated operation. It does not visibly interpret the delegated result before performing its accounting update.

The likely intent is to serialize access to a selected record and enforce a shared resource limit before an operation. That interpretation is inferred from the call structure and arithmetic. The meanings of the records, accounting units, forwarded argument, and status values are not established.

## Interface

### Inputs

The function takes three arguments:

| Argument | Visible representation | Contract |
|---|---|---|
| Record selector | Unsigned 32-bit integer | Selects one of four records when its value is 0 through 3. Values 4 through 4,294,967,295 skip all record-specific work. |
| Forwarded argument | Opaque 32-bit value | Passed unchanged to F-80172b251073 if delegation occurs. This function does not visibly dereference or otherwise interpret it. |
| Requested adjustment | Signed 32-bit integer | Participates in the admission calculation and is passed unchanged to F-80172b251073. There is no independent nonnegativity check. |

Calling a value “opaque” does not establish that it is a pointer, handle, size, or owned object.

### Output

The output is a 32-bit value with these visible sources:

- **8** when the selector is outside the accepted range.
- **8** when the selector is accepted but F-96d4ea8519eb returns zero.
- **25**, hexadecimal **0x19**, when the admission calculation fails.
- The exact 32-bit result returned by F-80172b251073 when delegation occurs.

There is no visible mechanism to distinguish an identical result value originating from different branches. In particular, delegated results of 8 or 25 remain possible.

### Referenced storage

F-6e90ba5205cc supplies a base address. Relative to that base, the visible layout is:

| Region or field | Relative location | Visible use |
|---|---:|---|
| Record 0 | 0 bytes | Selected record |
| Record 1 | 36 bytes | Selected record |
| Record 2 | 72 bytes | Selected record |
| Record 3 | 108 bytes | Selected record |
| Record accounting field | 12 bytes within a record | Read before admission and read again after delegation |
| Record-associated dependency object | 28 bytes within a record | Address passed to two surrounding dependencies |
| Shared accounting field | 144 bytes from the base | Read for admission; read and written after delegation |

The accounting fields are accessed as 32-bit integer quantities. The object at record offset 28 has no established type or extent.

For a valid selector, the caller’s environment must provide usable storage and dependency-compatible objects at these locations. There is no visible null check, allocation check, bounds check on the returned storage, or independent validation of its lifetime.

## Behavior

### 1. Obtain the base before validating the selector

F-6e90ba5205cc is called unconditionally, before the selector is tested.

Consequently, an out-of-range selector does not make the entire function side-effect-free: the base-provider dependency still runs. Its behavior is not otherwise specified here.

The returned base value is retained for the rest of the invocation. The function does not visibly reacquire it after later dependency calls.

### 2. Reject out-of-range selectors without record access

The selector comparison is unsigned. Only 0, 1, 2, and 3 enter the record-processing path.

For every other selector, the function returns 8 after the base-provider call. It does not visibly:

- Compute and use a selected record address.
- Invoke either dependency surrounding record processing.
- Invoke the record predicate.
- Invoke the delegated operation.
- Read or write either accounting field.

A bit pattern corresponding to a negative signed integer is still interpreted as a large unsigned selector and therefore rejected.

### 3. Enter the selected record’s guarded region

For an accepted selector, the record address is the base plus 36 times the selector.

The function calls F-338d16f9ce72 with the address 28 bytes into that record. It then calls F-96d4ea8519eb with the record address.

The return value of F-338d16f9ce72 is not visibly inspected. The structure resembles acquiring a per-record synchronization object, but neither synchronization nor successful acquisition is independently established by the supplied evidence.

If F-96d4ea8519eb returns zero, the function skips the admission calculation and delegated operation. It retains result 8 and proceeds to the closing dependency call.

Any nonzero predicate result admits the accounting check. The actual nonzero value is otherwise ignored.

### 4. Capture the prior record value and evaluate admission

After a nonzero predicate result, the function reads and saves the record’s accounting field at offset 12.

It then reads the shared accounting field at base offset 144 and combines it with the saved record value and requested adjustment. The admission quantity is:

**shared accounting value − saved record accounting value + requested adjustment**

The completed quantity is interpreted as an unsigned 32-bit value and compared with **102,401**, hexadecimal **0x19001**. Delegation occurs only when the unsigned value is strictly smaller.

Thus the accepted unsigned range is **0 through 102,400 inclusive**. Equality with 102,401 fails.

This is not an independent requirement that the requested adjustment be nonnegative or at most 102,400. A negative adjustment can pass if the complete quantity is in range; a positive adjustment can fail because of the other accounting values.

For ordinary calculations that do not overflow signed 32-bit intermediates, a negative completed quantity converts to a large unsigned value and fails. For example, −1 becomes 4,294,967,295 and is rejected.

The supplied representation uses signed arithmetic before the unsigned comparison. Exact behavior for signed-overflow intermediates is an uncertainty, not an established portability guarantee. A wrapping 32-bit interpretation is a plausible target-machine model, but must not be silently substituted for evidence of the actual instruction semantics.

### 5. Handle admission failure

If the unsigned comparison fails:

- The function selects result 25.
- F-80172b251073 is not called.
- This function performs no explicit accounting write.
- The closing dependency still runs before normal return.

Dependencies already invoked may have changed storage. “No explicit accounting write” does not mean that all accounting state necessarily remains unchanged.

### 6. Delegate the admitted operation

If admission succeeds, F-80172b251073 is called with:

1. The selected record address.
2. The unchanged opaque argument.
3. The unchanged requested adjustment.

Its 32-bit return value becomes the function’s selected result.

No result-code check gates the subsequent accounting update. A result that an external consumer interprets as failure still leads to the update if the dependency returns normally.

### 7. Update shared accounting using post-delegation state

After the delegated operation returns, the function reads:

- The record accounting field again.
- The shared accounting field again.

It writes the shared field with the value represented by:

**post-delegation record value − saved pre-delegation record value + post-delegation shared value**

The prior record value is the snapshot taken before admission. It is not refreshed immediately before delegation.

The shared value used for this write is freshly read after delegation. It is not necessarily the shared value used for admission. This matters because the delegated operation may itself modify the record field, the shared field, or other reachable state.

The function does not visibly write the record accounting field itself. Any change to that field during delegation is attributable to the dependency or other actors, not to an explicit write in this function.

Both post-delegation accounting reads precede the resulting shared-field write. Their relative evaluation order is not established by the supplied expression. The evidence also does not establish whether this update is atomic or protected against concurrent changes.

### 8. Close the selected record’s guarded region

For every accepted selector, all normally returning branches call F-1333331b2da1 with the same record-offset-28 address supplied to F-338d16f9ce72.

This includes:

- A zero predicate result.
- Admission failure.
- Delegation followed by accounting adjustment.

The return value of the closing dependency is not visibly inspected. The previously selected result is then returned.

There is no visible cleanup path for a dependency that does not return normally.

## Side effects

### Explicit memory effects

The function’s only visible direct write is to the shared accounting field, and only after an admitted delegated operation returns normally.

Direct accounting reads occur only after a nonzero predicate result:

- One record-field snapshot before admission.
- One shared-field read for admission.
- On delegation, fresh record and shared reads before the accounting write.

### Dependency-mediated effects

Every dependency may have effects not visible in this function. No dependency is assumed to preserve globals, record contents, allocation state, or referenced objects.

The retained base and selected record addresses are reused after dependency calls. Safe normal execution therefore requires that these addresses remain usable for subsequent accesses. The visible function provides no recovery if a dependency invalidates them.

### Ownership

No allocation, release, copying of an owned payload, or ownership transfer is explicitly established. Passing the opaque argument or record address to a dependency does not by itself prove that the dependency borrows, consumes, retains, or releases anything.

### Errors and exceptional control flow

Only the numeric results and branch conditions described above are established. No diagnostic output, exception mapping, retry, rollback, or compensating action is visible.

In particular, the function does not visibly undo dependency mutations when admission fails, delegation returns an error-like result, or the post-delegation accounting value differs from expectations.

## Dependencies

| Opaque ID | Invocation and use | Unknown contract |
|---|---|---|
| F-6e90ba5205cc | Called once before selector validation; supplies the retained base address | Storage provenance, initialization, lifetime, failure behavior, and side effects |
| F-338d16f9ce72 | Called for an accepted selector with the record-offset-28 address | Whether it acquires synchronization, can fail, or mutates state |
| F-96d4ea8519eb | Called with the selected record; zero skips admission, nonzero permits it | Meaning of the predicate, mutations, and external effects |
| F-80172b251073 | Called only after admission; receives the record and both forwarded operation arguments | Operation semantics, ownership, mutations, and meaning of its result |
| F-1333331b2da1 | Called after all normally completed valid-selector paths with the record-offset-28 address | Whether it releases synchronization, other effects, and abnormal behavior |

## Acceptance examples

These are conditional contract examples, not observed executions. Unless stated otherwise, assume valid storage, normally returning dependencies, no concurrent mutation, and arithmetic within signed 32-bit range.

1. **Selector outside the accepted range.**  
   With selector 4, F-6e90ba5205cc runs and the result is 8. None of the four record-processing dependencies runs.

2. **Predicate rejects the record.**  
   With selector 2 and a zero predicate result, the surrounding dependencies receive the address at base plus 100 bytes: 72 bytes for record selection plus 28 bytes for its associated object. The delegated operation is skipped and the result is 8.

3. **Admission at the upper accepted boundary.**  
   A shared value of 102,400, saved record value of 0, and requested adjustment of 0 produce 102,400. Delegation is admitted.

4. **Admission at the first rejected boundary.**  
   A shared value of 102,401, saved record value of 0, and requested adjustment of 0 produce 102,401. Delegation is skipped and the result is 25.

5. **Negative total rejected by unsigned comparison.**  
   A shared value of 0, saved record value of 1, and requested adjustment of 0 produce −1. Its unsigned conversion exceeds the limit, so the result is 25.

6. **Negative requested adjustment can pass.**  
   A shared value of 50, saved record value of 20, and requested adjustment of −10 produce 20. The requested adjustment’s sign alone does not prevent delegation.

7. **Accounting reflects record change, not the requested amount.**  
   Suppose admission uses shared value 100 and saved record value 40. After delegation, the record value is 47 and the shared field is still 100. The function writes 107 to the shared field, regardless of whether the requested adjustment was 7.

8. **Dependency changes to shared accounting are included.**  
   With the same saved record value of 40, suppose delegation leaves record value 47 and shared value 130. The function writes 137, not 107.

9. **A delegated error-like result does not suppress accounting.**  
   If the delegated operation returns 25 while changing the record field from 40 to 47 and leaving the shared field at 100, the function writes 107 and ultimately returns 25.

## Uncertainties

The following are **documented_only**:

- The semantic meaning and units of either accounting field.
- Whether 8 and 25 represent errors, states, or another status convention.
- The type, meaning, and ownership of the opaque argument.
- The actual locking semantics of the surrounding dependencies.
- Whether the predicate or delegated operation modifies shared storage.
- The validity and lifetime guarantees for the returned base.
- Exact signed-overflow behavior in admission and accounting arithmetic.
- Concurrency guarantees, memory ordering, atomicity, and reentrancy.
- Dependency failure behavior, including faults and nonlocal exits.
- The ordering of the two operand reads within the final accounting expression.

The resource-limit and synchronization interpretations are inferred intent. They are not additional acceptance requirements.

## Evidence

Established static evidence consists of the supplied function representation and its dependency-ID mapping. It supports the selector range, relative offsets, call sequence, branch conditions, 32-bit values, result sources, and explicit accounting write described above.

The representation does not include dependency implementations or a runtime trace. Semantic names such as “record accounting” and “shared accounting” describe their roles in this function, not independently verified domain meanings.

No runtime tests have been performed. The acceptance examples are derived consequences under their stated conditions, not test results.
