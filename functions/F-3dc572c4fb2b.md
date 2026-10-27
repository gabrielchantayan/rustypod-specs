<!-- Trusted provenance: F-3dc572c4fb2b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3dc572c4fb2b

## Purpose

F-3dc572c4fb2b conditionally records a 32-bit input in a caller-supplied object and performs a sequence of dependency calls. Its Boolean result reports whether the initial condition dependency returned nonzero. It does not independently report successful allocation, construction, dispatch, or ownership transfer.

The established sequence is:

- Read the object’s first 32-bit word, add 96, and submit the result to F-b7abcf60c990.
- If that dependency returns zero, return false without executing the accepted-path write or calls.
- Otherwise, write the input at object byte offset 24 and call F-0a77364c41cc with 12.
- After that call returns, read a value from a referenced global-data location and supply the value as the second argument to F-afb630a7107c.
- Read the object’s word at byte offset 4 after F-afb630a7107c returns, and pass it with that dependency’s result to F-2cb84dcba01a.
- Return true if the accepted sequence completes normally.

The global argument is a value read from global storage, not an established constant, descriptor, or fixed address. Earlier dependencies are not established to preserve that storage.

“Construction” and “dispatch,” where used below, describe inferred roles rather than known dependency contracts.

## Interface

### Inputs and object layout

The function receives an object reference and a 32-bit integer input.

| Input | Established use |
|---|---|
| Object reference | Supplies two 32-bit words, receives one 32-bit write, and is passed to F-afb630a7107c |
| Integer input | Initializes a local 32-bit cell; its value is written to the object on the accepted path; the cell’s address is passed to F-afb630a7107c |

The directly accessed object layout is:

| Byte offset | Width | Direct access |
|---|---:|---|
| 0 | 4 bytes | Read before F-b7abcf60c990 |
| 4 | 4 bytes | Read after F-afb630a7107c returns |
| 24 | 4 bytes | Written after a nonzero condition result, before F-0a77364c41cc |

The function performs no explicit validation of nullness, alignment, extent, or permissions. A conventional sufficient precondition for its direct object accesses is suitably aligned storage extending through byte offset 27, with readable words at offsets 0 and 4 and writable storage at offset 24. Required accesses must remain valid when they occur. Dependencies may impose additional requirements or change the validity of later accesses.

The global-data location must also support the accepted-path read when it occurs. Its declaration, exact storage width, and relationship to other storage are not supplied.

### Output

The declared result is Boolean:

- False when F-b7abcf60c990 returns zero.
- True when F-b7abcf60c990 returns any nonzero value and the accepted sequence reaches the normal return.

Negative condition results are accepted. The exact nonzero result is not exposed through the Boolean interface. No later dependency result is checked to determine this Boolean outcome.

The physical calling-convention representation of the Boolean is not established.

### Widths and arithmetic

The object words, integer input, local input cell, and condition result are represented as 32-bit integers. The results forwarded from F-0a77364c41cc and F-afb630a7107c are opaque four-byte quantities. Their width alone does not establish pointer, resource, or status semantics.

The initial arithmetic adds hexadecimal 60, decimal 96, to the value read at object offset 0. The evidence does not establish that this is a byte-pointer adjustment.

For a signed 32-bit operand between −2,147,483,648 and 2,147,483,551 inclusive, the mathematical sum is representable as a signed 32-bit integer. Behavior above that upper bound is documented_only: the supplied representation does not justify choosing wraparound, trapping, or another overflow policy.

The global-data declaration is absent. Although its value is supplied as one argument, neither its exact read width nor any conversion at that argument boundary is independently established.

## Behavior

### Preparation and condition evaluation

The function retains the original object reference and initializes a local four-byte cell from the integer input.

It reads the 32-bit word at object offset 0, adds 96, and calls F-b7abcf60c990 with the resulting operand. The returned integer is retained as the decision value.

There is one condition call. The final Boolean result depends on this retained decision value, not on a later reread of the object or another condition evaluation.

### Rejected path

When F-b7abcf60c990 returns zero:

- There is no explicit write to object offset 24.
- F-0a77364c41cc is not called.
- The global-data read used to form the second argument of F-afb630a7107c is not performed by this function.
- F-afb630a7107c and F-2cb84dcba01a are not called.
- The function returns false.

This does not guarantee unchanged external state. The initial dependency has already executed and may have side effects, including changes to global storage or the object through state available to it.

### Accepted path

