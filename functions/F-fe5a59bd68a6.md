<!-- Trusted provenance: F-fe5a59bd68a6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fe5a59bd68a6

## Purpose

F-fe5a59bd68a6 updates an existing named property or prepares and commits a new property record in a caller-supplied state object.

An existing active property is updated according to its stored type. A new property requires definition lookup, possible definition creation, possible record-array growth, value preparation, and conditional registration. Certain newly committed names also update cached state fields. Existing-property updates bypass those cache updates.

This contract distinguishes directly established operations from inferred property-management intent. Allocation, release, conversion, comparison, and registration semantics remain dependent on opaque callees. Failures are not transactional.

## Interface

### Inputs

| Input | Width | Established use |
|---|---:|---|
| State address | 32 bits | Base address for mutable state fields and an embedded lookup structure |
| Name argument | 32 bits | Passed to lookup, definition-creation, and comparison dependencies |
| Value argument | 32 bits | Nullable address passed to type-dependent dependencies |

The name argument appears to identify text, but its encoding, termination, alignment, and readable extent are not independently established. The value argument is not uniformly interpreted: type 1 uses a length-like operation and an external transfer operation; types 2 and 3 use conversion-shaped operations.

### State layout

Offsets are relative to the state address.

| Offset | Access width | Observed role |
|---|---:|---|
| `0x1c` | 32 bits | Cached spacing classification |
| `0x24` | 32 bits | Cached default-character value |
| `0x28` | 32 bits | Cached ascent value |
| `0x2c` | 32 bits | Cached descent value |
| `0x48` | 32 bits | Record capacity |
| `0x4c` | 32 bits | Committed record count |
| `0x50` | 32 bits | Record-array address |
| `0x80` | 32 bits | Active-property lookup value |
| `0x4088` | 32 bits | Allocation-related context |
| `0x408c` | 32 bits | Extended-definition-table address |
| `0x4094` | Structure extent unknown | Embedded definition-lookup structure |

The allocation-related context is read before the first dependency invocation and retained. Later allocation and registration operations use that captured value rather than reloading this field.

A separate global location holds a 32-bit fixed-definition-table base. Its value is read during definition selection, including when the selected definition ultimately comes from the extended table. Its immutability is not established.

Each record occupies 16 bytes:

| Record offset | Width | Meaning established by use |
|---|---:|---|
| `0` | 32 bits | Definition word supplied to registration |
| `4` | 32 bits | Type discriminator |
| `8` | 32 bits | Additional definition metadata |
| `12` | 32 bits | Value word or value address |

Nonzero lookup results are treated as addresses containing a 32-bit index at offset `4`.

### Return value

The return is a 64-bit concatenation:

- Bits `31:0`: local status, initially zero.
- Bits `63:32`: a working word initially equal to the name argument.

The high word changes only along the following visible paths:

- Before a qualifying type-1 value allocation, it becomes zero.
- Before record-array growth, it becomes zero if capacity is zero; otherwise it becomes the then-read record-array address.
- Other paths retain its previous value.

The low word can be assigned a definition-creation or registration result. Allocation calls can receive its address. Consequently, neither a conventional whole-result error code nor a universally zero high word is established.

### Validity requirements

There is no general input, index, bounds, allocation-result, or arithmetic validation. Defined execution requires valid storage for every access actually reached, including dependency-specific input requirements.

In particular:

- A repeated definition lookup is dereferenced without another null check.
- A selected `SPACING` value must permit a byte read, regardless of its type.
- Reaching definition selection requires the fixed-table-base global to be readable even for an extended-table index.
- A zero allocation status does not itself establish a nonzero returned address.

## Behavior

### Existing active property

The first lookup invokes F-a19afa5473aa with the name and the active lookup value read from state offset `0x80`.

A nonzero result selects an existing record using the record-array address and the lookup index scaled by 16. The selected record address is retained across subsequent calls; it is not recomputed after each dependency invocation.

For **type 1**, the function:

- Passes the captured allocation context and the old value word to F-bf7242c98d06.
- Writes zero to the record’s value word after that call.
- For a nonnull value argument, invokes F-1cb585febd84 and tests whether its signed integer result plus one is greater than one.
- If the test passes, sets the return high word to zero and invokes F-6dd003795db7 with the captured context, `1`, zero, the computed size, zero, and the status address.
- Stores the returned word into the record before checking status.
- If status is zero, invokes F-UNMAPPED-1 with the returned word and the original value argument.
- Returns without rollback.

If the value argument is null or the size test fails, the record value is written as zero again. No value allocation or external transfer occurs on that branch.

Thus, the old value is relinquished to the release-shaped dependency before replacement success is known. Actual release behavior is not established.

For **type 2**, F-3f754031ddf9 receives the value argument, zero, and decimal base `10`. Its returned 32-bit word replaces the record value.

For **type 3**, F-b5b672be5c1a receives the same argument pattern, and its returned word replaces the record value.

Other types leave the value word unchanged. These numeric and unsupported-type paths contain no explicit status assignment.

Every existing-property path bypasses definition creation, array growth, registration, count increment, and special-name cache updates.

### Definition lookup and creation

If the active lookup returns zero, a second lookup invokes F-a19afa5473aa with the name and the address of the embedded structure at state offset `0x4094`.

If this lookup returns zero:

- F-681d3b42fcc7 receives the name and `1`.
- Its returned word becomes status.
- Nonzero status causes immediate return.
- Zero status permits another embedded-structure lookup.

There is no second null-result check before the repeated lookup result is later dereferenced. Creation success therefore does not locally guarantee a valid definition result.

Definition creation precedes record growth and value preparation. Later failure does not visibly undo creation.

### Conditional record-array growth

Growth occurs only when the committed count equals capacity. A count greater than capacity does not enter this branch.

Before the growth call, the return high word becomes zero for zero capacity, or the current record-array address otherwise. An auxiliary argument, initially the original value argument, becomes the status address.

The supplied representation exposes F-6dd003795db7 receiving the captured context and `16`. Its complete growth-call interface is unresolved; the value-allocation signature must not be assumed to establish missing growth arguments.

The returned word is stored at state offset `0x50` before status is checked. Nonzero status causes return without restoring the former address.

On zero status:

- The function selects a slot using the then-read array address plus the then-read capacity scaled by 16.
- It writes zero to the slot’s four words in ascending offset order.
- It increments capacity by one.

There is no visible rejection of a null returned array address. The function does not visibly undo slot writes or capacity growth if later preparation or registration fails.

### Definition-table selection and record initialization

After any required growth has succeeded, the following read sequence is established:

1. Read the lookup result’s index at offset `4` as an **unsigned 32-bit value**.
2. Unconditionally read the 32-bit fixed-definition-table base from its global location.
3. If the index exceeds `82`, read state offset `0x408c` and substitute that extended-table base.

The global read therefore occurs for both fixed and extended definitions. On the extended path it precedes the extended-table-base read, even though its returned value is replaced. The supplied evidence does not justify treating this global as immutable or substituting a value captured before intervening dependency calls.

The effective entry selection is:

- Index `0` through `82`: fixed-table base plus index × 16.
- Index above `82`: extended-table base plus `(index − 83) × 16`.

For the extended case, the represented address calculation scales the original index and then subtracts 1,328 bytes, equivalent to 83 records in non-overflowing arithmetic. There is no bounds or overflow guard.

The destination is selected using the current record-array address and committed count scaled by 16. The first three definition words are read and written to corresponding destination offsets in ascending order. The source type word is then read again. The fourth definition word is not copied.

These are ordered word operations, not an established atomic record copy. Aliasing or dependency-induced changes must not be excluded without additional preconditions.

### New-record value preparation

For **type 1**, a nonnull value argument is passed to F-1cb585febd84. Its result is increased by one and subjected to a signed greater-than-one test.

If that test passes:

- The auxiliary argument becomes the status address.
- The return high word becomes zero.
- F-6dd003795db7 receives the captured context, `1`, zero, computed size, zero, and status address.
- Its returned word is stored in the destination value field before status is checked.
- Nonzero status causes immediate return.
- Zero status permits F-UNMAPPED-1 to receive the returned word, original value argument, and computed size.

Unlike the existing-record path, this visible external invocation supplies a third size argument.

For null input or a failed size test, the destination value is written as zero.

For **types 2 and 3**, the corresponding conversion-shaped dependency receives the value argument, zero, and `10`; its returned word is stored in the destination value field. Null input is passed through without a local guard.

For **other types**, value preparation does not write the fourth word. It is zero only if an applicable earlier write established that value—for example, if growth zeroed this same slot and nothing subsequently changed it. Otherwise its prior contents remain unestablished.

### Registration and commitment

F-7f14f1ab6851 compares the name against `COMMENT` with count `7`.

A zero result skips registration but still proceeds to commitment. Otherwise, F-c33185036507 receives:

1. The retained destination record’s first word.
2. The then-read committed count.
3. The then-read active lookup value.
4. The captured allocation context.
5. The current return high word.
6. The current auxiliary argument.

The last two arguments depend on earlier branches. They are not uniformly the original name and value arguments.

The registration result replaces status. Nonzero status prevents commitment and cache updates but does not roll back prior writes or allocations.

On skipped or successful registration, the current committed count is incremented by one.

### Post-commit cache updates

After the count increment, comparisons occur in this order. Each subsequent comparison is reached only if the preceding one returned nonzero.

| Compared name | Count | Action on zero result |
|---|---:|---|
| `DEFAULT_CHAR` | `12` | Copy retained record value to state offset `0x24` |
| `FONT_ASCENT` | `11` | Copy retained record value to `0x28` |
| `FONT_DESCENT` | `12` | Copy retained record value to `0x2c` |
| `SPACING` | `7` | Read one byte through the retained record’s value word |

For `SPACING`, `p` or `P` writes `8` to offset `0x1c`; `m` or `M` writes `16`; `c` or `C` writes `32`. Any other byte leaves that field unchanged.

There is no type or null check before this byte read. Commitment has already occurred when it is attempted. Cache values are read through the retained destination address, not through a freshly selected array slot.

## Side effects

Visible effects include value replacement, record-array-address replacement, capacity changes, partial record initialization, committed-count increments, and cache writes. Dependencies may additionally modify definition storage, lookup structures, allocation state, or other reachable state.

Failure is nontransactional. Surviving effects can include:

- An old type-1 value passed to the release-shaped dependency.
- A created definition.
- A replaced array address, including after reported growth failure.
- Increased capacity.
- An initialized but uncommitted record.
- A newly allocated value retained after registration failure.

No local cleanup releases a newly prepared value following registration failure.

Ownership transfer, allocator compatibility, and whether any dependency releases or retains supplied storage remain unknown. No dependency is assumed to preserve globals, the state object, or reachable memory. Reads performed after calls observe the storage then present; retained addresses and captured arguments are not automatically refreshed.

The unconditional fixed-table-base read is part of the observable access sequence, even when an extended table supplies the selected entry.

## Dependencies

| Opaque ID | Established invocation role |
|---|---|
| F-a19afa5473aa | Lookup using a name and a lookup value or embedded-structure address; a nonzero result supplies an index at offset `4` |
| F-bf7242c98d06 | Receives captured context and old type-1 value before local clearing; release intent is inferred |
| F-1cb585febd84 | Produces the integer used in type-1 size preparation; length intent is inferred |
| F-6dd003795db7 | Returns words stored as array or value addresses; explicit value-allocation calls expose status storage |
| F-3f754031ddf9 | Type-2 conversion-shaped operation with zero and base `10` |
| F-b5b672be5c1a | Type-3 conversion-shaped operation with zero and base `10` |
| F-681d3b42fcc7 | Definition-creation-shaped operation whose result gates repeated lookup |
| F-7f14f1ab6851 | Comparison against fixed text with an explicit count; zero selects a matching branch |
| F-c33185036507 | Registration-shaped operation whose result gates commitment |
| F-UNMAPPED-1 | Opaque external operation after successful type-1 allocation; transfer or copying intent is inferred |

