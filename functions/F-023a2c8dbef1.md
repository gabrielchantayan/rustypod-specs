<!-- Trusted provenance: F-023a2c8dbef1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-023a2c8dbef1

## Purpose

F-023a2c8dbef1 searches an integer sequence associated with a supplied record for the first occurrence of a supplied value. If a match is found, it clears one associated byte, invokes a second dependency, and conditionally invokes an unresolved indirect target.

A preparatory dependency is invoked before any search decision, including when the record’s count is zero or negative. A failed search therefore does not imply an absence of side effects.

This contract describes behavior established by the supplied static representation. It does not establish the application meaning of the record, the cleared byte, either dependency, or the indirect invocation. In particular, interpretations such as “remove,” “disable,” “unlock,” or “notify” are not established.

## Interface

### Inputs

The function has two explicit inputs:

- **Record address:** an address used both as the base of signed integer fields and as the base for byte-offset addressing.
- **Search value:** a signed integer compared for equality with candidate entries.

The supplied representation uses 32-bit integer fields and a 32-bit address model. The offsets below describe that represented model. The actual machine ABI, alignment requirements, and exact source-language types have not been independently verified.

### Referenced record layout

| Byte offset from record address | Represented access | Role |
|---|---|---|
| 0 | Read a signed 32-bit integer | Current search count |
| 12 + 4 × entry index | Read a signed 32-bit integer | Candidate value |
| 52 + matched index | Write one byte | Byte cleared after a match |
| 64 | Pass the address to the first dependency | Preparatory dependency argument |
| 64 | Later read a 32-bit address value | Optional pointer to an integer |
| 72 | Pass the address to the second dependency | Post-match dependency argument |

The addresses passed to the dependencies are interior addresses within the record’s address space. The function does not first load the contents at offsets 64 or 72 to obtain those arguments.

The layout does not establish separate, nonoverlapping arrays. Candidate entries, cleared bytes, and the field at offset 64 can overlap for some indices. No fixed sequence capacity is established.

### Output

The function has no explicit return value. It returns normally when:

- The current count does not exceed the current search index.
- A matched entry has been processed and the later pointer field is null.
- That pointer is nonnull but the integer read through it is zero.
- The guarded indirect invocation returns.

There is no explicit success indicator, match index, or error code.

### Preconditions

For a defined, normally completing execution:

- The record address and all addresses actually accessed must be valid for the corresponding operations.
- The dependencies must accept the supplied interior addresses.
- Each candidate read must address a readable 32-bit integer.
- The selected byte must be writable if a match occurs.
- A nonnull pointer subsequently loaded from offset 64 must address a readable 32-bit integer.
- If the indirect invocation is reached, its target and calling environment must be valid.
- Index and address calculations must remain meaningful in the represented address model.

The function provides no explicit validation of these conditions. It does not check the record address for null.

## Behavior

### 1. Invoke the preparatory dependency

Before reading the count or any candidate, the function invokes **F-338d16f9ce72**, supplying the address 64 bytes beyond the record address.

Its return value is not used. Its effects on the record, other memory, ownership, or global state are unknown.

Consequently, the search is governed by the state observed after this dependency returns, not necessarily the state present at function entry. Even an initially empty or negative-count record reaches this dependency first.

### 2. Search from index zero

The search starts at index zero. At each iteration, the function first reads the signed 32-bit count at offset 0.

If the count is less than or equal to the current index, the function returns. Otherwise, it reads the candidate at offset 12 plus four times the current index and compares it with the supplied search value.

If they differ, the index increases by one and the process repeats. If they are equal, the search ends immediately at that index.

Observable consequences include:

- With a stable positive count, entries are examined in increasing index order.
- The first matching entry is selected.
- Entries after the selected entry are not examined.
- A zero or negative count causes no candidate reads, provided that is the count observed after the preparatory dependency.
- The count is read again for each loop decision; it is not retained as a single entry-time bound.
- No second dependency or indirect target is invoked merely because the search exhausts its current bound.

