<!-- Trusted provenance: F-7aafa3c67f00; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7aafa3c67f00

## Purpose

F-7aafa3c67f00 conditionally updates a word-based numeric object using an unsigned 32-bit operand.

The established behavior has two paths:

- A direct path modifies backing words and sometimes object metadata, then returns the 32-bit value 1.
- A delegated path clears a metadata field, calls F-2f01ae013cd8 with the object, sets that field to 1, and returns the dependency’s result unchanged.

The direct arithmetic is consistent with subtracting an unsigned word from a nonnegative, little-endian multiword magnitude. A special single-word case stores the positive difference and marks the object when the operand exceeds its nonzero magnitude. These numeric interpretations are inferred intent, not independently established representation guarantees. In particular, the supplied evidence does not establish what the dependency computes.

## Interface

### Inputs

The function receives three arguments:

1. **Object address.** An address of mutable storage containing at least four 32-bit fields.
2. **Operand.** An unsigned 32-bit integer, ranging from 0 through 4,294,967,295.
3. **Additional 32-bit argument.** This argument has no established influence on the result, writes, or dependency selection. On one path its local value is replaced with a backing word before a zero test; elsewhere its incoming value is not used.

The relevant object layout is:

| Byte offset | Width | Established use |
|---|---:|---|
| 0 | 32 bits | Address of backing storage containing 32-bit words |
| 4 | 32 bits | Count-like field, tested against 0 and 1 and sometimes decremented |
| 8 | 32 bits | Not directly read or written by this function |
| 12 | 32 bits | Flag-like field, tested against zero and explicitly assigned 0 or 1 |

The address representation shown is 32 bits. The evidence does not establish portability to wider-pointer layouts.

Backing words are addressed at four-byte intervals, starting at the stored backing address. Index zero is the first word.

### Output

The return value is 32 bits:

- Every direct arithmetic completion returns exactly 1.
- A delegated completion returns the exact 32-bit value produced by F-2f01ae013cd8.

There is no independently established success/error convention for these values. In particular, returning 1 on a direct path does not establish that a dependency result of zero denotes failure.

### Preconditions

For defined, normally completing execution:

- The object address must permit all object accesses reached by the selected path.
- Every backing-word address actually accessed must be valid for the relevant read or write.
- Arithmetic traversal must reach a word large enough to terminate the borrow propagation before leaving valid storage.
- Object and backing storage must remain valid across the dependency call so that the final flag write can occur.

There are no explicit null, capacity, alignment, count-range, or allocation checks in the supplied body. The object’s apparent count does not bound the arithmetic traversal.

The normal numeric interpretation additionally depends on a valid representation, compatible metadata, and no interfering mutation. Those are representation assumptions, not checks performed here.

## Behavior

### 1. Select the direct or delegated path

The function first reads the count-like field at offset 4.

If this field is zero, it proceeds directly to delegation. It does not first inspect the backing word or read the flag.

If the field equals one, the function reads the first backing word and uses its zero/nonzero value for path selection. This read occurs before testing the flag:

- A zero first word selects delegation without a flag read.
- A nonzero first word allows the flag test.

If the count-like field is nonzero and not one, there is no corresponding first-word zero test before the flag test.

For every case that reaches the flag test:

- A zero flag selects direct arithmetic.
- Any nonzero flag selects delegation.

Thus, direct arithmetic is entered only when the initial count-like field is nonzero, the flag is zero, and a count of one does not describe a zero first word.

The incoming additional argument cannot change these decisions.

### 2. Handle a smaller single-word magnitude

After selecting direct arithmetic, a count-like field initially equal to one causes another read of the first backing word.

If that word is unsigned-less-than the operand:

1. Replace the first backing word with operand minus the word just read.
2. Set the flag at offset 12 to 1.
3. Return 1.

This comparison is unsigned. For example, a first word with its high bit set is not treated as negative for the comparison.

Under stable, ordinary storage, the first word on this path is nonzero and strictly smaller than the operand. The stored difference is therefore between 1 and 4,294,967,294. The count-like field is not changed.