When F-b7abcf60c990 returns nonzero, the following sequence is established, assuming normal completion of each operation.

First, the function writes the local input cell’s value to object offset 24 as a four-byte word. The cell was initialized from the input, and its address has not yet been passed to a dependency.

Second, it calls F-0a77364c41cc with the integer value 12. Its opaque four-byte result is retained. The argument resembles a size, but allocation semantics are not established.

Third, after F-0a77364c41cc returns, the function reads the referenced global-data location. The value obtained by this read supplies the second argument to F-afb630a7107c.

This is a call-time read, not an entry-time snapshot. If the initial dependency or F-0a77364c41cc changes that global, the subsequently read value must reflect the state at the read, subject to any unresolved concurrency or memory-model constraints. The function makes no preservation assumption about either dependency.

Fourth, F-afb630a7107c is called with these arguments in logical argument order:

1. The opaque result from F-0a77364c41cc.
2. The value read from the global-data location after F-0a77364c41cc returned.
3. The address of the local four-byte input cell.
4. The integer value 4.
5. The original object reference.

Logical argument order does not establish the evaluation order of independent argument expressions beyond the sequencing described above.

The local-cell address is not the address of object offset 24. A dependency can therefore alter the cell without that alteration itself constituting a write to the object field.

F-afb630a7107c returns another opaque four-byte value. That result, not necessarily its first argument, becomes the second argument of the final call.

Fifth, after F-afb630a7107c returns, the function reads the object’s 32-bit word at offset 4. It calls F-2cb84dcba01a with that newly read word as the first argument and the preceding opaque result as the second argument. No result from this final call is used.

Finally, the function returns true from the retained nonzero decision value.

### Observable ordering

The established sequence includes these boundaries:

1. Offset-0 read before the condition call.
2. Nonzero condition result before the offset-24 write.
3. Offset-24 write before F-0a77364c41cc.
4. Return from F-0a77364c41cc before the global-data read.
5. Global-data read before invocation of F-afb630a7107c.
6. Return from F-afb630a7107c before the offset-4 read.
7. Offset-4 read before F-2cb84dcba01a.
8. Normal return from the final dependency before the function’s normal Boolean return.

An entry-time snapshot of either the global argument or the offset-4 word would not preserve this behavior when intervening calls change the relevant storage.

The explicit write to offset 24 is not a final-state guarantee. Dependencies may subsequently change that field. The function does not restore it or copy the local cell back after F-afb630a7107c.

## Side effects

### Direct reads and writes

The function directly reads object offset 0 on every execution that reaches condition evaluation. On the accepted path, it also reads the referenced global-data location and object offset 4.

Its only explicit object write is the four-byte write at offset 24 on the accepted path. It also initializes an invocation-local four-byte cell and exposes that cell’s address to F-afb630a7107c.

There is no explicit assignment to the referenced global-data location by name. However, its relationship to object storage and other reachable memory is unknown; this is not a general non-aliasing or global-preservation guarantee.

### Dependency effects and ownership

No dependency is established to preserve globals, object contents, external resources, or other reachable memory.

F-afb630a7107c receives both the original object reference and the local-cell address. Whether it reads, mutates, copies, or retains either is documented_only.

No ownership transfer is established for either opaque intermediate result or for the global value. The function contains no explicit release operation. This alone establishes neither a leak nor successful transfer of release responsibility.

The local cell has invocation-local lifetime. Passing its address does not prove that the recipient copies its contents before return. Any retention requirements or lifetime compatibility remain unresolved.

### Errors and partial progress

There are no explicit checks for invalid storage, failure-like intermediate values, dependency-reported failure, or arithmetic overflow.

A normally returned zero from F-0a77364c41cc is forwarded to F-afb630a7107c. A normally returned zero from F-afb630a7107c is forwarded to F-2cb84dcba01a. Neither causes a local false return.

The function provides no explicit rollback. If a later operation faults or fails to return normally, the earlier offset-24 write has already occurred. Exception propagation, fault recovery, and nonlocal control flow are not established. A false result must not be substituted for such behavior without additional evidence.

## Dependencies