No synchronization mechanism is visible. Repeated count reads in the supplied representation do not establish atomicity, a supported concurrent-modification protocol, or any particular hardware memory-order guarantee.

### 3. Clear the selected byte

After finding a match, the function writes zero to exactly one byte at offset 52 plus the matched index.

This is a byte write, not a 32-bit integer write. It is performed even if the byte already contains zero.

The function does not visibly:

- Decrement the count.
- Shift entries.
- Replace the matched candidate.
- Clear corresponding bytes for later duplicate matches.
- Test the selected byte’s previous value.

Those statements describe the function’s direct operations. They do not restrict effects performed by dependencies or effects arising from overlapping fields.

### 4. Invoke the post-match dependency

After the byte write, the function invokes **F-b8a325ddd21b**, supplying the address 72 bytes beyond the record address.

Its return value is not used.

The ordering is significant: the byte has already been cleared when this dependency is invoked. This dependency may alter the record, globals, or other memory; no preservation guarantee is available.

### 5. Evaluate the indirect-invocation guard

After the post-match dependency returns, the function reads the address value stored at record offset 64.

If that address is null, the function does not dereference it and does not perform the indirect invocation.

If the address is nonnull, the function reads one signed 32-bit integer through it:

- Zero suppresses the indirect invocation.
- Any nonzero value permits the indirect invocation, including a negative value.

The pointer and pointee are observed after the post-match dependency. Their entry-time values do not determine this decision.

The integer read through the pointer is only a zero/nonzero gate. It is not established as an argument to the indirect target, a count of invocations, or an address.

### 6. Perform the guarded indirect invocation

When the loaded pointer and its integer value are both nonzero, the function invokes an unresolved indirect target obtained through a separate static location.

The supplied representation shows no explicit arguments. That does not establish a complete calling convention, absence of implicit arguments, or behavior of the target.

There is no visible loop around this invocation. If it returns normally, F-023a2c8dbef1 returns without further explicit record operations.

## Side effects

### Direct memory effects

The only explicit memory write performed by this function is the one-byte zero store following a successful match.

Its relevant direct memory accesses occur in this order:

1. After the preparatory dependency, read the current count.
2. If the bound permits, read the current candidate.
3. Repeat those reads for later indices until exhaustion or a match.
4. On a match, clear the selected byte.
5. After the post-match dependency, read the address at offset 64.
6. If nonnull, read the integer through that address.
7. If that integer is nonzero, obtain and invoke the indirect target.

This is the ordering expressed by the supplied representation, not an independently measured instruction or bus-access trace.

### Dependency effects

Both named dependencies can produce effects not visible here. The indirect target can do the same. None may be assumed to preserve globals, preserve the record, allocate or release memory, retain arguments, or avoid reentrancy.

The function itself contains no explicit allocation, deallocation, ownership transfer, or reference-count adjustment. That does not establish the absence of such behavior inside a dependency.

### Overlapping layout consequences

The offset formulas permit aliasing:

- Candidate index 10 is read at offset 52, where the selected-byte region begins.
- A match at index 12 clears the byte at offset 64, which is also the beginning of the later pointer field.
- A match at index 20 clears the byte at offset 72, the address supplied to the post-match dependency.

These are arithmetic consequences of the represented accesses, not evidence that such indices are valid for an intended record type.

If a valid record permits these indices, the write affects the shared storage. In particular, the pointer eventually read at offset 64 is not guaranteed to retain its earlier representation: the byte write or the post-match dependency may alter it. Endianness and subsequent dependency effects determine any resulting pointer value.

### Errors and abnormal completion

No explicit error-reporting or recovery path is shown.

Invalid memory, an invalid indirect target, or abnormal behavior within a dependency can prevent normal completion. The supplied evidence does not establish an exception policy, fault translation, rollback, or cleanup guarantee.

If abnormal completion occurs after the byte write, that write has already been issued in the represented sequence. There is no visible compensating write.

