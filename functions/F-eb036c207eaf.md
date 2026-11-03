<!-- Trusted provenance: F-eb036c207eaf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-eb036c207eaf

## Purpose

F-eb036c207eaf sets a selected bit in word-addressed storage, extending the descriptor’s logical extent when necessary. For an existing logical word, its direct update preserves the word’s other bits. For an extension, it clears a range determined by the descriptor’s logical count **after** any dependency call, assigns the requested logical count, and then performs the bit update.

Extension does not unconditionally produce a target word containing only the requested bit. That result requires the clearing range to include the target word. If the dependency advances the logical count beyond the target, clearing is skipped and the final update preserves any other bits present in that word at update time.

Preservation by this routine’s direct operations is distinct from preservation across a dependency call. The dependency’s effects on existing contents, descriptor fields, and other state are unknown.

The routine returns a 32-bit success indicator. With a valid descriptor, its only explicit failure return follows a null dependency result. It does not validate the position, descriptor, allocation bounds, or usable capacity after a nonnull dependency result.

This is a static behavioral contract. No runtime tests have been performed.

## Interface

### Inputs and layout

**Storage descriptor:** A pointer to a descriptor containing three consecutive 32-bit fields:

| Byte offset | Width | Observed role |
|---|---:|---|
| 0 | 32 bits | Backing-storage address representation |
| 4 | 32 bits | Logical word count, compared as signed |
| 8 | 32 bits | Capacity-like word count, compared as signed |

Backing elements occupy four bytes and are accessed as 32-bit words. Word index zero identifies the first word at the represented backing address.

This layout establishes a 32-bit address representation, not a portable native-pointer structure for wider address spaces. Byte order and alignment requirements are not established.

**Bit position:** A signed 32-bit integer.

For nonnegative positions, division by 32 selects the word and the remainder selects the bit within that word. Positions 0–31 select word zero; positions 32–63 select word one.

### Output

The result is a 32-bit value:

- **1:** Execution reached the success return after the selected-word update.
- **0:** On an extension requiring the dependency, that dependency returned null and this routine returned before its subsequent clearing, count assignment, and bit update.

These are the only explicit result values. Invalid accesses and unresolved shift semantics have no separately represented error result.

A result of 1 does not independently certify valid allocation bounds, internally consistent metadata, or dependency correctness.

### Preconditions for ordinary bit-storage behavior

The ordinary interpretation requires:

- A valid descriptor supporting the represented reads and writes.
- Writable backing storage covering every word actually accessed.
- A nonnegative position.
- Metadata consistent with the storage available before and after any dependency call.
- A nonnull dependency result accompanied by usable state in the **original descriptor**, not merely in a separately returned object.
- No descriptor/backing overlap that makes the routine’s stores alter subsequent metadata or address reads.
- No concurrent mutation of the relevant storage.

These conditions are not checked comprehensively by the routine.

They do **not** require the dependency to preserve the logical count or existing word contents. Any guarantee needing such preservation must state it separately. In particular, usable post-call storage alone does not ensure that the target word is cleared.

## Behavior

### Position mapping and integer widths

The calculated word index is signed division of the position by 32 with truncation toward zero.

| Position | Calculated word index |
|---:|---:|
| 0 | 0 |
| 31 | 0 |
| 32 | 1 |
| −1 | 0 |
| −31 | 0 |
| −32 | −1 |
| −33 | −1 |

Across the signed 32-bit input range, the word index ranges from −67,108,864 through 67,108,863. The requested logical count is that index plus one, ranging from −67,108,863 through 67,108,864. This addition does not overflow a signed 32-bit integer.

For nonnegative positions, the selected bit number is between 0 and 31. Under the conventional 32-bit mask interpretation, the final operation sets that bit using bitwise OR.

Negative positions are not rejected. Their remainder and shift behavior require separate qualification under **Uncertainties**; truncation toward zero does not make them valid storage positions.

Word addresses use four-byte scaling. Representability of a word index or count does not establish that the corresponding address is valid.

### Request within the current logical extent

If the initial logical count is strictly greater than the calculated word index:

- The capacity-like field is not consulted for extension.
- F-08bc2fc94856 is not called.
- No clearing range is processed.
- The logical count is not assigned.
- The selected word’s existing value is combined with the bit mask and written back.
- Execution returns 1.

For an ordinary valid input, this direct operation preserves every other bit in the selected word. Other backing words are not directly written.

An already-set bit does not cause an explicit early return. The represented operation still includes a write to the selected word.

### Request requiring logical extension

If the initial logical count is less than or equal to the calculated word index, the requested count is the word index plus one.

The capacity-like field determines whether the dependency is called:

- Capacity at least equal to the requested count: continue without a dependency call.
- Capacity smaller than the requested count: call F-08bc2fc94856 with the original descriptor and requested count.