| Opaque function ID | Invocation and arguments | Established result use |
|---|---|---|
| F-b7abcf60c990 | After the offset-0 read and addition; receives that word plus 96 | Integer result tested against zero and retained for the final Boolean |
| F-0a77364c41cc | Accepted path, after the offset-24 write; receives 12 | Opaque four-byte result becomes the first argument of F-afb630a7107c |
| F-afb630a7107c | After F-0a77364c41cc returns and the global is read; receives the preceding result, the newly read global value, the local-cell address, 4, and the original object reference | Opaque four-byte result becomes the second argument of F-2cb84dcba01a |
| F-2cb84dcba01a | After F-afb630a7107c and the offset-4 read; receives that word and the preceding result | No returned value is used |

The referenced global-data location is an additional data dependency. Its current contents are read only on the accepted path, after F-0a77364c41cc returns. It is not established to contain a descriptor, an address, or an immutable value.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests. Assume valid required accesses, representable arithmetic, and normal dependency returns unless stated otherwise.

### Zero condition result

If F-b7abcf60c990 returns zero, the result is false. None of the other three dependencies is called, and this function performs neither its offset-24 write nor its accepted-path global-data read.

Effects produced by the condition dependency remain possible.

### Negative condition result

If the condition result is −1, the accepted sequence executes and returns true. Accepting only positive results would violate the zero-versus-nonzero decision.

### Ordinary accepted operation

Suppose offset 0 contains 100, the input is −7, and offset 4 remains 42 until its read.

The condition dependency receives 196. If it returns nonzero, the function writes −7 at offset 24 before calling F-0a77364c41cc with 12.

F-afb630a7107c receives the allocator-like result, the value subsequently read from global storage, a local-cell address initially containing −7, 4, and the original object reference. The final dependency receives 42 and the result of F-afb630a7107c.

### Initial dependency changes the global

Suppose the global initially contains one value, and F-b7abcf60c990 replaces it with another while returning nonzero. If no intervening operation changes the global again, F-afb630a7107c must receive the replacement value.

Using the entry-time value would violate the established read timing.

### F-0a77364c41cc changes the global

Suppose F-0a77364c41cc changes the global before returning. If the global is not changed again before the subsequent read, its changed value must become the second argument of F-afb630a7107c.

If both earlier dependencies changed the global, the required argument is determined by the later read, not by either earlier snapshot.

### Construction changes offset 4

If F-afb630a7107c changes offset 4 from 42 to 99 and that value remains until the subsequent read, F-2cb84dcba01a receives 99 as its first argument.

### Construction changes the local input cell

If the input is 5 and F-afb630a7107c changes the addressed cell to 8 without otherwise changing offset 24, the function’s earlier field write remains 5. There is no subsequent copy of 8 into the object.

### Zero opaque results

If F-0a77364c41cc returns zero, that zero is still passed to F-afb630a7107c. If F-afb630a7107c returns zero normally, that zero is still passed to F-2cb84dcba01a. Their validity under the dependency contracts is unknown.

### Interrupted accepted path

If the condition is nonzero, the offset-24 write succeeds, and a later dependency does not return normally, no normal Boolean result is established. The function has no explicit operation undoing the write.

## Uncertainties

The following remain documented_only:

- Semantic meanings of the object fields.
- The intended interpretation of the initial addition and its overflow behavior.
- The global-data location’s exact declared width, type, contents, mutability rules, and aliasing relationships.
- Whether the global value represents an address, descriptor, handle, scalar, or something else.
- Any dependency guarantee to preserve or modify global storage.
- Whether 12 denotes an allocation size or 4 denotes a payload length.
- Whether the local cell is copied, modified, or retained.
- Ownership and release obligations for intermediate values.
- Dependency error conventions, nonlocal control flow, and external effects.
- Synchronization, concurrent modification, atomicity, and observer guarantees.

The inferred intent is to gate a value-bearing operation and route a prepared result through an object-associated destination. That interpretation adds no acceptance requirements.

## Evidence

Established static evidence consists of the supplied function body and opaque dependency mapping.

It supports the object access offsets and widths, local-cell address flow, zero-versus-nonzero branch, dependency call order, explicit argument relationships, and absence of local result validation or cleanup.

In particular, the second argument to F-afb630a7107c is obtained by reading referenced global storage after F-0a77364c41cc returns. The body does not establish an immutable operand or a fixed descriptor address. No supplied dependency body establishes preservation of that global, so changes made by either earlier dependency cannot be assumed away.

The evidence also establishes the delayed offset-4 read and forwarding of the result returned by F-afb630a7107c. It does not establish dependency internals, resource ownership, global-data semantics, or runtime success.

No runtime tests have been performed. All acceptance examples are conditional consequences of static evidence.
