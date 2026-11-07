<!-- Trusted provenance: F-a01d4dfb5f88; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a01d4dfb5f88

## Purpose

F-a01d4dfb5f88 obtains a 32-bit value associated with an unsigned index in a mutable, word-indexed table. It requests table growth when the index is outside the recorded extent, removes a low-bit-tagged entry from a linked structure when necessary, and creates a partially initialized 12-byte object if the selected table entry is zero.

The apparent intent is lazy acquisition of an object from a table that also encodes unused entries as linked-list nodes. That interpretation is inferred. The established behavior is manipulation of 32-bit words, conditional dependency calls, and return of the selected table word.

The function is not a checked lookup. It contains no explicit rejection of an invalid index, no verification that a growth request succeeded, no allocation-failure branch, and no traversal bound.

## Interface

### Inputs

The function receives:

- A pointer to a mutable control structure containing at least four consecutive 32-bit words.
- An unsigned 32-bit index selecting a table entry.

The control structure fields used here are:

| Byte offset | Observed role |
|---|---|
| 0 | Address representation of the table base |
| 4 | Not directly accessed by this function |
| 8 | Recorded extent used for the growth decision |
| 12 | Head index or terminal value for the tagged-entry structure |

The table consists of 32-bit entries at four-byte intervals. The selected entry is addressed using the table base plus four times the index.

The available representation stores both addresses and entry values in 32-bit words. This contract therefore describes a 32-bit address model; it does not establish portability to a wider-pointer environment.

### Entry representations

The function distinguishes three cases:

- **Low bit set:** the entry is processed as a tagged link.
- **Exactly zero:** the entry requires object creation.
- **Nonzero with low bit clear:** the entry is returned without direct modification.

For a tagged entry, shifting its unsigned 32-bit value right by one produces the value used as the next index. A tagged word of all ones is treated specially only through dependency results; this function does not directly compare a table entry against that value.

The control structure’s head field can be written with `0xffffffff`.

### Result

The return value is the 32-bit word read from the selected table entry at the end of execution.

In the ordinary object-acquisition interpretation, it is an object address. The interface itself does not validate that interpretation: a nonzero, low-bit-clear input word is returned without checking whether it denotes a valid object.

### Preconditions

Normal completion requires sufficient valid memory for all accesses actually reached:

- A readable and writable control structure.
- A usable table base after any growth operation.
- Valid table storage for the selected entry and every traversed entry.
- Successful, writable 12-byte storage if object creation is needed.
- A tagged-entry structure whose traversal and dependency responses permit progress to a valid stopping point.

These are operational preconditions, not conditions checked by the function. Synchronization, malformed-link handling, and failure recovery are not established.

## Behavior

### 1. Extent check and possible growth

The function reads the control structure’s recorded extent as an unsigned 32-bit value and compares it with the requested index.

If the extent is less than or equal to the index, it calls F-b018ee7e1687 with:

- The control-structure pointer.
- The index plus one, calculated as an unsigned 32-bit value.

The addition wraps at 32 bits. In particular, an index of `0xffffffff` produces a growth argument of zero.

The growth dependency’s return value is not used. The function does not repeat the extent check or explicitly verify that the requested entry became accessible. It next reads the selected table entry using the table base available after that call.

If the extent is strictly greater than the index, this dependency is not called.

### 2. Tagged-entry detection

The selected table word is read and its low bit tested.

If the low bit is clear, the function skips all linked-structure removal work. It proceeds directly to the zero-entry check.

If the low bit is set, the function reads the head field and selects one of two removal paths.

### 3. Removal when the selected index is the head

When the head equals the requested index, the function calls F-a7b33ca08f0c. At this call site, the supplied representation shows no explicit argument.

The dependency’s result is interpreted only as zero or nonzero:

- **Zero:** the function reads the selected table entry after the call, shifts the unsigned word right by one, and uses that value as the new head.
- **Nonzero:** the new head is `0xffffffff`.

It then writes the new head to byte offset 12 of the control structure.

The post-call read matters. If the dependency changes the table base or the selected table word, the zero-result path uses the state read after that call, not a saved copy of the original tagged word.

The exact call signature of F-a7b33ca08f0c remains uncertain because other call sites visibly supply an address.

### 4. Removal when the selected index is not the head