A null dependency result causes return value 0. A nonnull result permits continuation without a second capacity check.

On continuation, the routine obtains the clearing start from the **original descriptor’s current logical count**. If a dependency call occurred, this is its post-call value. It is not a saved copy of the initial count.

The clearing range begins at that current count and ends immediately before the requested count. Each word in the range is written as zero, in ascending index order. If the current count is already equal to or greater than the requested count, there are no clearing stores.

The routine then assigns the requested count to the original descriptor’s logical-count field, performs the selected-word OR update, and returns 1.

### Conditions on clearing and preservation

With valid nonoverlapping storage, the target word lies in the clearing range exactly when the clearing start is less than or equal to the target index. In that case, the target is zeroed before the final OR and ends with only the requested bit set, subject to the mask-semantics qualification.

If the clearing start exceeds the target index, the target is not cleared. Its final value is its value at the final read, with the requested bit set. Other bits can remain set.

For an extension that does not call the dependency, ordinary consistent metadata makes the clearing start the initial logical count. The newly exposed range is therefore cleared, and previously active words are not directly modified.

For an extension that calls the dependency:

- The clearing range may differ from the initially inactive range.
- The dependency may already have changed previously active words.
- The dependency may have changed the target word.
- A lowered post-call count can cause this routine to clear words that were active before the call.
- A post-call count above the requested count causes no clearing, followed by a reduction of the logical count to the requested count.

Consequently, neither preservation of all initially active contents nor zeroing of all initially inactive words is guaranteed across an unknown dependency call.

### Dependency result handling

The returned pointer is used only as a null/non-null continuation test. A nonnull result does not replace the original descriptor for later accesses.

A dependency that returns a usable replacement descriptor but leaves the original descriptor unusable does not satisfy this caller’s storage requirements.

The routine does not save and restore the original backing address, logical count, capacity, or backing contents. Later accesses use the original descriptor as it then exists.

## Side effects

### Direct memory effects and ordering

The established ordering is:

1. Read the logical count to determine whether extension handling is needed.
2. On the extension path, consult capacity before deciding whether to call the dependency.
3. If called, complete the dependency invocation before subsequent clearing.
4. Obtain the clearing start from the original descriptor’s then-current logical count.
5. Perform clearing stores in ascending word-index order.
6. Assign the requested logical count after the clearing range has been processed.
7. Read and write the selected word for the final OR update.
8. Return success after that update.

Without extension, only the applicable descriptor/address reads and selected-word update are performed.

The routine does not directly assign the backing-address field or capacity field. The dependency may change either.

The final word value depends on the word read for that update, not necessarily on its value before the function was entered. This distinction matters when a dependency changes backing contents or storage location.

The static representation does not establish a complete instruction-level trace, including the exact number and ordering of backing-address reloads. These ordering guarantees do not establish atomicity or synchronization.

### Failure and partial effects

A null dependency result prevents this routine’s subsequent clearing, logical-count assignment, and bit update. It does not guarantee an unchanged descriptor or unchanged memory: the dependency may mutate state before returning null.

There is no rollback.

An invalid access during clearing or the final update has no represented recovery path. Earlier stores may already have occurred. The logical count is assigned before the final update, so a failure during that update may leave the new count recorded without a completed bit write.

### Ownership

No direct allocation, deallocation, release, or ownership transfer is established for F-eb036c207eaf.

The dependency’s returned pointer is neither installed as the active descriptor nor explicitly released here. Its ownership meaning is unknown. No assumption that it is newly allocated, borrowed, or independently releasable is justified.

## Dependencies

### F-08bc2fc94856

**Invocation condition:** The request requires logical extension, and signed capacity is smaller than the requested count.

**Arguments:**

- The original descriptor pointer.
- The requested word count, equal to the target word index plus one.

**Result use:**

- Null: return 0 without this routine’s subsequent stores.
- Nonnull: continue using the original descriptor.

**Inferred intent:** Make enough backing storage available for the requested number of words, potentially by updating the original descriptor in place.

**Established caller behavior:** No post-call capacity validation, no replacement of the original descriptor with the returned pointer, and no restoration of pre-call metadata or contents.

**Unknown dependency behavior:** Allocation policy, growth policy, content preservation, initialization, logical-count changes, capacity changes, ownership, global-state effects, and failure atomicity.

Usable post-call storage is a precondition for ordinary execution, not a property proven by the nonnull result.

## Acceptance examples

These are conditional acceptance cases derived from static evidence, not observed test results. Unless stated otherwise, assume valid nonoverlapping storage, no concurrent mutation, nonnegative positions, and conventional 32-bit bit-mask behavior.

### Existing logical word

Given logical count 2, word 1 equal to `0x00000005`, and position 33:

- No dependency call or clearing occurs.
- Word 1 becomes `0x00000007`.
- The logical count remains 2.
- Word 0 is not directly modified.
- The result is 1.

### Bit already set

Given logical count 1, word 0 equal to `0x00000008`, and position 3:

- Word 0 remains numerically `0x00000008`.
- The represented update still writes the word.
- No extension occurs.
- The result is 1.

### Extension within capacity

Given logical count 1, capacity 4, and position 70:

- The target is word 2, bit 6; the requested count is 3.
- No dependency call occurs.
- Words 1 and 2 are zeroed in that order.
- The logical count becomes 3.
- Word 2 then becomes `0x00000040`.
- Word 0 is not directly modified.
- The result is 1.

### Capacity equality

Given logical count 1, capacity 2, and position 32:

- Capacity equality does not trigger the dependency.
- Word 1 is cleared.
- The logical count becomes 2.
- Word 1 becomes `0x00000001`.
- The result is 1.

### Dependency advances count past the target

Given initial logical count 1, capacity 1, and position 70, suppose F-08bc2fc94856 returns nonnull after leaving the original descriptor with logical count 3, capacity 3, usable storage, and word 2 equal to `0x00000080`:

- The requested count is 3.
- The post-call clearing start is 3, so no words are cleared.
- The logical count is assigned 3.
- Word 2 becomes `0x000000C0`, not `0x00000040`.
- The result is 1.

This case satisfies the usable-storage preconditions without requiring preservation of the initial logical count or zero initialization.

### Dependency leaves the target in the clearing range

For the same position, suppose the dependency instead leaves logical count 2 and usable storage through word 2:

- Clearing begins at word 2.
- Word 2 is zeroed regardless of its post-call contents.
- The logical count becomes 3.
- Word 2 becomes `0x00000040`.

No claim is made about preservation of other words across the dependency.

### Dependency leaves a count above the request

For a requested count of 3, suppose the dependency leaves logical count 5 and valid target storage:

- No clearing stores occur.
- The logical count is reduced to 3.
- The target retains its other bits while the requested bit is set.

### Dependency returns null

Given logical count 1, capacity 1, and position 32:

- The dependency receives requested count 2.
- If it returns null, the result is 0.
- This routine performs no subsequent clearing, count assignment, or bit update.
- Dependency-induced changes are not undone.

### Empty logical extent

Given logical count 0, capacity 1, and position 0:

- No dependency call occurs.
- Word 0 is cleared.
- The logical count becomes 1.
- Word 0 becomes `0x00000001`.
- The result is 1.

### Negative exact-multiple boundary

Position −32 calculates word index −1 and remainder zero. With a nonnegative logical count, extension is bypassed and the selected address is one word before the backing base.

This demonstrates unchecked addressing, not a supported negative-index convention.

## Uncertainties

- **Negative-position shifts — documented_only.** Negative positions not divisible by 32 produce negative signed remainders. Restricting those remainders to eight bits gives shift counts from 225 through 255. Executable handling of those counts is not established; no portable bit result is promised.
- **Highest-bit mask — documented_only.** Positions such as 31 require bit 31. The signed-literal shift representation leaves language-level signed-shift concerns unresolved. Conventional machine behavior is a condition of the corresponding examples, not runtime evidence.
- **Address arithmetic — documented_only.** Four-byte scaling and address formation do not prove allocation coverage, valid pointer provenance, or behavior at address-space boundaries. Malformed negative counts can also expose arithmetic outside ordinary storage assumptions.
- **Malformed metadata — documented_only.** Negative counts and inconsistent capacities are not rejected. They can produce unexpected clearing ranges or invalid accesses.
- **Dependency effects — documented_only.** Metadata, contents, ownership, allocation, and global-state effects remain unknown. A nonnull result does not establish their preservation.
- **Aliasing and access traces — documented_only.** Descriptor/backing overlap can make stores affect later accesses. Exact generated load ordering is unavailable.
- **Execution environment — documented_only.** Endianness, alignment faults, memory protection, atomicity, and fault handling are not established.

## Evidence

**Established static evidence:** Three 32-bit descriptor fields; four-byte word addressing; signed extent comparisons; truncation-toward-zero indexing; conditional dependency invocation; null-result failure; continued use of the original descriptor; post-call clearing-start selection; ascending zero stores; requested-count assignment before the final OR; and explicit results 0 and 1.

The clearing bounds establish that a post-call count equal to the requested count skips clearing. They therefore permit an existing target value of `0x00000080` to become `0x000000C0` when bit 6 is selected.

**Inferred intent:** Growable bit storage with a capacity-provisioning dependency.

**Evidence limits:** Only the supplied static representation was available. No dependency implementation, runtime execution, tests, allocation traces, or instruction-level validation was available. Acceptance examples are conditional requirements, not recorded executions.
