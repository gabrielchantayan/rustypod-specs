<!-- Trusted provenance: F-9eefbcf5756f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9eefbcf5756f

## Purpose

F-9eefbcf5756f conditionally processes a sequence of nonzero 32-bit values against a bitmap-like descriptor. For each value whose membership dependency returns zero, it increments a count-like field and sets a selected bitmap bit. It then performs a branch-completion sequence. On every normally completed path, it calls F-c452dd64d509 with the original first two arguments.

The bitmap interpretation is inferred from the membership query, count increment, and bitwise insertion. The domain meanings of the objects, global operands, sequence values, and completion calls are not established.

This contract distinguishes visible operations from inferred intent and unresolved dependency behavior. It is based on static evidence only; no runtime tests have been performed.

## Interface

The function accepts four arguments and has no explicit return value.

| Argument | Visible representation | Use |
|---|---|---|
| First | Signed 32-bit integer | Initializes the first working word. The original argument is passed to F-c452dd64d509. |
| Second | Opaque 32-bit value | Initializes the second working word, which is later cleared and passed by address to an indirect dependency. The original argument is passed to F-c452dd64d509. |
| Third | Unsigned 32-bit value | Initializes the third working word. That word is exposed to F-78beca4e4455, then masked to obtain an object address. |
| Fourth | Opaque 32-bit value | Initializes the fourth working word. No subsequent explicit read by this function is shown. |

The working storage is represented as four adjacent four-byte slots in argument order. F-78beca4e4455 receives the address of the third slot. Its access extent is unknown; the fourth slot must not be treated as irrelevant merely because this function does not explicitly read it again.

The original first and second arguments, rather than their subsequently reused working words, supply the final direct call.

### Referenced global operands

Four operands are read from global storage. They are not established constants.

| Contract label | Use |
|---|---|
| G1 | Second argument to F-5710a8628cba |
| G2 | Third argument to F-5710a8628cba |
| G3 | Additional argument to the indirect dependency at table byte offset 248 |
| G4 | Second argument to F-85722f62e20a |

These labels identify the four referenced global operands, not their values or domain types. Their declarations, contents, mutability rules, and initialization requirements are not supplied.

G1 and G2 are read after F-eea3e31f6354 returns. G3 is read after F-938c950e5046 returns. G4 is read after F-3ecf06e452fb returns. No relative evaluation order between the G1 and G2 reads is established.

### Preconditions for normal completion

Every address actually accessed must support the corresponding read, write, or indirect call. This includes:

- The object returned by F-938c950e5046 and its method table.
- The object selected from the third working word.
- The lookup address when the processing branch is entered.
- The bitmap descriptor and its accessed fields.
- The selected bitmap storage words.
- All method-table entries used by the function.

There are no visible checks for bitmap capacity, sequence bounds, object alignment, or valid indirect-call targets. Address validity alone does not imply that the accessed regions are disjoint.

## Behavior

### Setup and global-read timing

The four working words are initialized from the arguments. The following sequence then occurs:

1. F-eea3e31f6354 is called with no visible arguments.
2. After it returns, G1 and G2 are read while preparing F-5710a8628cba. That dependency receives the preceding result, G1, and G2, in that argument order. Its result is retained as a lookup address.
3. F-938c950e5046 is called with no visible arguments.
4. After it returns, G3 is read while preparing an indirect invocation on the returned object. The invocation uses the method-table entry at byte offset 248 and receives the object and the value read from G3.
5. F-682adfd5a561 receives that indirect invocation’s result and the integer 5504.
6. F-78beca4e4455 receives the address of the third working word and an address 164 bytes beyond the result of F-682adfd5a561.

The return value of F-78beca4e4455 is not consumed. Its possible modification of the third working word affects subsequent object selection.

The global arguments are obtained at their respective call-preparation points, not captured together at function entry. In particular, changes made by F-eea3e31f6354 can affect G1 and G2, and changes made by F-938c950e5046 can affect G3. Preservation by these or other dependencies is not established.