When the head differs from the requested index, the function searches from the head.

Each iteration retains a current index and calls F-a7b33ca08f0c with the address of that current table entry. It then proceeds according to the result:

- **Nonzero:** stop the search immediately.
- **Zero:** read the current entry after the call, shift its unsigned 32-bit value right by one, and treat the result as the next index. If that next index is the requested index, stop; otherwise continue from that next index.

The loop therefore stops either because a traversed entry’s decoded successor equals the requested index or because the dependency reports a nonzero result.

There is no direct check for an invalid head, an out-of-range successor, repeated indices, or a cycle. A malformed structure can cause invalid accesses or nontermination.

After the search stops, the function calls F-a7b33ca08f0c again, this time with the address of the selected entry. This call occurs even when the search stopped because an earlier dependency result was nonzero.

The function then obtains the table base again and rewrites the entry at the retained current index:

- **Zero result from the selected-entry call:** read the selected entry, clear its low bit, set that bit to one, and store the resulting word in the retained current entry.
- **Nonzero result:** store `0xffffffff` in the retained current entry.

The clear-and-set operation preserves bits 1 through 31 and forces bit 0 to one. If the selected entry remains tagged, this reproduces its complete tagged value. If a dependency changed it to an even value, the stored value is that modified value with its low bit set.

On a well-formed linked structure with suitable dependency behavior, the retained current entry is the predecessor of the selected entry. That predecessor relationship is inferred intent, not an unconditional guarantee: a nonzero search result permits an earlier stop.

This path does not directly write the control structure’s head field.

### 5. Clear the selected tagged entry

After either removal path, the function writes a full 32-bit zero to the selected entry.

Its destination is calculated using the table base read at this point. It is not necessarily the same address used before dependency calls.

Thus, tagged-entry processing completes before object creation begins. The function does not first allocate an object and then attempt to unlink the entry.

### 6. Create an object for a zero entry

The function calculates and retains the address of the selected table entry, then reads that entry.

If the word is nonzero, no allocation occurs.

If the word is zero, the function calls F-8157a180d901 with the size argument 12. It treats the returned value as a writable address and performs these stores in order:

1. A 32-bit zero at object byte offset 0.
2. A 32-bit zero at object byte offset 4.
3. A single zero byte at object byte offset 8.
4. The returned object address, represented as a 32-bit word, into the previously retained table-entry address.

The function does not initialize object bytes 9 through 11. Their contents depend on the storage returned by the dependency.

There is no explicit null or failure check before the initialization stores. A failure return that is not writable storage does not produce a defined error result from this function.

The table-entry address is retained across the allocation call. If that dependency relocates the table, the object address is still written to the old retained entry address. The implementation does not compensate for such relocation.

### 7. Return the current selected word

The function finally reads the table base again and returns the word at the requested index.

This is a fresh table lookup, not a direct return of the allocation result or an earlier cached entry value. Consequently, if a dependency changes the table base, the final return can read a different location from the retained location used for the allocation-result store.

## Side effects

### Direct state changes

Depending on the path, the function can directly:

- Update the control structure’s head field.
- Replace one retained linked-structure entry.
- Clear the selected tagged entry.
- Initialize nine bytes of a newly obtained 12-byte object.
- Store that object’s address into a table entry.

The function does not directly access the control word at byte offset 4, directly update the recorded extent, or directly free memory.

### Dependency-visible ordering

The dependency-call order is:

1. Optional growth request.
2. Optional tagged-entry processing:
   - One head-path predicate call; or
   - One or more traversal predicate calls, followed by a selected-entry predicate call.
3. Optional 12-byte allocation request.

The selected entry is cleared after tagged-entry processing and before allocation. The object is initialized before its address is published into the retained table entry.

These are execution-order observations. No atomic operations, locks, or memory-ordering guarantees are shown.

### Ownership

The allocated address is stored in the table, which suggests table-associated retention. The evidence does not establish:

- Who owns the returned object.
- Whether the caller may release it.
- Whether any dependency releases or replaces memory.
- Whether repeated acquisition extends an object’s lifetime.
- Whether the returned value is borrowed, shared, or exclusively owned.

No ownership-preservation guarantee may be assumed for the dependencies.

## Dependencies

