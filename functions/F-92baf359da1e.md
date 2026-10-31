<!-- Trusted provenance: F-92baf359da1e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-92baf359da1e

## Purpose

F-92baf359da1e searches a dependency-provided collection for an entry whose 32-bit field at byte offset 4 equals a supplied 32-bit key. On the first match reached by its traversal, it writes the entry’s address representation to a caller-provided output location and returns the 32-bit value 1. If the collection-size query reports that the current traversal index is outside the collection, it returns the 32-bit value 0 without directly writing the output.

The traversal is not an unrestricted increasing 32-bit index. Its update narrows the next index to 16 bits and sign-extends that value back to 32 bits under the represented arithmetic-right-shift model. This affects searches that reach index 32,767.

The apparent intent is a lookup over a collection of entry addresses. That intent does not establish collection stability, ownership, valid entry layouts beyond the accessed field, or ordinary array traversal for large collections.

## Interface

### Inputs

The function has four positional inputs:

1. **First opaque input:** a 32-bit value. The visible body does not inspect it or explicitly pass it to either dependency.
2. **Second opaque input:** a 32-bit value. The visible body likewise does not inspect or explicitly forward it.
3. **Search key:** a 32-bit integer compared for exact equality with the entry field at byte offset 4. This is a bit-pattern equality test; signed ordering is not involved.
4. **Output location:** a pointer to storage capable of receiving one 32-bit integer. On success, the stored value is the matching entry’s address representation, not the field used for comparison.

No null check, capacity argument, or explicit validation is present for the output location. The function directly dereferences it only after finding a match. A successful call therefore requires writable output storage. A call that finds no match does not directly dereference this location, although dependency behavior remains unspecified.

### Return value

Normal returns are limited to:

- **0:** the most recent size query returned an unsigned 32-bit value less than or equal to the current index.
- **1:** an entry’s comparison field matched the key, and the output write completed.

There is no distinct return value for malformed collection state, inaccessible memory, dependency failure, or a failed output write. The visible body supplies no recovery path for those conditions.

### Collection representation

The visible accesses require the following representation:

- F-a516db142ed4 supplies a pointer to collection state.
- The first 32-bit word at that state address supplies an entry-table base address representation.
- The table is indexed in units of four bytes.
- Each selected table slot contains a 32-bit entry address representation.
- The comparison field is a 32-bit word located four bytes after the selected entry address.

No other collection-state fields or entry fields are directly inspected.

This describes the widths and offsets represented by the supplied source. It does not independently establish a portable native-pointer ABI or the complete layout of either object.

## Behavior

### Acquisition and initial state

The first visible action is a call to F-a516db142ed4 with no explicit arguments. Its result becomes the collection-state pointer used throughout this invocation.

The initial traversal index is the unsigned 32-bit value zero.

The function does not reacquire the collection-state pointer on later iterations. It does, however, query the collection size and reload the table base on each iteration that proceeds to an entry read.

### Per-iteration ordering

Each iteration has the following observable order:

1. Call F-4e8e9be587c6 with the retained collection-state pointer.
2. Interpret its result as an unsigned 32-bit size.
3. Compare that size with the current unsigned 32-bit index.
4. If the size is less than or equal to the index, return 0 immediately.
5. Otherwise, read the first 32-bit word of collection state to obtain the table base.
6. Read the 32-bit entry address representation from the selected table slot.
7. Read the entry’s 32-bit comparison field at byte offset 4.
8. Compare that field with the search key.
9. On equality, write the selected entry address representation to the output location and return 1.
10. On inequality, update the index and repeat.

Consequently, even an empty collection causes both the acquisition dependency and one size-query call. It does not cause a direct table-base, table-slot, entry-field, or output access.

A successful match causes no subsequent size query. For an ordinary stable collection whose size is within the nonwrapping traversal range, an unsuccessful search performs one additional size query after inspecting its final entry.

### Matching and output

The first matching entry encountered in traversal order wins. Later entries are not inspected.

The output contains the address representation read from the table slot. It does not contain the matching key, the index, the table-slot address, or a copied entry object.

The output store occurs after the comparison-field read. If the output location aliases collection state, table storage, or entry storage, the function still performs that store after determining the match. There is no alias check. Such aliasing can modify the collection as a side effect of a successful lookup.

### Index arithmetic and large collections

Index arithmetic is materially narrower than an ordinary 32-bit increment.

Under the represented 32-bit arithmetic and arithmetic signed-right-shift interpretation, each nonmatching iteration increments the index, retains the low 16 bits through a left shift, and sign-extends those bits back into a 32-bit result. The result is then held as an unsigned index.

The resulting progression begins:

- 0 through 32,767, in increasing order;
- then 4,294,934,528, the unsigned representation of signed −32,768;
- then successively larger unsigned representations of negative 16-bit values, if the size checks permit continued reads.

The index does **not** become 32,768 after inspecting index 32,767.

The size comparison remains unsigned. Thus, after a nonmatch at index 32,767, an ordinary size such as 32,769 is less than the next index. The function returns 0 at that next check and never examines ordinary index 32,768.

Table addressing uses four-byte slot scaling. With 32-bit modular address arithmetic, the large unsigned index 4,294,934,528 produces a byte displacement equivalent to −131,072 modulo 2³². If a sufficiently large size query permits that iteration, the resulting access need not be within the conventional forward extent of the table.

The largest unsigned index, 4,294,967,295, cannot pass the bounds check: no unsigned 32-bit size is greater than it. Therefore, under this arithmetic model, the visible traversal cannot wrap through that index back to zero. Assuming dependencies return and memory accesses complete, at most 65,535 entries can be inspected before a match or bounds termination. This is a derived arithmetic bound, not a runtime observation.