The first word is read separately for initial path selection and for this comparison. The evidence does not justify collapsing those into one observable read.

If the word is greater than or equal to the operand, the function continues into the general subtraction path.

### 3. Subtract with borrow propagation

The general direct path starts at backing-word index zero with the original operand as the subtraction amount.

For each examined index, the function reloads the backing address from the object and reads the indexed unsigned 32-bit word.

If the word is smaller than the current subtraction amount:

- It stores word minus subtraction amount in that position, wrapping modulo \(2^{32}\).
- It changes the subtraction amount to 1.
- It advances to the next word.

Consequently, the original operand is used only at index zero. Every subsequent propagated subtraction is a borrow of one.

The loop stops at the first word greater than or equal to the current subtraction amount. There is no explicit comparison between the index and the count-like field.

For the terminal word, the function performs a further read before computing and storing the result:

- It reloads the backing address from the object for the terminal read.
- It subtracts the current subtraction amount from that newly read word.
- It writes the result using the backing address retained from the loop’s terminal comparison.

With ordinary stable storage these addresses and values agree. The distinct accesses matter if storage aliases metadata or is modified externally; the function supplies no synchronization or consistency protection.

The explicitly unsigned borrow updates wrap modulo \(2^{32}\). Under stable storage, the terminal comparison guarantees that its subtraction does not require another borrow.

### 4. Possibly shorten the count

After writing the terminal result, the function tests whether that result is zero.

If it is nonzero, the count-like field is unchanged.

If it is zero, the function:

1. Reads the current count-like field.
2. Forms that value minus one.
3. Stores the decremented value only if it equals the terminal word index.

For an ordinary positive count, this removes exactly one highest counted word when that word has just become zero. It does not scan for additional leading zero words.

A zero result in a lower-index word does not shorten the count.

The function then returns 1. The general subtraction path does not explicitly change the flag.

### 5. Delegate

Every delegated path has this order:

1. Store 0 in the flag at offset 12.
2. Call F-2f01ae013cd8 with the original object address.
3. After a normal return, store 1 in the flag at offset 12.
4. Return the dependency’s 32-bit result unchanged.

The initial flag write occurs even if the flag was already zero.

The post-call flag write is unconditional with respect to the dependency’s return value. A zero or otherwise error-like result does not suppress it.

If the dependency does not return normally, the evidence does not establish that the final write occurs.

## Side effects

### Directly established writes

Depending on the path, this function can write:

- The first backing word in the special single-word case.
- Each backing word traversed by subtraction.
- The count-like field, at most once, when the terminal zero-word condition matches.
- The flag, either once in the special case or twice around delegation.

The general subtraction path performs a terminal word write even when the operand is zero and the stored numeric value is unchanged.

Offset 8 is not directly accessed. This does not guarantee its preservation across a dependency call or through overlapping storage.

### Ownership and lifetime

The function contains no explicit allocation, release, or ownership transfer on its direct paths.

No ownership conclusion follows for F-2f01ae013cd8. The evidence does not establish whether it allocates, frees, replaces backing storage, changes other object fields, or modifies global state.

After delegation returns, this function writes to the original object address. Safe completion therefore requires that address still designate writable object storage.

### Partial updates and errors

There is no explicit rollback or local error return.

Borrow propagation writes each insufficient word before examining the next. An invalid later access can therefore occur after earlier words have already changed.

Similarly, delegation changes the flag before invoking the dependency. There is no demonstrated recovery if that call fails abnormally.

No concurrency, atomicity, or reentrancy guarantees are established.

## Dependencies

### F-2f01ae013cd8

**Invocation condition:** The initial count-like field is zero; or a count of one has a zero first word; or the tested flag is nonzero.

**Argument:** The original object address only.

**Call boundary state:** This function has just set the object flag to zero. It has performed no direct backing-word arithmetic on that invocation.

**Return handling:** The dependency’s 32-bit return value is retained without interpretation. The object flag is then set to one before that value is returned.

