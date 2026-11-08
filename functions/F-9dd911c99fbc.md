<!-- Trusted provenance: F-9dd911c99fbc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9dd911c99fbc

## Purpose

F-9dd911c99fbc allocates a block-like region and incorporates its address into a caller-supplied state object. Its subsequent behavior depends on a predicate supplied by F-3538c6888938:

- A nonzero predicate result selects an initialization path.
- A zero result selects an existing-state path, which either advances within the current address table or replaces that table with a larger allocation.

The state contains two four-word records and table-related metadata. The apparent intent is to maintain two iterator-like records over a table of block addresses, with spare table space arranged around the active entries. These semantic roles are inferred. The established behavior is the allocation, dependency-call, address-calculation, and state-write sequence described below.

No runtime tests have been performed. Dependency semantics, including allocation failure behavior and ownership transfer, are not established by the supplied material.

## Interface

**Input:** one pointer to mutable state.

**Output:** no explicit return value or status. Results are observable through state changes, writes to addressed memory, and dependency side effects.

### State layout

The visible operations use 32-bit words and four-byte slot spacing.

| Byte offset | Width | Observed use |
|---|---:|---|
| 0–15 | Four 32-bit words | First record; replaced by output from F-5bb1b38eb3e5 |
| 16–31 | Four 32-bit words | Second record; replaced or copied from the first record |
| 32–35 | One 32-bit word | Not explicitly read or written |
| 36–39 | One 32-bit word | Table base address |
| 40–43 | One 32-bit word | Table capacity-like value |

The words at offsets 12 and 28 also participate directly in table-position arithmetic. Their treatment as table-slot addresses is established by those operations; their meaning within the four-word records is not independently known.

The input must support all explicit accesses through offset 43. Additional accessibility requirements depend on the invoked dependencies and on the pointers stored in the state. No null, alignment, range, or consistency checks are visible.

### Arithmetic conventions

The supplied representation uses 32-bit signed and unsigned quantities. Address arithmetic uses four-byte slot scaling. This contract describes low-32-bit arithmetic where the representation shows unsigned calculations or machine-sized address operations; it does not establish that overflowing signed expressions would be portable in a higher-level implementation.

Two right-shift cases are distinct:

- The existing-state distance calculation right-shifts a signed quantity by two.
- Initialization midpoint calculations right-shift unsigned quantities by one.

The signed shift is consistent with arithmetic right shift on the represented target. Negative-distance behavior and signed overflow are not established as valid input cases.

## Behavior

### 1. Allocate the incoming block before selecting a path

Every invocation begins with this sequence:

1. Call F-dc364b41daca.
2. Shift its 32-bit result left by two.
3. Call F-871e64ddc6de with that shifted value as the first argument and zero as the second argument.
4. Retain the returned value as the incoming block address.
5. Call F-3538c6888938 with the state pointer.

Thus the incoming block allocation precedes the predicate call. The predicate does not control whether this allocation is attempted.

There is no visible check of either the size-producing result or the allocation result. In particular, a zero returned address does not select a special failure path.

The function also forms the address of the second record at byte offset 16. Later record replacements target this location.

### 2. Existing-state path: predicate returns zero

The function reads the word at offset 28 and compares it with:

\[
\text{table base} + 4 \times \text{capacity} - 4.
\]

Equality selects table replacement. Inequality selects an in-place advance. This is an equality test, not a bounds test: a position already beyond the calculated final slot does not, merely for that reason, select replacement.

#### 2.1 Advance without replacing the table

When the equality test fails:

1. Save the previously read position at offset 28.
2. Write that position plus four back to offset 28.
3. Store the incoming block address at the memory location designated by that advanced position.
4. Invoke F-5bb1b38eb3e5 with:
   - a temporary four-word output area;
   - the incoming block address;
   - the current word at offset 28.
5. Copy the four output words into the second record, in ascending offset order: 16, 20, 24, then 28.

The write at offset 28 before the block-address store is observable. The final record copy can replace that same word with a different value returned by F-5bb1b38eb3e5.

This path does not explicitly replace the first record, table base, or capacity. That restriction applies only to direct writes by this function; it is not a promise that dependencies leave those locations unchanged.

#### 2.2 Replace the table at its final slot

When the equality test succeeds, the function derives a signed slot distance from the previously read position and the current word at offset 12:

\[
d = (\text{position} - \text{word at offset 12}) \mathbin{\text{signed-shift-right}} 2.
\]

It then forms a 32-bit unsigned count:

\[
n = d + 1.
\]

For ordinary aligned, nonwrapping addresses with a nonnegative difference, \(n\) is the number of slots in the inclusive range between those positions. This interpretation is conditional, not a validation performed by the function.

The replacement sequence is:

1. Call F-871e64ddc6de with \(8n\) as its first argument and zero as its second.
2. Retain that result as the replacement allocation address.
3. Calculate a byte displacement:
   \[
   p = (2n) \mathbin{\&} \texttt{0xFFFFFFFC}.
   \]
4. Call F-8a47fbc36db6 with:
   - the current word at offset 12;
   - the current word at offset 28 plus four;
   - replacement allocation address plus \(p\).
5. Call F-ff1bdc376dcd with:
   - the current table base;
   - the current capacity;
   - zero.
6. Write the retained replacement allocation address into the table-base field at offset 36.
7. Store the incoming block address at:
   \[
   \text{replacement allocation address}
   +4\left(d+\left((n\mathbin{\&}\texttt{0x7FFFFFFF})\mathbin{\text{unsigned-shift-right}}1\right)\right)
   +4.
   \]
8. Invoke F-5bb1b38eb3e5 using a temporary four-word output area, the current first word of the state, and the current table base plus \(p\).
9. Copy its output into the first record, in ascending offset order: 0, 4, 8, then 12.
10. Invoke F-5bb1b38eb3e5 again, using the temporary output area, the incoming block address, and:
    \[
    \text{current table base}+p+4d+4.
    \]
11. Copy that output into the second record, in ascending offset order: 16, 20, 24, then 28.
12. Write \(2n\), reduced to 32 bits, into the capacity field at offset 40.

The apparent copy-like operation occurs before the release-like operation. The table-base field is changed only after both calls return. The capacity field is updated last, after both record reconstructions.

The block-address store uses the retained allocation result, whereas the record-construction calls use the table base reread from the state. These are not interchangeable if an intervening dependency modifies state.

For ordinary small counts, \(p=4\lfloor n/2\rfloor\). That simplification must not replace the stated 32-bit expression for unrestricted inputs. The high-bit masking in the block-address store also matters for large counts.

### 3. Initialization path: predicate returns nonzero

This path performs two additional, separate calls to F-dc364b41daca. The function does not require their results to match each other or the result used for the initial block allocation.

The sequence is:

1. Call F-dc364b41daca and write its result to the capacity field at offset 40.
2. Call F-54841aae8aad with:
   - a temporary four-byte area;
   - the just-obtained capacity value;
   - the current table-base field.
3. Write its returned value to the table-base field.
4. Reread the capacity as unsigned, divide it by two by right shift, and store the incoming block address at:
   \[
   \text{returned table address}
   +4\left\lfloor\frac{\text{current unsigned capacity}}{2}\right\rfloor.
   \]
5. Call F-dc364b41daca again.
6. Invoke F-5bb1b38eb3e5 with:
   - a temporary four-word output area;
   - incoming block address plus four times the unsigned half of this latest result;
   - current table base plus four times the unsigned half of the current capacity.
7. Copy the four output words into the first record in ascending offset order.
8. Copy the first record into the second record, word by word in ascending offset order.

Consequently, the second record equals the first record at the end of this path, provided all preceding operations return normally. No dependency calls intervene between the two record writes.

The incoming block address is stored in the table, but the address supplied as the second argument to F-5bb1b38eb3e5 may point inside that block rather than to its beginning. The offset uses the final size-producing call, not the initial allocation-size result.

## Side effects

### State and memory writes

The function can directly write:

- Either or both four-word records.
- The table-base and capacity fields.
- One table slot containing the incoming block address.
- Temporary stack storage supplied to dependencies.

It does not explicitly access the word at offset 32.

The function passes the entire state pointer to F-3538c6888938. Other dependencies receive values read from the state, memory addresses, or temporary output storage. Their additional reads and writes are unknown.

### Ownership and lifetime

The calls and subsequent uses suggest that F-871e64ddc6de provides storage, F-54841aae8aad obtains table storage, F-8a47fbc36db6 transfers table entries, and F-ff1bdc376dcd releases old table storage. These are inferred roles, not established ownership contracts.

The visible function:

- Does not explicitly release the incoming block.
- Invokes F-ff1bdc376dcd only on the table-replacement path.
- Does not inspect the temporary four-byte area passed to F-54841aae8aad.
- Copies the temporary four-word output of F-5bb1b38eb3e5 into persistent state.

It is unknown whether dependencies retain pointers, release storage, transfer ownership, or alter shared globals. No preservation guarantees should be inferred from a dependency’s apparent role.

### Errors and partial updates

There is no explicit error return, allocation-result validation, exception handling, or rollback.

Writes occur incrementally. For example:

- An in-place advance publishes the advanced position before record construction.
- Replacement may invoke the release-like dependency before publishing the replacement base.
- Initialization publishes capacity before obtaining and publishing its table address.
- Replacement publishes capacity only after both records have been reconstructed.