| Opaque ID | Established use | Not established |
|---|---|---|
| F-b018ee7e1687 | Called with the control structure and wrapped index-plus-one when the recorded extent does not exceed the index. Its result is ignored. | Growth policy, failure behavior, resulting extent, table initialization, preservation of other state. |
| F-a7b33ca08f0c | Its result selects zero/nonzero branches during tagged-entry removal. Traversal and selected-entry calls visibly pass an entry address; the head call has no explicit argument in the supplied representation. | Exact signature, meaning of the result, memory access behavior, side effects, ownership behavior. |
| F-8157a180d901 | Called with size 12; its result is used as writable storage and encoded into a table entry. | Allocation guarantees, alignment, failure convention, initial contents, global-state preservation, release requirements. |

The function’s repeated reads after dependency calls must not be replaced by an assumption that those calls preserve the control structure, table contents, or global state.

## Acceptance examples

These examples are conditional static expectations, not executed tests. Unless stated otherwise, assume valid memory, no concurrent mutation, and dependencies that preserve unrelated state.

### Existing nonzero entry

Given an extent greater than index 3 and a selected word of `0x1000`:

- No growth request occurs.
- No tagged-entry predicate call occurs.
- No allocation occurs.
- The function returns `0x1000`.

Its interpretation as a valid object address is outside this function’s checks.

### Zero entry

Given an in-range selected word of zero and an allocation result denoting valid 12-byte storage:

- F-8157a180d901 receives 12.
- Object bytes 0 through 8 become zero.
- Bytes 9 through 11 are not written by this function.
- The object address is stored in the selected entry and returned, provided the table base remains unchanged.

### Extent boundary

Given recorded extent 8 and requested index 8, F-b018ee7e1687 receives the control structure and 9 before the selected entry is inspected.

Given recorded extent 9 and requested index 8, that growth call does not occur.

### Tagged head with a successor

Given head index 4, requested index 4, and selected word `0x0000000d`, assume the head-path dependency returns zero and leaves that word unchanged:

- The new head is 6.
- The selected entry is cleared.
- Object creation follows because the cleared entry is zero.

### Tagged head with a nonzero predicate result

Given a tagged selected entry that is also the head, a nonzero head-path result causes the head to become `0xffffffff`. The selected entry is still cleared, and object creation follows under ordinary stable-table conditions.

### Non-head unlink

Given head index 2 and requested index 5, assume entry 2 contains `0x0000000b`, entry 5 contains `0x0000000f`, and both relevant predicate calls return zero without mutation:

- Entry 2 decodes to successor 5, ending the search.
- Entry 2 becomes `0x0000000f`.
- Entry 5 is cleared and then populated through allocation.
- The head remains 2 through this function’s direct writes.

### Allocation failure

If the allocation dependency returns a null address, the function does not return a null failure result through an explicit check. It attempts initialization stores through that address. Any resulting fault or external handling is environment-dependent.

### Maximum index

For index `0xffffffff`, the growth comparison necessarily requests growth, and the requested size argument wraps to zero. Table addressing also involves 32-bit scaled-index arithmetic. Safe access or successful completion is not established for this input.

## Uncertainties

The following remain **documented_only**:

- The logical meaning of the recorded extent beyond its comparison role.
- The exact terminal-entry encoding and why F-a7b33ca08f0c is needed to interpret it.
- The complete calling convention of F-a7b33ca08f0c, especially its head-path invocation.
- Whether dependencies mutate the table, control structure, globals, or storage lifetimes.
- Allocation alignment, failure behavior, and ownership rules.
- Intended bounds on indices and decoded links.
- Behavior under cycles, invalid pointers, concurrent access, or reentrant dependency calls.
- Whether bytes 9 through 11 are padding or meaningful object state.
- The intended relationship between 32-bit arithmetic wraparound and valid address ranges.

The free-list and lazy-object interpretations are inferred intent. They do not justify adding validation, rollback, error returns, synchronization, or dependency-preservation guarantees absent from the evidence.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping.

Established observations include the 32-bit field and entry operations, low-bit tagging, unsigned successor decoding, conditional call order, explicit writes, partial object initialization, retained allocation-store address, and final fresh table lookup.

No dependency implementations, runtime traces, external type definitions, or ownership documentation were available. No runtime tests have been performed.
