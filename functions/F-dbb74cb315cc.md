<!-- Trusted provenance: F-dbb74cb315cc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dbb74cb315cc

## Purpose

F-dbb74cb315cc conditionally stores a caller-supplied object address in a shared, four-position table and writes an index-derived marker into the object.

A byte at object offset 16 gates the operation. Any nonzero value causes an immediate return. Otherwise, the function examines indices 0 through 3 in order, stopping at the first index whose comparison reads zero. It then evaluates the shared table-base expression again, stores the supplied address at that index relative to the newly obtained base, writes the corresponding marker into the object, and returns.

**The successful comparison and subsequent store use separate evaluations of the shared table-base expression.** Without a stable-base precondition, they need not access the same location. Selection establishes an index; it does not preserve the address of the location observed to contain zero.

If all four comparisons read nonzero, the function returns without an explicit write.

“Registration,” “object,” and “marker” describe inferred intent. The established behavior is an ordered sequence of memory accesses; no application-level registry invariant is independently established.

## Interface

### Input and object layout

The function accepts one integer-valued address. The supplied representation uses that value both to address memory and as the value written into the table.

The only directly accessed object field is:

| Byte offset from supplied address | Access | Meaning within this function |
|---|---|---|
| 16 | One-byte read; one-byte write on the insertion path | Zero permits the search; nonzero suppresses it |

The function performs no null check, address validation, object-type check, or bounds check.

The representation declares an integer argument but does not independently establish the original ABI, argument width, pointer width, or all integer-to-address conversion rules. Ordinary execution requires the supplied value and the offset-16 address calculation to be valid under the target’s actual model.

### Shared table layout

A shared expression supplies the table base. The search uses these indices and byte displacements:

| Index | Displacement from the base evaluated for that access | Empty comparison value | Marker expression |
|---|---:|---:|---|
| 0 | 0 | Integer zero | `A` |
| 1 | 4 | Integer zero | `A` plus 1 |
| 2 | 8 | Integer zero | `A` plus 2 |
| 3 | 12 | Integer zero | `A` plus 3 |

Each table access is represented as an integer access. Four-byte spacing is explicit; independent confirmation of the original target’s integer access width is not supplied. A conventional four-byte integer layout makes these four adjacent, non-overlapping slots, but spacing alone does not establish that ABI fact.

Every nonzero comparison result counts as occupied. The function does not check whether the value is a valid address or whether it equals the input.

### Output

There is no explicit return value or status result.

Under valid, non-overlapping storage and a stable table base, the observable outcomes are:

- Initial marker nonzero: no table access and no explicit write.
- Initial marker zero, with an empty position found: the input value is stored at that position, followed by the corresponding marker write.
- Initial marker zero, with all four positions occupied: no explicit write.

Without base stability, the insertion-path store is only guaranteed to use the selected **index** relative to the base evaluated for that store. It is not guaranteed to update the location that passed the zero comparison.

### Preconditions

Ordinary successful execution requires:

- A readable byte at object offset 16.
- For a zero initial marker, a valid shared table-base evaluation and readable integer location for every comparison reached.
- On the insertion path, a valid fresh table-base evaluation, a writable integer location at the selected displacement from that base, and a writable object marker byte.
- Valid address arithmetic and suitable access alignment under the actual target model.

These conditions are not checked.

A stable base, non-overlapping storage, and absence of external mutation are additional preconditions for the ordinary “fill an empty slot and mark the object” postcondition. They are not enforced by the function.

## Behavior

### Marker gate

The first established memory access reads the byte at object offset 16.

If the byte is nonzero, the function returns immediately. It does not evaluate the table-base expression, inspect table entries, or validate the marker’s meaning.

Any nonzero byte suppresses the search, including values unrelated to the markers produced by this function. This branch does not verify that the input address appears in any table.

If the byte is zero, the search begins at index 0. The marker is not checked again before the later write.

### Ordered comparisons

For each reached index, in ascending order:

- The shared table-base expression is evaluated for that comparison.
- An integer is read at the resulting base plus four times the current index.
- A nonzero result advances the search.
- A zero result selects that index and enters the insertion path.