### Changing collection state

Neither the size nor the table base is snapshotted for the whole search.

F-4e8e9be587c6 runs before every bounds decision. Its effects, including any changes to collection state or table contents, can affect the subsequent reads. A shrink can terminate the search; a growth can permit further traversal; a changed table base can redirect a later entry read.

The bounds check does not establish that the table or selected entry remains valid until the subsequent memory accesses. No visible locking, version check, retry, or consistency validation is present.

## Side effects

### Direct effects

The only explicit memory write in the visible body is the successful 32-bit output store.

The function directly reads:

- the collection-state word containing the table base, once per inspected entry;
- one table slot per inspected entry;
- one comparison field per inspected entry.

It does not visibly allocate memory, copy entry objects, release memory, remove entries, or update collection metadata.

### Dependency-mediated effects

F-a516db142ed4 is invoked once. F-4e8e9be587c6 is invoked once per attempted iteration, including the final terminating bounds check when one occurs.

Their effects are unknown. The absence of visible allocation or release in this function does not establish that its dependencies perform neither operation. Likewise, the function must not be described as globally read-only or side-effect-free merely because its own search accesses are reads.

### Ownership and lifetime

The successful output is a reference-like address value, not evidence of ownership transfer.

No visible operation retains the selected entry, extends its lifetime, or releases the collection-state pointer. Whether the caller may modify, retain, or release the returned entry is **documented_only**.

On a return of 0, this function itself leaves the output storage unwritten. That is not a guarantee that the storage is unchanged across the entire call if a dependency or concurrent actor can access it.

## Dependencies

| Opaque ID | Visible use | Established contract at this call site | Unknowns |
|---|---|---|---|
| F-a516db142ed4 | Called once before traversal, without explicit arguments | Its return is used as a pointer to collection state | Validity, failure representation, lifetime, ownership, global effects, allocation or release behavior |
| F-4e8e9be587c6 | Called before each bounds decision with the retained collection-state pointer | Its return is consumed as an unsigned 32-bit size | Whether it actually reports an entry count, whether results are stable, validation behavior, mutation, global effects, allocation or release behavior |

The interpretation of the second dependency’s result as a collection size follows from its use in the bounds comparison. Its name and implementation are not available as evidence.

Safe completion requires accessible storage for every direct read that execution reaches, and writable output storage if a match occurs. No visible check establishes those conditions.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. They assume the described dependency results, valid reached memory accesses, and no unmentioned interference.

### Empty collection

If acquisition returns usable collection state and the first size query returns 0:

- the function returns 0;
- the size dependency is called once;
- no table base, table slot, or entry comparison field is directly read;
- the output is not directly written.

### Match at the first entry

If the first size query returns a value greater than zero and entry zero’s comparison field equals the key:

- only entry zero is inspected;
- its address representation is written to the output;
- the function returns 1;
- no second size query occurs.

### Duplicate matching keys

If entries zero and one both match, and the first size query permits entry zero:

- entry zero is selected;
- entry one is not inspected;
- the output identifies entry zero.

### Ordinary unsuccessful search

For a stable three-entry collection with no matching field:

- size is queried before entries zero, one, and two;
- each of those entries is inspected in that order;
- size is queried again at index three;
- the function returns 0 without directly writing the output.

### Size changes between iterations

If entry zero does not match, and the next size query returns 1:

- the next index is one;
- the bounds check terminates the search;
- entry one is not read, even if earlier collection state suggested more entries.

### Match beyond the signed 16-bit boundary

For a stable size of 32,769, with no match among entries zero through 32,767 and a matching entry at ordinary index 32,768:

- the first 32,768 entries are inspected;
- the next index is the unsigned representation of −32,768;
- the next bounds check returns 0;
- ordinary entry 32,768 is not inspected.

### Very large reported size

If a nonmatch at index 32,767 is followed by a size greater than 4,294,934,528, the next iteration passes the unsigned bounds check.

Under the represented 32-bit address model, its table access uses a displacement equivalent to −131,072 bytes from the currently loaded table base. Acceptance of a normal result requires that this reached slot and its referenced entry field are accessible. The visible function does not validate them.

### Invalid output storage

If a match is found but the output location is not writable, no defined successful result is established by the visible body. It attempts the store before returning 1. There is no alternate error return for this condition.

## Uncertainties

The following remain **documented_only**:

- The full collection-state and entry layouts.
- The semantic meaning of the comparison field.
- Dependency failure conventions and exceptional behavior.
- Ownership, lifetime, and release rules for all returned addresses.
- Thread safety, reentrancy, and synchronization requirements.
- Whether either dependency modifies globals, collection storage, or caller-visible memory.
- Alignment requirements and the platform’s response to invalid accesses.
- Whether negative 32-bit size encodings can originate from the size dependency; at this call site, such bit patterns are interpreted as large unsigned sizes.
- Whether the unusual index narrowing is deliberate or an artifact of the supplied representation.
- Independent confirmation of the target ABI and signed-right-shift behavior.

The arithmetic boundary behavior above is established under the source-represented 32-bit, arithmetic-right-shift model. It has not been independently validated against a running target.

## Evidence

**Established static evidence:** one acquisition call; repeated size queries; unsigned size/index comparison; four-byte table stride; a 32-bit comparison read at entry offset 4; first-match output storage; return values 0 and 1; and the narrowing/sign-extension index update.

**Inferred intent:** lookup by a key field in a collection of entry addresses. No stronger claim about object type, ownership, synchronization, or collection invariants follows.

**Unknown:** dependency internals, complete ABI details, runtime memory validity, and behavior outside the stated access preconditions.

No runtime tests have been performed. All acceptance examples and arithmetic consequences are derived from the supplied static evidence.