If a dependency fails, does not return, or faults, the function provides no visible mechanism to restore earlier state. The actual failure behavior remains dependency- and platform-dependent.

## Dependencies

| Opaque ID | Established use |
|---|---|
| F-dc364b41daca | Produces 32-bit values used for the incoming allocation size and, on initialization, capacity and an interior-block offset |
| F-871e64ddc6de | Called with a byte-count-like value and zero; its result is used as an address |
| F-3538c6888938 | Receives the state pointer; zero selects existing-state processing and nonzero selects initialization |
| F-54841aae8aad | Receives temporary storage, a capacity-like value, and an existing table-base value; its result becomes the table base |
| F-8a47fbc36db6 | Receives two source-side addresses and a replacement destination address |
| F-ff1bdc376dcd | Receives the old table-base and capacity values with a trailing zero |
| F-5bb1b38eb3e5 | Receives a four-word output area and two address-like values; its output replaces a record |

Dependency returns and writes are consumed without visible validation. Argument-evaluation ordering within an individual call is not independently established; the inter-call and explicit-write ordering above is established.

## Acceptance examples

These examples are conditional static expectations, not runtime results. Assume dependencies return normally, provide valid storage and fully initialized record output where required, and do not unexpectedly modify the state.

### Initialization with distinct size results

Suppose:

- The initial F-dc364b41daca result is 8.
- The predicate returns nonzero.
- The next size result is 10.
- F-54841aae8aad returns a valid table address.
- The final size result is 6.

Expected behavior:

- The first allocation request is 32 bytes.
- The capacity becomes 10 before F-54841aae8aad is invoked.
- The incoming block address is stored 20 bytes from the returned table base.
- F-5bb1b38eb3e5 receives an address 12 bytes into the incoming block and a table-slot address 20 bytes from the current table base.
- Both records finish with the four-word result of that call.

This example distinguishes the three independent size results.

### Existing table with an available next slot

Suppose the predicate returns zero and the position at offset 28 is not equal to the calculated final-slot address.

Expected behavior:

- Offset 28 is first advanced by four.
- The incoming block address is stored at that advanced address.
- F-5bb1b38eb3e5 is called once.
- Its output replaces the second record only.
- No table-replacement allocation or release-like call occurs.

A record output containing a different fourth word must become the final value at offset 28; the earlier advanced value is not preserved unconditionally.

### Replacement with three active slots

Suppose valid aligned state yields \(d=2\), hence \(n=3\), and the current position equals the final-slot address.

Expected behavior:

- Replacement allocation requests 24 bytes.
- The destination passed to F-8a47fbc36db6 is replacement base plus four.
- The incoming block address is stored at replacement base plus 16.
- The first record reconstruction uses replacement base plus four.
- The second record reconstruction uses replacement base plus 16.
- Capacity becomes 6 only after both records are written.

### Equality is the sole replacement trigger

Suppose the predicate returns zero and the current position is numerically beyond, but unequal to, the calculated final-slot address.

The visible branch selects in-place advancement. This is not a guarantee of safe access: such state may violate necessary validity preconditions. An implementation that adds a greater-than comparison would change the observed branch rule.

## Uncertainties

The following remain **documented_only**:

- The abstract data structure and public meaning of its records.
- Whether the predicate actually reports emptiness.
- Valid capacity ranges, alignment requirements, and block-size constraints.
- Behavior for negative slot distances, signed overflow, wrapped addresses, zero capacities, or insufficient storage.
- Allocation failure conventions and whether address zero is ever valid.
- Whether F-8a47fbc36db6 uses an exclusive source endpoint, supports overlap, or performs transformations.
- Whether F-ff1bdc376dcd releases memory and how its capacity argument is interpreted.
- Whether F-54841aae8aad reuses or releases the supplied prior table value.
- Whether F-5bb1b38eb3e5 always initializes all four output words.
- Dependency effects on globals, aliased state, and storage ownership.
- Concurrency, reentrancy, and synchronization requirements.

A safe consumer needs valid writable state and valid referenced storage for the selected path. The supplied evidence does not establish sufficient numeric invariants to guarantee those conditions for arbitrary inputs.

## Evidence

**Established static evidence:** the supplied function representation shows the 32-bit field accesses, dependency identities and arguments, branch conditions, arithmetic, record-copy order, and absence of visible checks described above.

**Inferred intent:** block allocation, centered table initialization, active-range relocation, table release, and iterator-like record reconstruction are interpretations of those operations. They are not substitutes for dependency contracts.

**Verification limit:** no runtime execution, tests, dependency implementations, or independent layout declarations were available. Acceptance examples specify conditional behavior derived from static evidence; they do not report observed execution.