The possible comparison displacements are exactly 0, 4, 8, and 12 bytes. The bounded index arithmetic does not require counter wraparound or a large-index conversion. Validity of the complete address calculation still depends on the base value and target arithmetic model.

The base expression is evaluated separately for each comparison. Therefore, if it changes during the search, the four possible reads may refer to different tables. “First empty position” means the first zero **observed in this ordered sequence**, not necessarily the first empty position in any single table snapshot.

No comparison is made at index 4 or beyond.

### Insertion-path store and marker write

After a comparison reads zero at index *k*, the source-level sequence is:

1. Evaluate the shared table-base expression again for the integer store.
2. Store the input value at displacement four times *k* from that newly obtained base.
3. Write the index-derived marker into the byte at object offset 16.
4. Return.

The store does not reuse a preserved comparison address. There is no second zero check at the store destination.

Consequently, a changed base can direct the store to a location whose prior contents were never read. That location may already contain a nonzero value. The function does not detect this distinction or prevent the overwrite.

Under a stable base, the comparison and store address the same location, so the store fills the position observed to be empty. References to a “selected entry” have this stronger meaning only under that precondition.

The marker depends on the selected index, not on the base used for either access or the previous contents of the store destination. The expression uses the character value for `A` plus an index from 0 through 3. In an ASCII-compatible character set, the resulting bytes are 0x41 through 0x44, conventionally displayed as `A` through `D`. The supplied evidence does not separately establish the target character encoding.

The table store precedes the marker write. No further table-base evaluation is shown after that store. A write that aliases the shared base storage may change that storage, but the function does not subsequently consult it on this path.

The two writes are not an established atomic transaction. No rollback is shown if the table store completes but the marker write cannot complete.

### Exhausted search

If all four comparisons read nonzero, the function returns without an explicit store to the table or object.

With a stable base, this means the four examined positions of one table were occupied. With a changing base, it means only that each comparison read nonzero at the address computed for that comparison.

There is no eviction, replacement policy, retry, allocation, or capacity expansion.

### Duplicate values and repeated calls

The search tests only for zero. It does not search for an existing copy of the input value.

A zero-marked object may therefore acquire another table occurrence even if its address is already present. Conversely, a nonzero-marked object is not reinserted or repaired if another component has removed its table entry.

## Side effects

### Direct access order

The following describes source-level logical accesses, not guaranteed machine load counts:

| Path | Established ordered accesses |
|---|---|
| Initial marker nonzero | Read marker; return |
| Zero observed at index *k* | Read marker; for each index 0 through *k*, evaluate base and read comparison location; evaluate base again and store at index *k*; write marker; return |
| Four nonzero comparisons | Read marker; evaluate base and read comparison location separately for indices 0 through 3; return |

For example, success at index 2 includes three comparison-related base evaluations and one additional store-related evaluation. It does not imply four distinct machine instructions or volatile loads.

No direct write clears the marker, removes an entry, compacts a table, or changes another object field.

### Aliasing

The function does not reject overlap among the object, table-access locations, and shared base storage.

If these regions overlap:

- The integer store may modify the marker before the explicit byte write.
- The marker write may modify bytes of the stored integer.
- Either write may modify shared base storage.
- A final table value may differ from the input value because the later marker write overlaps it.

The established write order still matters, but a persistent input-valued table entry is not an unconditional postcondition. Exact overlap effects depend on access widths, byte order, layout, and valid execution under the target memory model.

### Ownership and lifetime

The function stores an address value; it does not copy the object’s contents.

No direct function calls, allocation, release, reference-count adjustment, or lifetime extension appear. No ownership transfer or later cleanup policy is established. A completed store does not prove that the referenced object remains alive for subsequent users.

### Concurrency and failures

No synchronization, atomic reservation, or memory barrier is explicit.

Concurrent callers may observe the same zero location. External changes may also alter the marker, entry contents, or table base between accesses. The source-level reevaluation rule does not establish what a compiler or processor guarantees during an unsynchronized race.

There is no explicit error status, exception handling, or recovery path. Invalid memory, invalid address arithmetic, or unsuitable alignment falls outside the successful-execution contract; the resulting failure behavior is undocumented.

## Dependencies

No called-function dependency is present in the supplied body. No additional opaque function ID is required.

Direct dependencies are:

- The input address value.
- The byte at input offset 16.
- Shared state supplying the table base on each evaluation.
- Integer-access locations at the reached displacements.
- The target’s arithmetic, integer, character, alignment, and memory-access rules.

The table’s initialization, ownership, lifetime, replacement policy, and external synchronization are not established. No behavior of an unknown surrounding component is assumed.

## Acceptance examples

These are conditional criteria derived from static evidence, not executed tests.

Unless a case explicitly varies the base, assume valid, non-overlapping storage, a stable base, no external mutation, and representable address calculations.

### Nonzero marker

Given any nonzero initial marker:

- Return after the marker read.
- Do not evaluate the table-base expression.
- Perform no explicit write.

This holds even if the marker is not one of the four intended letters and the input address is absent from the table.

### First position empty

Given a zero marker and zero at index 0:

- Evaluate the base for the index-0 comparison.
- After observing zero, evaluate the base again for the store.
- Store the input at index 0, then write the marker corresponding to `A`.
- Do not compare indices 1 through 3.

The stable-base precondition makes the store destination equal the comparison location.

### Later position empty

Given a zero marker, nonzero values at indices 0 and 1, and zero at index 2:

- Compare indices 0, 1, and 2 using separate base evaluations.
- Evaluate the base again for the index-2 store.
- Store the input, then write the marker corresponding to `C`.
- Do not compare index 3.

An empty index 3 would not change the choice.

### Last position empty or table full

Given nonzero values at indices 0 through 2:

- If index 3 reads zero, perform the fresh-base index-3 store, then write the marker corresponding to `D`.
- If index 3 reads nonzero, return without an explicit write; the marker remains zero under the stated assumptions.

### Base changes between comparison and store

Given valid storage, a zero marker, and a comparison that reads zero at index 1 relative to one base, suppose the next base evaluation yields a different base:

- Store the input at index 1 relative to the second base.
- Do not recheck that destination for zero.
- Write the marker corresponding to `B`.
- Do not require the original zero location to receive the input.

If the second destination contained a nonzero value, this path overwrites it. This example describes the source-level evaluation rule; it does not assert a defined concurrent execution mechanism.

### Base changes between comparisons

Given that the index-0 comparison uses one base and reads nonzero, while the index-1 comparison uses another base and reads zero:

- Select index 1.
- Evaluate the base yet again for the store.
- Derive the marker from index 1, regardless of which base that final evaluation supplies.

The observation does not establish that either table has index 1 as its first empty position.

### Duplicate and repeated invocation

Given a zero marker, an earlier occupied position containing the input, and a later zero position, the existing occurrence is treated as occupied and the later position is selected.

After ordinary successful insertion, another call with the marker unchanged returns at the marker gate. It does not inspect or repair the table.

## Uncertainties

The following remain **documented_only**:

- Original ABI, argument and pointer widths, integer access width, and complete conversion rules. Four-byte table spacing is established.
- Target character encoding and the exact numeric values of the four marker expressions.
- Whether the table base is stable, externally synchronized, or replaceable.
- Whether external actors can change memory between the established accesses.
- Permitted overlap and the target-specific results of overlapping accesses.
- Whether address zero can designate accessible storage; no null check is present.
- Ownership and lifetime obligations for the object and table.
- Application-level meaning of the stored address and marker.
- Machine instruction ordering, atomicity, compiler transformations, and concurrent-execution guarantees.
- Failure behavior for invalid addresses, alignment, or arithmetic.

These unknowns do not obscure the established comparison-to-store reevaluation: the source evaluates the base separately for both accesses.

## Evidence

Established static evidence is limited to the supplied representation of F-dbb74cb315cc:

- One integer-valued input supplies both an object address and a stored value.
- A byte read at offset 16 gates all table activity.
- The search reaches only indices 0 through 3, with four-byte displacement increments.
- Each comparison evaluates the shared table-base expression.
- Zero selects an index; every nonzero value advances the search.
- Success evaluates the base expression again for the integer store at that index.
- The integer store precedes the index-derived marker write.
- The store destination is not rechecked for zero.
- No explicit calls, allocation, deallocation, synchronization, rollback, or status return appear.

The registry interpretation is inferred intent. Broader invariants and target details remain unresolved as identified above.

No runtime tests have been performed.