**Unknown contract:** Its numerical operation, additional reads and writes, errors, global effects, allocation behavior, and ownership behavior are documented_only. Nothing here assumes that it preserves other object fields.

## Acceptance examples

These examples are conditional on valid, nonoverlapping ordinary storage with no interfering mutation. Words are listed in increasing address order. Unmentioned fields remain unchanged by this function’s direct actions.

### Single-word subtraction without a borrow

Given a count of 1, flag 0, backing words [9], and operand 4:

- The backing word becomes 5.
- The count remains 1.
- The flag remains 0.
- The return value is 1.
- The dependency is not called.

### Single-word subtraction reaching zero

Given a count of 1, flag 0, backing words [9], and operand 9:

- The backing word becomes 0.
- The terminal index is zero, matching count minus one.
- The count becomes 0.
- The flag remains 0.
- The return value is 1.

### Operand exceeding a nonzero single word

Given a count of 1, flag 0, backing words [3], and operand 8:

- The backing word becomes 5.
- The count remains 1.
- The flag becomes 1.
- The return value is 1.

This demonstrates the special difference-and-mark behavior, not general unsigned wraparound subtraction.

### Borrow into the next word

Given a count of 2, flag 0, backing words [0, 2], and operand 1:

- The first word becomes 4,294,967,295.
- The second word becomes 1.
- The count remains 2.
- The flag remains 0.
- The return value is 1.

### Borrow removing one highest word

Given a count of 2, flag 0, backing words [0, 1], and operand 1:

- The words become [4,294,967,295, 0].
- The count becomes 1.
- The flag remains 0.
- The return value is 1.

### No general normalization

Given a count of 3, flag 0, backing words [5, 0, 0], and operand 5:

- The terminal subtraction is at index zero.
- The words become [0, 0, 0].
- The count remains 3 because count minus one is 2, not zero.
- The return value is 1.

The function does not normalize arbitrary leading zeros.

### Zero operand on a multiword object

Given a count of 2, flag 0, backing words [0, 7], and operand 0:

- The first word is read and written as zero.
- No borrow reaches the second word.
- The count remains 2.
- The flag remains 0.
- The return value is 1.

### Delegated zero representation

Given a count of 1 and first word zero, delegation occurs regardless of the operand or initial flag.

If F-2f01ae013cd8 returns a value \(R\) normally and leaves the object writable:

- It is called after the flag has been set to zero.
- The flag is set to one after its return.
- This function returns \(R\).

Other final object contents depend on the dependency.

### Invalid insufficient magnitude

Given a count of 2, flag 0, backing words [0, 0], and operand 1, the function writes both represented words as 4,294,967,295 and continues to index two.

The count does not stop traversal. Without valid accessible storage and an eventual terminating word, safe completion is not established.

## Uncertainties

- **Inferred intent:** The backing words appear to be a base-\(2^{32}\), least-significant-word-first magnitude; the flag appears related to sign. Neither interpretation is independently confirmed.
- **documented_only:** The meaning and legal range of the count-like field beyond its observed comparisons and decrement.
- **documented_only:** The meaning of flag values other than zero and one. All nonzero values select delegation when tested.
- **documented_only:** F-2f01ae013cd8’s actual operation, error semantics, ownership effects, and global effects.
- **documented_only:** Behavior under overlapping object/backing storage, asynchronous mutation, invalid addresses, address overflow, or an index that exceeds the supported arithmetic range. The described access order does not imply such cases are supported.
- **documented_only:** Any caller-level guarantee that inputs are normalized or that direct-path magnitude is sufficient for subtraction.
- **documented_only:** The intended role of the additional argument, whose incoming value has no observed effect here.

## Evidence

The supplied function body statically establishes the field offsets, 32-bit word operations, unsigned comparisons, path conditions, direct return value, dependency call, and ordered flag writes described above.

The numeric representation and sign interpretation are explicitly identified as inference. Dependency behavior and caller invariants are not supplied and are not assumed.

No runtime tests have been performed. Acceptance examples are conditional consequences of the static behavior, not observed execution results.
