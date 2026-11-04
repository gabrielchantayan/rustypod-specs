<!-- Trusted provenance: F-3a1cdd4c1ec9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3a1cdd4c1ec9

## Purpose

F-3a1cdd4c1ec9 performs an ordered search of a supplied collection using a value obtained from a separate input. It stops at the first entry whose comparison dependency reports equality.

A matching entry produces a nonzero result only if it passes two gates: its second 32-bit word must be zero, and a dependency invoked on its third 32-bit word must return nonzero. Even after those gates pass, the selected output word can itself be zero.

Consequently, a zero result does not identify a unique outcome. It can indicate failed input resolution, an absent outer collection, no matching entry, a rejecting first match, or a selected payload whose value is zero.

The apparent intent is a keyed lookup with an entry-level exclusion field and an associated collection from which a first payload is selected. Those semantic labels are inferred. The established behavior is the dependency-call sequence, conditional reads, and early returns specified below.

## Interface

### Inputs

The function accepts two arguments:

- **Outer collection:** A 32-bit value, represented as a signed integer in the supplied evidence. Zero suppresses the search, but only after the second input has been passed to the resolution dependency. Nonzero values are passed unchanged to the collection dependencies.
- **Lookup input:** An opaque 32-bit word passed unchanged to F-83fcd2d25cd8. Its encoding and validity requirements are unknown.

The outer collection is not directly dereferenced by this function. Its validity is governed by F-10b760faca36 and F-6bbddce1b930.

### Output

The function returns one 32-bit word:

- Zero on every explicit rejection or unsuccessful-search path.
- Otherwise, a word read through the associated structure of the first matching entry.

The output is not established to be a Boolean, pointer, handle, or owned object. Its interpretation and lifetime are unknown.

### Entry layout

F-6bbddce1b930 supplies an address used as an entry containing these fields:

| Byte offset | Width | Observed use |
|---|---:|---|
| 0 | 32 bits | Passed as the first argument to F-9878699bd825 |
| 4 | 32 bits | Tested for zero after a successful comparison |
| 8 | 32 bits | Passed to F-10b760faca36, then conditionally used as an address |

The layout establishes the fields read by this function, not the entry’s complete size or other contents. The returned entry address must support each read reached on the selected execution path. No check rejects a zero or otherwise invalid entry address before the first field is read.

### Associated structure layout

On a qualifying match, the word at entry offset 8 is used as the base address of an associated structure:

- At byte offset 4 of that structure, the function reads a 32-bit address.
- At byte offset 0 of the referenced storage, it reads the 32-bit return value.

The function does not directly read an associated-structure field at offset 0. In particular, the supplied evidence does not establish that such a field stores a count.

The explicit offsets and word accesses describe a 32-bit address-and-word representation. Byte order, alignment rules, and behavior for invalid addresses are not established.

## Behavior

### 1. Resolve the lookup input

The first observable dependency call is F-83fcd2d25cd8, with the lookup input as its sole argument.

Its result is retained as the comparison value. A zero result causes this function to return zero without invoking either collection dependency or the comparison dependency.

Resolution occurs even when the outer collection argument is zero. Therefore, callers cannot use a zero outer collection to suppress the resolution call or any effects it may have.

If resolution produces a nonzero value but the outer collection is zero, the function returns zero without searching.

### 2. Traverse the outer collection

If both the resolved comparison value and outer collection are nonzero, traversal starts at index zero.

Before attempting each entry, the function calls F-10b760faca36 with the outer collection. It interprets the result as a signed 32-bit bound and compares the current signed index against it.

- If the index is less than the returned bound, traversal proceeds at that index.
- Otherwise, the function returns zero.

The bound is obtained anew for every traversal condition. It is not captured once at the beginning. Changes visible through successive calls can therefore change how many entries are visited.

At index zero, a zero or negative bound ends traversal immediately. No entry-access or comparison call occurs in that case.

For an index that passes the bound check, the function calls F-6bbddce1b930 with the outer collection and current index. It then reads the first 32-bit word at the returned entry address.

The function passes that word and the retained comparison value, in that order, to F-9878699bd825.

### 3. Interpret the comparison result

A comparison result of exactly zero is treated as a match. Any nonzero result is treated as a nonmatch; its sign and magnitude have no further significance.

After a nonmatch, the function increments the index by one and repeats the traversal condition, including another call to F-10b760faca36.

After a match, traversal ends permanently. No later entry is considered, regardless of whether the matching entry ultimately yields a zero result.

For the signed 32-bit bounds shown, the loop does not require signed index overflow: an index can advance to the maximum signed value, but no representable signed bound can then be greater than it.

### 4. Apply the matching-entry gates

After F-9878699bd825 returns zero, the function reads the entry’s word at offset 4.

If that word is nonzero, the function immediately returns zero. It does not inspect the entry’s word at offset 8 or seek another matching entry.

If the offset-4 word is zero, the function reads the word at offset 8 and passes it to F-10b760faca36.

This call is made even when the offset-8 word is zero. There is no preceding null check for this particular invocation.

If the dependency returns exactly zero, the function returns zero. Unlike the outer traversal bound, this result is tested only for equality with zero. A negative result is therefore accepted by this gate, just as a positive result is.

### 5. Select the return word

After the associated-value dependency returns nonzero, the function reads the entry’s offset-8 word again.

If that word is now zero, the function returns zero without dereferencing an associated structure.

If it is nonzero, the function:

1. Reads the 32-bit address at byte offset 4 from that associated-structure base.
2. Reads a 32-bit word at the resulting address.
3. Returns that word unchanged.

There is no explicit null check on the intermediate address read from associated offset 4.

