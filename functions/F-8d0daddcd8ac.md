<!-- Trusted provenance: F-8d0daddcd8ac; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8d0daddcd8ac

## Purpose

F-8d0daddcd8ac conditionally clears an association stored in a caller-supplied record. The successful path requires both a matching discriminator in the associated record and a matching 32-bit value supplied by the caller.

On that path, the function clears one byte in the caller-supplied record, invokes F-4b90c515829a, clears the association field, and returns the 32-bit value 1. Otherwise, it returns the 32-bit value 0 without directly writing memory or invoking that dependency.

This is a contract for the behavior visible in the supplied static evidence. The meanings of the record types, discriminator, cleared byte, and dependency are not established. In particular, “association” describes the stored reference; it does not imply ownership or resource-release semantics.

## Interface

The function accepts two inputs and produces one 32-bit result.

| Item | Interpretation established by the supplied evidence |
|---|---|
| First input | An address-like value identifying a writable base record. |
| Second input | A 32-bit value compared for exact equality with a field in an associated record. |
| Result | 32-bit 0 for a nonmatching path; 32-bit 1 after the successful path completes normally. |

The relevant layout is:

| Location | Width | Use |
|---|---:|---|
| Base record, byte offset +12 | 32 bits | Initially read as the associated-record address. A zero value means no associated record. Cleared after the dependency returns on the successful path. |
| Base record, byte offset +16 | 8 bits | Written with zero before invoking the dependency on the successful path. Never read by this function. |
| Associated record, byte offset +4 | 32 bits | Discriminator compared with a global comparison value. |
| Associated record, byte offset +16 | 32 bits | Value compared with the second input, but only after the discriminator matches. |
| Global comparison location | 32 bits | Supplies the discriminator value required for success. Its contents and meaning are not supplied. |

The associated-record address is obtained from the base record. No associated-record address is passed separately.

The comparisons are equality comparisons, not ordering comparisons. No signed range, positive-value restriction, or special interpretation of the second input is established. A zero second input is permitted by the visible logic and can match a zero associated-record value.

### Preconditions

For normal completion, each memory location actually accessed must support the corresponding access. The function performs no explicit validation.

At minimum:

- The base record’s 32-bit field at offset +12 must be readable.
- If that field is nonzero, the associated record’s 32-bit field at offset +4 and the global comparison location must be readable.
- If the discriminator matches, the associated record’s 32-bit field at offset +16 must also be readable.
- If both comparisons match, the base record must support an 8-bit write at offset +16 and a later 32-bit write at offset +12.
- F-4b90c515829a must complete normally, and the base record must remain usable for the final write, for this function to return 1 normally.

The evidence does not establish alignment requirements, address-space bounds, or behavior for invalid addresses. It does not show a null check on the first input.

## Behavior

### Association lookup

The function first reads the 32-bit field at base-record offset +12.

If the value read is zero, the function returns 0. On this path, it does not access an associated record, does not read the global comparison value, does not invoke F-4b90c515829a, and performs no direct writes.

If the value is nonzero, that captured value supplies the address used for the associated-record accesses. The function does not visibly reload the base record’s association field before making those accesses.

### Discriminator gate

For a nonzero associated-record address, the function reads the 32-bit field at associated-record offset +4 and compares it with the global comparison value.

If they differ, the function returns 0. It does not read the associated-record field at offset +16 on this path. It also performs no direct writes and does not invoke F-4b90c515829a.

The discriminator is compared with the global value available during this comparison. No fixed discriminator constant can be recovered from the supplied material.

### Caller-value gate

Only if the discriminator comparison succeeds does the function read the 32-bit field at associated-record offset +16.

That value is compared with the second input. If they differ, the function returns 0 without directly writing memory or invoking F-4b90c515829a.

There is no additional condition on the prior contents of the base-record byte at offset +16. That byte is not inspected before success is selected.

### Successful transition

When both comparisons match, the function performs these operations in order:

1. Writes zero to the single byte at base-record offset +16.
2. Invokes F-4b90c515829a with no explicit arguments shown.
3. After that invocation returns normally, writes a 32-bit zero to the field at base-record offset +12.
4. Returns the 32-bit value 1.

The order of the two writes relative to the dependency invocation is part of the observable behavior. The association field is not cleared by this function before the invocation.

The initial byte write is not a wider field reset. It directly affects only one byte. The later association-field write affects the four-byte region beginning at offset +12.

The function does not visibly repeat either comparison after invoking the dependency. Changes made during that invocation do not trigger a fresh eligibility check. If the dependency returns normally and the final write remains valid, the function proceeds to clear the association field and return 1.

### Arithmetic and width boundaries

The visible address calculations use fixed byte offsets: +4, +12, and +16. There is no multiplication, indexing loop, length calculation, or arithmetic involving the second input.

The supplied representation uses 32-bit quantities for the association, discriminator, comparison value, caller-supplied match value, and result. The offset +16 write to the base record is explicitly only 8 bits.

No contract is established for address calculations that overflow the applicable address domain or refer outside valid storage. Such cases must not be treated as deliberately supported wraparound behavior.

## Side effects

On every normally completing path that returns 0, the function performs no explicit memory writes and makes no explicit dependency invocation. These paths still perform the conditional reads described above.

On the normally completing path that returns 1, the direct writes are:

- An 8-bit zero at base-record offset +16, before the dependency invocation.
- A 32-bit zero at base-record offset +12, after the dependency invocation.

There are no other explicit writes in the supplied body. That does not establish an absence of other effects: F-4b90c515829a may access or modify memory or external state.

### Intermediate state

Immediately before invoking F-4b90c515829a, this function has cleared the byte at base-record offset +16 but has not yet cleared the association field at offset +12.

For ordinary nonoverlapping storage, and without intervening mutation, the association field therefore still contains its earlier nonzero value at dependency entry. This intermediate state matters to any dependency behavior that observes the record.

No stronger invariant is justified. Aliasing, concurrent mutation, or other execution-context effects could change what the dependency observes.

### Ownership and lifetime

The function does not visibly allocate memory. It does not directly perform an identifiable memory-release operation.

Clearing the association field removes the stored reference from that field. It does not, by itself, establish that the associated record was freed, transferred, retained elsewhere, or made invalid.

F-4b90c515829a’s ownership behavior is unknown. In particular, there is no evidence that it preserves the base record’s lifetime. Normal completion of this function requires that the final base-record write remain valid after the dependency returns.

### Abnormal completion

No explicit error handler, rollback, or recovery path is shown.

If the dependency does not return normally, the final association clear and return of 1 are not established. The earlier byte clear has already occurred before invocation; no undo operation is visible.

Invalid accesses are not translated into a defined return value by the visible body.

## Dependencies

### F-4b90c515829a

This is the only explicit function dependency.

Its invocation contract, as far as this function establishes, is:

- It is invoked only after a nonzero association and two successful equality comparisons.
- It is invoked once on that successful path.
- No explicit arguments are shown.
- It is invoked after clearing the base-record byte at offset +16 and before clearing the base-record association field at offset +12.
- No return value is used by this function.
- Its normal return permits the final association clear and the result 1.

The dependency’s own interface, hidden inputs, global accesses, memory effects, ownership rules, reentrancy, and failure behavior are documented_only unknowns. The absence of explicit arguments does not establish that it has no inputs or cannot identify the affected record.

### Global comparison value

The function also depends on one 32-bit global comparison value. It is consulted only when the association field is nonzero.

The global’s initialization, mutability, lifetime, and domain meaning are unknown. The function performs no explicit write to it. F-4b90c515829a must not be assumed to preserve it.

## Acceptance examples

These examples are conditional acceptance criteria derived from static evidence. No runtime tests have been performed. They assume valid required accesses and no unmentioned mutation.

1. **No association.**  
   Given a zero value at base-record offset +12, the expected result is 0. No associated-record field or global comparison value is read, no dependency is invoked, and no direct write occurs.

2. **Discriminator mismatch.**  
   Given a nonzero association whose offset +4 value differs from the global comparison value, the expected result is 0. The associated-record field at offset +16 must not be read by this function. The base record is not directly modified.

3. **Matching discriminator, mismatching caller value.**  
   Given a matching discriminator and an associated-record offset +16 value unequal to the second input, the expected result is 0. F-4b90c515829a is not invoked, and neither base-record field is directly written.

4. **Full match with normal dependency return.**  
   Given both matches and a dependency invocation that returns normally without invalidating the base record, the byte at base-record offset +16 is cleared before invocation. The association field is cleared after invocation. The expected result is 1.

5. **Already-zero byte.**  
   Given both matches and an initially zero byte at base-record offset +16, the successful path still writes that byte, invokes F-4b90c515829a, clears the association field, and returns 1 if completion is normal. The byte’s initial value is not an eligibility condition.

6. **Zero match value.**  
   Given a nonzero association, a matching discriminator, and zero in the associated-record field at offset +16, a zero second input satisfies the caller-value gate.

7. **Dependency changes the association field.**  
   Given both matches, if the dependency writes a different value to base-record offset +12 and then returns normally while the record remains valid, this function subsequently overwrites that field with zero. It does not preserve or revalidate the dependency’s replacement.

8. **Dependency does not return.**  
   Given both matches and a dependency invocation that does not return normally, the pre-invocation byte clear has occurred. This function’s final association clear and result 1 are not guaranteed.

## Uncertainties

The following are documented_only rather than inferred requirements:

- The semantic type of either record.
- Whether the associated-record discriminator denotes a type, owner, category, or other property.
- The meaning of the second input beyond exact 32-bit equality.
- Whether the cleared byte is a flag, status, or part of a larger representation.
- The actual value and lifecycle of the global comparison value.
- The complete calling convention and any implicit inputs to the dependency.
- Allocation, release, and ownership effects of the dependency.
- Permitted aliasing among the records and global storage.
- Thread-safety, atomicity, memory ordering, and reentrant behavior.
- Alignment rules, address-overflow behavior, and fault handling.
- A broader record-size requirement beyond the accesses exercised on each path.

An inferred intent is that the function conditionally detaches or clears an association. That interpretation is consistent with its writes but is not necessary to the established behavioral contract.

## Evidence

The supplied static body establishes the conditional reads, two equality gates, 8-bit pre-invocation write, single dependency invocation, 32-bit post-invocation write, and explicit results 0 and 1.

It does not establish the dependency’s implementation, object ownership, global preservation, synchronization, or caller invariants. Those properties have not been assumed.

No runtime tests, execution traces, or independent implementation inspection have been performed. The acceptance examples describe consequences of the supplied static evidence, not observed executions.