The relative order of the two global reads for F-5710a8628cba is unspecified by the supplied representation. Argument position does not establish read order.

### Object selection and processing gate

After F-78beca4e4455 returns, bit 0 of the third working word is cleared with the 32-bit mask 0xFFFFFFFE. The resulting value becomes the selected object address.

This masking does not validate the address. A working value of either zero or one selects address zero, without a null check before the following indirect invocation.

The selected object’s method at table byte offset 20 is invoked with the object as receiver. Its result is tested for zero versus nonzero.

The processing branch requires both:

- A nonzero result from that method.
- A nonzero retained lookup address.

The method is invoked before the lookup-address condition is tested. A null lookup result therefore does not avoid object selection, the offset-20 invocation, or preceding setup.

If either condition fails, the function skips all processing-branch operations and proceeds to the final direct call.

### Processing-branch initialization

When both conditions hold:

1. F-8f4035bf2a29 is called with 33.
2. The selected object’s method at table byte offset 40 is invoked with the object as receiver.
3. Its result is retained as the bitmap descriptor address.
4. The first 32-bit word at the lookup address is read into the first working word.

The lookup word is read after both preceding calls. It is not a snapshot captured when the lookup address was obtained. Changes to the pointed-to storage by intervening dependencies can affect this read.

If the word read is zero, no sequence-advance call occurs. The branch nevertheless proceeds to its completion sequence.

### Sequence advancement

For an initially nonzero first working word, F-d3922c7fe0ec is repeatedly called with that word’s address. Each result is interpreted as an unsigned 32-bit value.

A zero result terminates iteration immediately. No membership query or bitmap insertion is performed for that result.

A nonzero result selects:

- A bitmap word index equal to the value shifted right by five bits.
- A bit position equal to the value’s low five bits.

F-88658862e573 receives the descriptor address, word index, and bit position.

After the initial nonzero check, the caller does not independently use the working word’s current contents as a continuation condition. Continuation depends on successive return values from F-d3922c7fe0ec. Changes to the working word are permitted by its address-based interface, but its encoding and progression rules are unknown.

A dependency that continues returning nonzero values can prevent termination.

### Conditional count and bitmap updates

A nonzero result from F-88658862e573 suppresses the caller’s explicit count and bitmap writes for that sequence value. This does not establish that the dependency itself made no changes.

A zero result causes these operations, in order:

1. Read the 32-bit count-like field at descriptor byte offset 4, add one, and write the result to that field.
2. Use the storage-address field at descriptor byte offset 8 to locate the selected 32-bit bitmap word, read that word, OR in the selected bit, and write the resulting word.

The selected bitmap word is at the storage address plus four times the word index.

The bitmap access follows the count update. The storage-address field is used after the membership call and after the explicit count update; its value is not established as an earlier snapshot.

These operations are not an atomic transaction. No caller-side rollback is visible if the count update succeeds but a later access fails.

The caller does not verify that a zero membership result corresponds to a clear bit. If the dependency returns zero for a bit already set, the count still increases, while the OR may leave the bitmap word unchanged.

### Aliasing and layout consequences

The visible descriptor accesses establish only:

| Byte offset | Visible use |
|---|---|
| 4 | Read and written as a 32-bit signed count-like field |
| 8 | Read as a 32-bit storage-address field |

The representation does not establish disjointness between bitmap words, descriptor metadata, object storage, lookup storage, globals, or other accessible memory.

Consequently, a bitmap write can change the count field, the storage-address field, or other state used later. Such changes can affect later iterations, method dispatch, or completion-call arguments. Valid addresses and otherwise side-effect-free dependencies do not exclude these effects.

A simple “count increases once per new bit” interpretation therefore additionally requires appropriate non-aliasing and membership invariants.

### Branch completion

After iteration returns zero, or immediately after a zero initial sequence word, the function:

1. Writes zero to the second working word.
2. Invokes the selected object’s method at table byte offset 28 with the object, the integer 2, and the address of that working word.
3. Calls F-3ecf06e452fb with 2.
4. After that call returns, reads G4 and calls F-85722f62e20a with 33 and the value read.

G4 is not an entry-time constant or snapshot. Any preceding operation that changes its storage can affect the value passed, including F-3ecf06e452fb itself.

Any value written into the second working word by the offset-28 method is not subsequently consumed by the visible function.

Each indirect invocation obtains its target through the receiver’s method table at that invocation. Preservation of that table by earlier dependencies is not assumed.

### Final direct call

On every normally completed path, F-c452dd64d509 receives the original first and second arguments. The function then returns without an explicit result.

This includes paths with a false processing predicate, a null lookup address, an initially zero sequence word, an immediate zero sequence result, or exclusively nonzero membership results.

This is not an exception-safe finalization guarantee. A fault, nonreturning dependency, or nonlocal control transfer can prevent the final call.

## Side effects

### Explicit writes and exposed storage

The visible caller writes:

- All four working words during initialization.
- The first working word when loading the lookup word.
- The second working word before the offset-28 invocation.
- The descriptor’s count-like field for each zero membership result.
- The selected bitmap word for each zero membership result.

F-78beca4e4455, F-d3922c7fe0ec, and the offset-28 dependency receive addresses into working storage. Their permitted access extent and ownership behavior are not supplied.

The function explicitly reads the four global operands at the points described above. It contains no direct assignment targeting those global operands. This does not guarantee that their storage remains unchanged: dependencies or overlapping writes can affect it.

### Widths and arithmetic

The sequence value and bitmap word are 32 bits wide. For any nonzero sequence result:

- The word index ranges from 0 through 134,217,727.
- The bit position ranges from 0 through 31.
- The relative bitmap-word byte offset ranges up to 536,870,908.

These are selection ranges, not storage-capacity guarantees. There is no visible bounds check.

Selecting bit 31 involves the high bit of a 32-bit word. The supplied shifted-literal expression does not by itself establish portable language-level behavior at that signed boundary.

The count increment is represented as signed 32-bit arithmetic. No overflow check or saturation is visible. Behavior at the signed limit is documented_only; neither portable overflow behavior nor guaranteed wrapping is asserted.

Address additions are unchecked. Their validity, including the setup offset of 164 bytes, is a precondition rather than an enforced outcome.

### Ownership and errors

No explicit allocation, deallocation, ownership transfer, status return, or recovery mechanism is shown. Dependencies may allocate, release, retain, or modify storage; their contracts are unavailable.

The calls involving 33 and 2 do not establish a lock, transaction, notification protocol, or ownership discipline. No synchronization or concurrency guarantee follows from their placement.

## Dependencies

| Dependency | Established use |
|---|---|
| F-eea3e31f6354 | Supplies the first argument to F-5710a8628cba; completes before G1 and G2 are read. |
| F-5710a8628cba | Receives that result and the values read from G1 and G2; returns the retained lookup address. |
| F-938c950e5046 | Returns the receiver for the offset-248 invocation; completes before G3 is read. |
| F-682adfd5a561 | Receives the offset-248 result and 5504; supplies the base for the address displaced by 164 bytes. |
| F-78beca4e4455 | Receives the third working word’s address and the derived address. |
| F-8f4035bf2a29 | Receives 33 at processing-branch entry. |
| F-d3922c7fe0ec | Receives sequence-state storage and returns values; zero ends iteration. |
| F-88658862e573 | Returns the predicate controlling explicit count and bitmap updates. |
| F-3ecf06e452fb | Receives 2 during branch completion; completes before G4 is read. |
| F-85722f62e20a | Receives 33 and the subsequently read G4 value. |
| F-c452dd64d509 | Receives the original first two arguments on normal completion. |

Indirect dependencies occur at method-table byte offsets 248, 20, 40, and 28. Their opaque function IDs are unavailable; their offsets identify invocation roles, not stable implementations.

