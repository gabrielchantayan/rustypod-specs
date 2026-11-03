<!-- Trusted provenance: F-dba4ee3fee8d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dba4ee3fee8d

## Purpose

F-dba4ee3fee8d examines a non-null byte sequence through two cursor-oriented dependencies, counts a qualifying suffix, and conditionally requests an update to the containing object. Its return value is the number of successful suffix-classification iterations, not the status of that update.

The apparent intent is to remove trailing whitespace-like characters. That interpretation is inferred from the backward traversal, the byte-classification mask, and the special treatment of numeric value 0x3000. The established behavior is narrower: it recognizes values according to a particular table predicate or exact equality with 0x3000. Neither the character encoding nor the meaning of the table’s classification bits is established.

The forward count and suffix count measure dependency invocations. They are character counts only if the dependencies implement the corresponding character traversal.

## Interface

**Input**

One argument identifies an object whose field at byte offset 4 contains a pointer to a byte sequence.

The observable interface requires:

- A readable object field at offset 4, even when that field contains a null pointer.
- If the field is non-null, readable sequence storage and cursor behavior sufficient for the forward traversal to reach a zero byte.
- Valid storage for any classification-table entry actually accessed.
- Valid inputs to the three dependencies described below.

There is no initial null check on the object argument itself. A null sequence pointer is supported; a null object pointer is not established as supported.

**Output**

The function returns an integer suffix count:

- Zero if the initially read sequence pointer is null.
- Zero if no backward result qualifies.
- Otherwise, the number of consecutive qualifying results obtained by backward traversal before termination.

The return value does not report whether the update dependency succeeded.

**Widths and representation**

The supplied representation uses signed integers for the forward count, suffix count, update length, and return value. It uses an unsigned integer for each backward result, and an unsigned byte for table entries. In the represented 32-bit interface, the unsigned result range is 0 through 4,294,967,295, and the signed count range is −2,147,483,648 through 2,147,483,647.

The table predicate examines bits selected by 0x06 in one byte. A match requires at least one selected bit, not both.

The object layout established here is limited to its sequence-pointer field at offset 4. Its other fields, overall size, pointer alignment requirements, and allocation policy are unknown.

**Arithmetic precondition**

The forward count increments once per completed forward-dependency call. The suffix count increments once per qualifying backward result. The update argument is the signed subtraction of suffix count from forward count.

The supplied evidence does not establish overflow handling. A defined-behavior contract therefore requires these increments and the subtraction to remain representable. No saturation, wraparound, or large-input error return is established.

## Behavior

### Initial read and null-sequence case

The function first reads the pointer field at object offset 4 into a local cursor. Both counts begin at zero.

If this initial pointer is null, it returns zero without invoking any dependency. This path does not read the classification-table pointer.

The null check applies only to the initially captured sequence pointer. The object’s pointer field is read again later; changes made by dependencies are not undone or screened by another equivalent null check.

### Forward traversal

For a non-null initial cursor, the function repeatedly examines the byte at the current cursor.

Each forward-loop condition evaluation has this observable order:

1. Read the global classification-table pointer and retain that pointer value.
2. Read the byte at the current cursor.
3. Continue the loop only if that byte is nonzero.

For each nonzero byte encountered at this check, F-d3922c7fe0ec receives the address of the local cursor. After that dependency returns, the forward count increases by one.

The function itself does not specify a cursor increment. Movement is delegated entirely to F-d3922c7fe0ec. In particular, the evidence does not establish whether one call advances by one byte, one encoded character, or another unit.

The loop ends only when a condition check observes a zero byte. Consequently, the table pointer retained for the backward phase is the pointer read immediately before that terminating zero-byte observation. It is not necessarily the pointer value seen before the first forward call.

No length bound, iteration limit, or progress check is present. If the dependency fails to move the cursor toward an accessible zero byte, termination and memory safety are not established.

### Backward traversal and classification

After the forward loop ends, the function attempts backward traversal.