F-UNMAPPED-1 is a contract-local designation, not an independently resolved dependency identity.

## Acceptance examples

These are conditional static cases, not executed tests. Unless specified otherwise, they assume valid storage, non-overflowing arithmetic, compatible dependency behavior, and no unmentioned dependency mutation.

1. **Existing numeric value:** Active lookup selects type 2, and F-3f754031ddf9 returns `42`. The record value becomes `42`. Count and caches are unchanged. Status remains zero and the return high word remains the name argument.

2. **Existing type 1, null input:** F-bf7242c98d06 receives the old value. The record value becomes zero. Neither the length-like dependency nor a value allocator is invoked.

3. **Existing type-1 allocation failure:** The size test passes and allocation sets nonzero status. The returned word is nevertheless stored. The external transfer is skipped, and the old value is not restored.

4. **Definition creation failure:** Both initial lookups fail and creation returns nonzero. No array growth or commitment follows. The low return word contains that status.

5. **Fixed-table boundary:** Index `82` causes the fixed-table-base global to be read and selects fixed entry `82`. The extended-table-base field is not read for selection.

6. **First extended entry:** Index `83` first causes the fixed-table-base global to be read, then causes state offset `0x408c` to be read. Extended entry `0` is selected. The earlier global read is not omitted merely because its value is replaced.

7. **Changed global base:** An earlier dependency changes the fixed-table-base global. When index `10` reaches definition selection, the function reads the then-current global value rather than an entry-time snapshot.

8. **New `COMMENT`:** The comparison returns zero after successful preparation. Registration is skipped, but committed count increases.

9. **Registration failure:** A prepared ordinary record receives nonzero registration status. Count and caches are not updated. Earlier record writes and any completed allocation remain without visible rollback.

10. **New ascent versus existing ascent:** A newly committed record selected by the `FONT_ASCENT` comparison supplies its value to offset `0x28`. Updating an already-active ascent record bypasses that cache write.

11. **Spacing classification:** A newly committed `SPACING` record points to byte `M`; offset `0x1c` becomes `16`. Byte `x` leaves the prior classification unchanged.

12. **Unsupported type without initialization:** With no growth and no other fourth-word write, an unsupported definition type preserves the destination’s prior fourth word and can still be committed.

## Uncertainties

The following remain **documented_only**:

- Intended calling convention and consumer interpretation of the 64-bit result.
- Complete growth-call arguments and allocation failure semantics.
- F-UNMAPPED-1 identity, full signature, transfer extent, and error behavior.
- Exact name-comparison semantics, including termination and prefix handling.
- Numeric syntax, signedness, range behavior, and error signaling.
- Allocation ownership, release behavior, and cleanup responsibility.
- Dependency mutation of globals, state, or retained storage.
- The fixed-table-base global’s update rules and lifetime.
- Recovery when repeated definition lookup returns zero.
- Table bounds, record bounds, alignment, and valid storage lifetimes.
- Overflow semantics beyond the exposed 32-bit operands and signed size comparison.

Index scaling, address addition, count/capacity increments, and size-plus-one have no visible guards. Algebraic entry formulas describe intended locations only where arithmetic is valid; they do not establish checked arithmetic. The signed size test is not a general overflow check.

## Evidence

Established static evidence comprises the supplied function representation and dependency-ID mapping: widths, offsets, branch conditions, explicit arguments, ordered writes, status gates, record layout, table threshold, comparisons, and return-word assembly.

The definition-selection sequence specifically establishes an unsigned index read, an unconditional fixed-table-base global read, and only then a conditional extended-table-base read. Nothing supplied establishes that the global pointer is immutable.

Property management, allocation, release, copying, decimal conversion, and registration are inferred roles rather than independently verified dependency contracts.

No runtime tests have been performed.