## Dependencies

| Opaque function ID | Invocation condition | Explicit argument | Established use of result |
|---|---|---|---|
| **F-338d16f9ce72** | Before every search decision | Record address + 64 bytes | None |
| **F-b8a325ddd21b** | After the first matching candidate and the byte clear | Record address + 72 bytes | None |

There is also an **unresolved indirect dependency**. No opaque function ID is supplied for its target. It is invoked only after a successful match, the second dependency, and a nonzero pointer/pointee guard.

Unknown dependency semantics are **documented_only**. The available evidence does not justify assigning these dependencies locking, notification, destruction, or ownership roles.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests. Unless stated otherwise, assume valid memory, meaningful arithmetic, normally returning dependencies, and no dependency changes to relevant state.

### Empty or negative count

If the count observed after F-338d16f9ce72 is zero or negative:

- F-338d16f9ce72 has still been invoked with record address + 64.
- No candidate is read.
- No selected byte is written.
- F-b8a325ddd21b and the indirect target are not invoked.

### No matching candidate

If the stable count is three and all three candidates differ from the search value:

- Candidates at offsets 12, 16, and 20 are examined in that order.
- The next bound check ends the search.
- The function performs no explicit memory write.
- The post-match dependency and indirect target are not invoked.

Effects of the preparatory dependency remain possible.

### First of two matching candidates

If the stable count is four and candidates at indices one and three both match:

- Index one is selected.
- The byte at offset 53 is cleared.
- Candidates at indices two and three are not examined.
- F-b8a325ddd21b is invoked after the clear.
- The byte corresponding to index three is not directly cleared by this invocation.

### Null post-dependency pointer

If a match occurs and the address at offset 64 is null after F-b8a325ddd21b returns:

- The selected byte has already been cleared.
- No pointee read occurs.
- No indirect invocation occurs.

### Zero versus nonzero pointee

If the post-dependency pointer is valid and nonnull:

- A pointee value of zero suppresses the indirect invocation.
- A pointee value of negative one permits it.
- The magnitude of a nonzero value does not change the visible number of indirect invocations.

### Dependency changes determine later behavior

If F-338d16f9ce72 changes the count from zero to one and establishes a valid matching first candidate, the function can process that match.

If F-b8a325ddd21b replaces the pointer at offset 64, the guard uses the replacement pointer, not the pointer present before that call.

These criteria must not assume either dependency preserves the fields it receives or neighboring fields.

## Uncertainties

The following are **documented_only**:

- The original record type, allocation size, intended capacity, and valid index range.
- The application meaning of the count, candidates, cleared bytes, and guarded integer.
- Exact ABI, endianness, alignment rules, and machine-level access details.
- Whether the interior addresses denote embedded objects or another structure.
- Dependency ownership, lifetime, synchronization, and reentrancy behavior.
- The indirect target’s identity, full calling convention, and effects.
- Whether apparently overlapping regions are reachable in valid application states.
- Behavior outside valid arithmetic and memory preconditions.

The signed count comparison is established by the supplied representation. For a stable count no greater than the largest signed 32-bit integer, the loop bound alone does not require the index to increment beyond that maximum. However, candidate indexing and address calculations can exceed valid arithmetic or storage limits much earlier. The count check is therefore not a storage-safety guarantee.

## Evidence

**Established static evidence:** an unconditional preparatory call; repeated signed count checks; increasing-index equality search; first-match selection; a one-byte zero store; a post-match dependency call; a subsequent pointer load and conditional pointee read; and a guarded indirect invocation.

**Inferred intent:** the byte write may deactivate or acknowledge an associated entry, and the indirect invocation may notify another component. Neither interpretation is required by this contract.

**Unknowns:** dependency implementations, actual record invariants, indirect-target semantics, and ABI details are unavailable. No preservation or ownership promises are inferred from names or comments.

No runtime tests have been performed. The acceptance examples specify conditional observable behavior; they are not execution results.