Before each backward-dependency call, it rereads the sequence-pointer field at object offset 4. It proceeds only if that freshly read pointer is strictly below the current local cursor under the represented pointer comparison.

If the comparison fails, backward traversal stops immediately. No backward dependency or table-entry read occurs for that condition evaluation.

If the comparison succeeds, F-13bcc08bc7a3 receives the address of the local cursor and returns an unsigned value. Classification then follows these rules:

- For a value below 256, the function reads the corresponding byte from the retained table and accepts the value if the byte has any bit in mask 0x06 set.
- Numeric value 0x3000 is accepted independently of the table.
- Every other value is rejected.

For a value below 256, the table index is its low eight bits, which equal the value itself. Thus zero is a possible table index if the backward dependency returns zero; this function does not independently reject that result.

For a value of 256 or greater, no classification-table entry is accessed. In particular, accepting 0x3000 does not require an entry at that index.

A qualifying result increments the suffix count by one and permits another backward condition evaluation. A rejected result ends backward traversal without incrementing the count.

The function does not restore the local cursor after rejection. The backward dependency has already been called and may already have moved that cursor or changed other state. Only the cursor’s local storage is known to be local; the dependency’s other effects are unspecified.

The table pointer is not reloaded during backward traversal. Its entries are read as needed, however, so their contents are not snapshotted. A dependency-induced change to the global table-pointer variable would not redirect these reads; a change to storage addressed by the retained pointer could affect them.

### Conditional object update

After backward traversal stops:

- If the suffix count is zero, no update dependency is called.
- If the suffix count is positive, F-4fac58ca213e is called exactly once with the original object argument and the forward count minus the suffix count.

Any normal return value from this dependency is ignored. After it returns normally, F-dba4ee3fee8d returns the suffix count.

No explicit check requires the suffix count to be no greater than the forward count. With coherent forward and backward traversal over unchanged storage, that relationship is plausible, but it is not enforced here. A changed object field or inconsistent dependency behavior could produce a negative update argument without being rejected by this function, provided the subtraction itself remains representable.

## Side effects

The function’s directly visible writes are to its local cursor and counters. It contains no direct write to the object’s sequence-pointer field or sequence bytes.

Externally visible effects may nevertheless arise from all three dependencies:

- F-d3922c7fe0ec receives writable access to the local cursor.
- F-13bcc08bc7a3 receives the same form of writable cursor access.
- F-4fac58ca213e receives the original object argument and a signed update length.

No dependency may be assumed to preserve globals, object fields, sequence contents, or allocation state. The repeated object-field reads and the timing of the table-pointer read are therefore part of the contract, rather than incidental implementation details.

The wrapper contains no explicit allocation or release operation. This does not establish that the overall operation is allocation-free, that dependencies preserve ownership, or that they cannot release storage.

The address supplied to the traversal dependencies refers to local cursor storage. Retaining and using that address after the call’s lifetime would require a separate lifetime guarantee not present here.

No explicit error code, rollback, diagnostic, or recovery path is established. A rejected classification is ordinary traversal termination, not a reported error. Dependency failure, invalid memory access, nontermination, and exceptional control flow have no specified translation into a return value.

## Dependencies

| Opaque ID | Supplied inputs | Result used by this function | Established role |
|---|---|---|---|
| F-d3922c7fe0ec | Address of the local sequence cursor | Cursor changes; any return value is ignored | Invoked once for each forward iteration whose current byte is nonzero |
| F-13bcc08bc7a3 | Address of the local sequence cursor | Updated cursor and unsigned result | Invoked after a successful lower-bound comparison; result determines suffix qualification |
| F-4fac58ca213e | Original object argument; signed forward count minus suffix count | Return value ignored | Invoked once when the suffix count is positive |

These roles do not establish decoding rules, byte consumption, error sentinels, ownership transfer, or update semantics.

The classification table is a separate data dependency. Its retained pointer must address each byte entry that is actually consulted. The function supplies no table length or validity check.