The repeated read of the entry’s offset-8 field matters. The value used for the final address calculation need not equal the value previously passed to F-10b760faca36 if that dependency changes the entry or storage affecting it. The evidence does not establish such mutation, but it also does not establish preservation.

## Side effects

### Direct effects

The function contains no explicit writes to the outer collection, entry, associated structure, or payload storage. It performs no explicit allocation, release, ownership transfer, or global-state update.

It retains traversal and comparison state internally and returns a single word. This does not establish that the function is pure: its dependencies may have effects.

### Observable call and read order

For an entry that reaches final payload extraction, the established sequence is:

1. Resolve the lookup input.
2. Obtain an outer bound.
3. Obtain the entry address for the selected index.
4. Read the entry’s offset-0 word.
5. Invoke the comparison dependency.
6. Read the entry’s offset-4 word.
7. Read the entry’s offset-8 word.
8. Invoke F-10b760faca36 on that word.
9. Read the entry’s offset-8 word again.
10. Read the address at associated offset 4.
11. Read and return the selected payload word.

Earlier nonmatching entries each add a bound call, entry-access call, offset-0 read, and comparison call before this sequence reaches the matching entry.

Fields used only after a match are not read by this function for nonmatching entries. A rejecting offset-4 word also prevents all associated-value reads and calls for that entry.

### Ownership, mutation, and failures

Dependency ownership rules are unknown. The function neither explicitly acquires nor releases referenced storage.

For normal completion, storage must remain readable whenever a listed read is reached. That includes entry storage after comparison and after the associated-value dependency call. No preservation guarantee for that storage is supplied.

There is no explicit error object, exception handler, retry, or recovery path. Invalid reads and dependency failures are not converted by visible logic into a defined zero result. Only the explicit conditions described above establish zero-return behavior.

## Dependencies

| Opaque ID | Invocation and observed role | Contract limits |
|---|---|---|
| F-83fcd2d25cd8 | Called once, first, with the lookup input; zero prevents searching, nonzero supplies the comparison value | Resolution semantics, side effects, and ownership are unknown |
| F-10b760faca36 | Called with the outer collection before every attempted entry; result is a signed bound | Stability, validity requirements, and internal representation are unknown |
| F-6bbddce1b930 | Called with the outer collection and current index; result is used as an entry address | No local validation or lifetime guarantee is established |
| F-9878699bd825 | Called with the entry’s first word and resolved comparison value; zero means match | Equality semantics, mutation, and other side effects are unknown |
| F-10b760faca36 | Also called with the matching entry’s offset-8 word; only zero rejects this gate | This use does not establish a positive-size requirement or safe dereference |

These roles describe how this function consumes dependency results. They do not impose additional behavior on the dependencies.

## Acceptance examples

The following are conditional acceptance criteria derived from the supplied static evidence, not executed tests. Unless stated otherwise, assume all reached reads are valid and dependencies complete normally.

1. **Resolution failure:** If F-83fcd2d25cd8 returns zero, the result is zero and no collection or comparison dependency is called.

2. **Zero outer collection:** If the outer collection is zero, F-83fcd2d25cd8 is still called. The result is zero, and traversal does not begin.

3. **Nonpositive initial bound:** With successful resolution and a nonzero outer collection, an initial outer bound of zero or −1 returns zero without fetching an entry.

4. **No matching entry:** With a stable outer bound of two and nonzero comparison results at indices zero and one, entries are visited in that order. A further bound call occurs at index two before the function returns zero.

5. **First-match rejection:** If index zero matches but its offset-4 word is nonzero, the result is zero. A later matching entry with a usable payload is not visited.

6. **Zero associated argument:** If the matching entry’s offset-4 word is zero and its offset-8 word is zero, F-10b760faca36 is called with zero. If it returns zero, the function immediately returns zero. If it returns nonzero and the field remains zero, the subsequent field check still returns zero.

7. **Negative associated result:** If F-10b760faca36 returns −1 for the associated value, that result passes the nonzero gate. A valid nonzero associated base then leads to payload extraction.

8. **Successful extraction:** If the matching entry passes both gates, its reread associated base is nonzero, and the address stored at associated offset 4 points to the word `0x12345678`, the function returns `0x12345678`.

9. **Zero payload:** Under the same conditions, a selected payload word of zero is returned as zero, without a separate success indicator.

10. **Mutation during the associated call:** If F-10b760faca36 returns nonzero but changes the matching entry’s offset-8 word to zero, the function returns zero and performs no associated-structure dereference.

11. **Changing outer bound:** If the first bound permits index zero, its comparison fails, and the next bound is one, traversal ends at index one without fetching that index.

## Uncertainties

The following remain **documented_only**:

- The semantic types and valid domains of both inputs.
- Whether collection bounds are stable, negative, or affected by dependency calls.
- Whether the offset-4 entry field is a flag, status, pointer, or another kind of value.
- Whether the associated structure is actually a collection, and whether the final read selects its first element.
- Dependency allocation, release, mutation, synchronization, and global-state behavior.
- The validity and lifetime of entry, associated, and payload storage across calls.
- Platform byte order, alignment constraints, and invalid-address behavior.
- Whether zero output is intended as failure, absence, a valid value, or several of these.

Inferred lookup terminology does not resolve these uncertainties.

## Evidence

This contract is based solely on the supplied static function body and dependency-ID mapping.

Established evidence includes the 32-bit word accesses, explicit offsets, signed outer-bound comparison, repeated bound calls, zero/nonzero gates, first-match termination, repeated associated-field read, and final two-stage payload access.

The keyed-lookup interpretation and collection-oriented terminology are inferred intent, not independently established dependency contracts.

No runtime tests have been performed. The acceptance examples specify conditional behavior rather than observed execution results.