## Acceptance examples

These are conditional specifications, not executed tests. Each assumes valid accesses, returning dependencies, defined arithmetic for the stated values, and no additional effects except those explicitly described.

### Processing gate fails

If the offset-20 method returns zero, setup and that invocation still occur. The offset-40 method, sequence processing, and branch-completion sequence do not occur. F-c452dd64d509 receives the original first two arguments.

If instead the method returns nonzero but the lookup address is null, the same branch operations are skipped. The null lookup address is not dereferenced.

### Empty initial state

If both branch conditions hold and the lookup word is zero when read, F-8f4035bf2a29 and the offset-40 invocation have already occurred. F-d3922c7fe0ec is not called. The caller performs no count or bitmap update, but does perform branch completion.

### Global values change during setup

Suppose F-eea3e31f6354 changes G1 and G2 and no later change occurs before their reads. F-5710a8628cba receives the changed values in its second and third argument positions. The contract does not require G1 to be read before G2.

Similarly, if F-938c950e5046 changes G3, the following offset-248 invocation receives the subsequently read value, not an assumed entry-time value.

### Completion global changes

If processing reaches completion and F-3ecf06e452fb changes G4, with no subsequent change before its read, F-85722f62e20a receives that changed value as its second argument.

### Word-boundary insertion with non-aliasing

Suppose iteration returns 31, then 32, then zero, and both membership queries return zero. Assume:

- The count initially equals 7.
- The selected bits are initially clear.
- The accessed bitmap words do not overlap descriptor metadata or other storage whose modification would affect this example.
- No dependency changes the relevant count, storage address, or bitmap words.

The first value selects word 0, bit 31; the second selects word 1, bit 0. Under these assumptions and the stated bit-operation interpretation, the explicit updates leave the count at 9, set both selected bits, and preserve the other bitmap bits.

Without the non-aliasing assumption, those final-state assertions do not follow merely from valid addresses.

### Membership and repetition

For value 65, a nonzero membership result suppresses the caller’s count and bitmap writes for word 2, bit 1.

If 65 occurs twice and membership returns zero and then nonzero, the caller performs one count increment. If membership returns zero twice, it performs two increments. Deduplication depends on the dependency and storage invariants.

### Zero result

A zero return from F-d3922c7fe0ec terminates iteration without a membership query. Bitmap position zero is not inserted through this sequence path.

## Uncertainties

- **documented_only — Global definitions:** Global operand widths, types, initialization, mutability, and synchronization are not supplied.
- **documented_only — Dependency effects:** No preservation of globals, object tables, descriptor fields, storage, or ownership is assumed.
- **documented_only — Working-storage extent:** Address-taking dependencies may access more than the directly addressed word; their extent is unknown.
- **documented_only — Sequence representation:** Encoding, progression rules, and termination guarantees are unavailable.
- **documented_only — Capacity and aliasing:** No storage bounds, disjointness, or allocation layout is established.
- **documented_only — Arithmetic boundaries:** Signed count overflow, high-bit shift portability, and invalid address arithmetic remain unresolved.
- **documented_only — Exceptional execution:** Fault handling, nonlocal exits, reentrancy, and concurrent access behavior are not shown.
- **Inferred intent:** The descriptor appears to track bitmap membership and a corresponding count. Accuracy requires additional dependency and layout invariants.
- **Inferred intent:** The completion calls may participate in an external protocol, but its meaning is unknown.

## Evidence

Established behavior comes from the supplied static representation: argument widths, working-storage layout, global operand reads, call boundaries, indirect-method offsets, predicates, memory accesses, and arithmetic expressions.

The representation establishes the timing of each global read relative to the preceding completed call, but not an evaluation order between G1 and G2. It also shows ordered count and bitmap updates without establishing non-aliasing.

Dependency bodies, global declarations, allocation contracts, and runtime observations were not supplied. No runtime tests have been performed. The acceptance examples express conditional consequences of the visible operations, not observed executions.