## Acceptance examples

The following are conditional acceptance scenarios, not executed tests. Unless stated otherwise, they assume stable object and table storage, representable arithmetic, normally returning dependencies, and coherent forward and backward traversal.

1. **Null sequence field**

   Given a readable object whose sequence field is null, the result is zero. None of the three dependencies is invoked, and the global table-pointer variable is not read.

2. **Empty sequence**

   Given a non-null sequence pointer whose first byte is zero, and an unchanged object field, the forward phase reads the table pointer and observes the zero byte without calling F-d3922c7fe0ec. The backward lower-bound comparison fails because the cursor equals the sequence base. The result is zero, with no update call.

3. **Nonqualifying final unit**

   Suppose forward traversal completes five calls, and the first backward result is 0x41 with a table entry whose 0x06 bits are clear. F-13bcc08bc7a3 is called once, the suffix count remains zero, and F-4fac58ca213e is not called. The result is zero even though the backward dependency may have changed the local cursor.

4. **Two qualifying suffix units**

   Suppose forward traversal completes five calls. Backward traversal returns two qualifying results and then a rejecting result. The update dependency receives the original object and length three. The function returns two.

   If the second qualifying step instead reaches the unchanged sequence base, the next lower-bound comparison stops traversal without a third backward call; the same update length and return count follow.

5. **Either classification bit suffices**

   For a backward result below 256, table bytes 0x02, 0x04, and 0x06 each qualify. A table byte of 0x01 does not qualify. Other bits do not affect this decision.

6. **Special numeric value**

   A backward result of 0x3000 qualifies without a table-entry read. A result of 0x0100 rejects without a table-entry read. Recognition of 0x3000 alone does not establish that all Unicode whitespace qualifies.

7. **All traversed units qualify**

   Suppose forward traversal completes three calls and backward traversal returns three qualifying units while moving exactly back to an unchanged base. The update dependency receives zero as its length argument, and the function returns three. What a zero-length update does to the object remains dependency-defined.

8. **Changed global table pointer**

   Suppose the terminating forward condition retains table pointer A, after which a backward dependency changes the global pointer to B. A subsequent below-256 classification still accesses A. This scenario requires A to remain valid.

9. **Changed object sequence field**

   If a dependency changes the object’s sequence field, the next backward lower-bound comparison uses that changed field, not the initial sequence pointer. No assertion requires the two values to agree.

## Uncertainties

**Established static evidence**

- The object’s sequence pointer is read at byte offset 4.
- Forward progress and backward interpretation are delegated.
- The table pointer is captured during forward condition evaluation, including the terminating evaluation.
- Backward traversal rereads the object field before each dependency call.
- Qualification uses the unsigned result range check, byte mask 0x06, and exact special value 0x3000.
- The update occurs only for a positive suffix count.
- The update result is ignored.

**Inferred intent**

- The traversal dependencies likely move between encoded text units.
- The table likely classifies byte-sized characters.
- The operation likely trims a whitespace-like suffix.
- The update length likely identifies a retained prefix.

These interpretations are not substitutes for the established predicates and call order.

**documented_only unknowns**

- Exact encoding and malformed-sequence behavior.
- Exact cursor movement and failure behavior of either traversal dependency.
- Classification-table contents, lifetime, and initialization.
- Whether the update mutates in place, reallocates, terminates storage, or releases memory.
- Ownership of the object and sequence.
- Concurrency, synchronization, and reentrancy requirements.
- Behavior outside valid-memory and representable-arithmetic preconditions.
- Pointer-ordering guarantees if dependencies move the cursor outside the original sequence storage.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping. Dependency bodies, runtime traces, table contents, and broader object definitions were not supplied.

The evidence supports the local branch structure, call order, field offset, classification predicate, counter updates, and return behavior described above. It does not establish the semantic names of the dependencies or their wider side effects.

No runtime tests have been performed. The acceptance examples specify conditional expected behavior rather than observed execution results.
