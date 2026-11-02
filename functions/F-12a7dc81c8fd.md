<!-- Trusted provenance: F-12a7dc81c8fd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-12a7dc81c8fd

## Purpose

F-12a7dc81c8fd conditionally dispatches an operation on an embedded object according to a discriminator in a second record.

Two discriminator values are recognized:

- **6:** dispatch the embedded-object operation without explicitly inspecting or updating the tracked flag.
- **21:** compare a masked flag from a dependency-selected record with a stored value in the first record. Dispatch only if those values differ, updating the stored value before dispatch.

Other discriminator values produce no dispatch by this function.

A record-selection dependency is invoked before the discriminator is inspected, including on paths that subsequently do nothing. Every normally returning path returns the 32-bit value zero. The return value does not distinguish ignored input, an unchanged flag, or completed dispatch.

The interpretation as change-triggered notification is **inferred intent**. The established behavior is the conditional comparison, state write, dependency calls, and indirect invocation described below.

## Interface

### Inputs

The function receives two 32-bit, address-bearing arguments:

| Input | Role |
|---|---|
| **Owner address** | Base address of a record containing an embedded object and a tracked 32-bit value. |
| **Event address** | Base address of a record containing a discriminator and supplied to the record-selection dependency. |

No explicit lengths, ownership annotations, nullability guarantees, or validation parameters are present.

The behavior requires readable or writable storage only when the selected execution path accesses it. For example, an unrecognized discriminator causes no direct access to the owner record, although the initial dependency can still impose its own requirements.

### Established layout

All offsets are byte offsets. The visible operations use 32-bit values and a 32-bit address representation.

| Location | Width | Visible role |
|---|---:|---|
| Event address + 4 | 32 bits | Discriminator, compared with 6 and 21. |
| Owner address + 0x18 | Address of embedded region | Argument to the preparation dependency and the eventual indirect operation. |
| Owner address + 0x18 | 32 bits | Dispatch-table address, read when resolving the indirect operation. |
| Owner address + 0x68 | 32 bits | Stored comparison value, read and conditionally overwritten on the discriminator-21 path. |
| Selected-record address + 0x10 | 32 bits | Flag word masked with 0x40000000. |
| Dispatch-table address + 0 | 32 bits | Callable address used for indirect invocation. |

The embedded region is addressed directly within the owner record. Passing owner address + 0x18 does not mean passing the 32-bit value stored there: the dependencies receive the address of that embedded region.

The selected-record address is the return value of F-682adfd5a561. No visible check establishes whether it is nonzero or points to a valid record.

### Output and completion

On every explicit normal return, the result is a 32-bit zero.

Results from F-8bb3020950cd, F-1c60e26b3436, and the indirect operation are not used to determine this return value. Their result types and meanings are not established here.

Normal completion requires all executed calls and memory accesses to complete. No visible local mechanism converts an invalid address, invalid indirect target, or dependency failure into a distinct result.

## Behavior

### 1. Resolve a record before examining the discriminator

The function first calls F-682adfd5a561 with:

1. The event address.
2. The 32-bit constant 0x3500, equal to decimal 13,568.

Its returned 32-bit value is retained as the selected-record address.

This call occurs on every path. In particular, the function does not first check whether the event discriminator is recognized.

The discriminator is read after this dependency returns. Therefore, the relevant value is the event record’s value at that time, not necessarily its value at function entry. The dependency’s effects on that record or other state are unknown.

The constant 0x3500 is an opaque selector. Its meaning as a type identifier, request code, size, or other quantity is not established.

### 2. Select the discriminator path

The function reads the 32-bit discriminator at event address + 4 and compares it with 6.

- If that read equals 6, execution proceeds directly to embedded-object dispatch.
- Otherwise, the supplied representation contains another access to the same discriminator field, this time comparing it with 21.
- If the second comparison does not equal 21, the function returns zero.
- If the second comparison equals 21, execution follows the flag-comparison path.

No dependency call appears between the two discriminator comparisons. Nevertheless, the source-facing contract preserves them as separate reads rather than treating them as one captured value.

Under ordinary stable-memory conditions, this gives the expected classification into 6, 21, and other values. Exact processor load count and behavior under concurrent or externally changing storage are not established by the supplied representation.

### 3. Handle discriminator 21

The discriminator-21 path first calls F-8bb3020950cd with the selected-record address.

Only after that call returns does the function evaluate the comparison between:

- The selected record’s 32-bit word at offset 0x10, masked with 0x40000000.
- The full 32-bit stored value at owner address + 0x68.

The mask retains only bit 30. The resulting masked value can therefore be only:

- 0x00000000; or
- 0x40000000.

This is not a conversion to numeric Boolean values zero and one.

The owner’s stored value is not masked before comparison. Its entire 32-bit representation participates. Consequently, a stored value containing unrelated set bits differs from either permitted masked value, even if its own bit 30 matches.

The supplied expression does not establish the relative evaluation order of the selected-record read and the owner-state read. Both belong to the comparison after the first F-8bb3020950cd call.

#### Equal comparison

If the masked selected flag equals the full stored value:

- The function returns zero.
- It makes no second call to F-8bb3020950cd.
- It makes no explicit write to the tracked owner value.
- It does not call the embedded-object preparation dependency.
- It does not perform the indirect operation.

The initial record-selection call and the first F-8bb3020950cd call have already occurred. Their side effects, if any, are not undone.

#### Unequal comparison

If the values differ:

1. F-8bb3020950cd is called a second time with the same retained selected-record address.
2. After that call returns, the selected record’s 32-bit word at offset 0x10 is read again.
3. That new value is masked with 0x40000000.
4. The masked result is written as a 32-bit value to owner address + 0x68.
5. Execution proceeds to embedded-object dispatch.

The stored result comes from the read after the second dependency call, not necessarily from the read that triggered the mismatch.

There is no second equality comparison. If the second dependency call changes the flag so that the newly stored value equals the owner value used in the earlier comparison, dispatch still occurs.

The second dependency call is not followed by another record-selection call. The function continues using the retained selected-record address; whether that address remains valid is an external precondition.

### 4. Dispatch the embedded-object operation

Both the discriminator-6 path and the unequal discriminator-21 path perform the following sequence:

1. Call F-1c60e26b3436 with the embedded-region address, owner address + 0x18, and a second argument of zero.
2. After that call returns, read the 32-bit dispatch-table address stored at the embedded-region address.
3. Read the callable address from the first 32-bit entry of that table.
4. Invoke that callable address with the embedded-region address as its sole visible argument.
5. Return zero if the invocation returns normally.

The indirect target is resolved after F-1c60e26b3436 returns. A table pointer or first table entry changed by that dependency therefore affects the subsequent invocation.

There is no visible null check, table-bound check, callable-target check, or alternative dispatch path.

On the discriminator-6 path, the function itself performs no direct read of selected-record flags and no direct access to the tracked owner field at offset 0x68. Dependencies may still affect those locations.

## Side effects

### Explicit write

The only explicit data write shown is a 32-bit store to owner address + 0x68 on the unequal discriminator-21 path.

That store writes exactly 0x00000000 or 0x40000000, based on the post-second-call flag read. It replaces the complete stored word; unrelated bits in the previous stored value are not preserved.

The store occurs before F-1c60e26b3436 and before the indirect operation.

### Call ordering

For ordinary stable discriminator storage, the visible call sequences are:

| Path | Calls in order |
|---|---|
| Discriminator other than 6 or 21 | F-682adfd5a561 |
| Discriminator 6 | F-682adfd5a561 → F-1c60e26b3436 → indirect operation |
| Discriminator 21, comparison equal | F-682adfd5a561 → F-8bb3020950cd |
| Discriminator 21, comparison unequal | F-682adfd5a561 → F-8bb3020950cd → F-8bb3020950cd → F-1c60e26b3436 → indirect operation |

These sequences describe this function’s calls, not any additional calls made internally by dependencies.

### Ownership and lifetime

No explicit allocation, release, ownership transfer, or reference-count operation is established.

The function neither visibly copies the records nor saves their addresses into an explicit persistent field. This does not establish that dependencies refrain from retaining addresses, transferring ownership, or releasing storage.

The owner, event, selected record, dispatch table, and callable target must remain usable for every access that actually executes. Lifetime guarantees across dependency calls are unknown.

### Aliasing and external state

No non-aliasing requirement is stated. If the supplied addresses overlap, an explicit write or a dependency effect may influence a later access through another address.

Claims such as “the stored flag is unchanged on this path” apply only to this function’s explicit writes, unless dependencies and aliasing are additionally constrained. No dependency is assumed to preserve globals, object state, table contents, or memory ownership.

## Dependencies

| Opaque ID | Visible invocation | Established role and limits |
|---|---|---|
| **F-682adfd5a561** | Event address, 0x3500 | Always called first. Returns the address-bearing value later used as the selected record. Selection semantics, failure representation, and ownership are unknown. |
| **F-8bb3020950cd** | Selected-record address | Called once before comparison on discriminator 21, and again on mismatch. Its effects and purpose are unknown. |
| **F-1c60e26b3436** | Embedded-region address, zero | Called immediately before resolving and invoking the indirect operation. Its effects and interpretation of zero are unknown. |

The indirect operation has no supplied opaque function ID. Its identity is determined dynamically from the first entry of the table addressed by the embedded region’s first word. It is not established to be any of the named dependencies.

Terms such as “refresh,” “lock,” “release,” “reset,” or “notify” would assign unverified semantics to these calls and are not contractual descriptions.

## Acceptance examples

These examples are conditional expectations derived from the supplied static representation. **No runtime tests have been performed.** Unless stated otherwise, dependencies return normally, required addresses remain valid, records do not alias, and dependencies do not alter the relevant fields.

### Unrecognized discriminator

After F-682adfd5a561 returns, both discriminator reads observe 9.

Expected behavior:

- The record-selection dependency has been called with the event address and 0x3500.
- No other dependency or indirect operation is called by this function.
- No explicit owner-state write occurs.
- The result is zero.

This is not a dependency-free rejection path.

### Discriminator 6

The first discriminator read observes 6.

Expected behavior:

- No direct selected-record flag read or tracked-state comparison occurs.
- F-1c60e26b3436 receives the embedded-region address and zero.
- The subsequently resolved first table entry is invoked with the embedded-region address.
- Normal completion returns zero.

Validity of the selected-record address is not needed for a direct selected-record dereference on this path, although the record-selection dependency has still executed.

### Discriminator 21 with equal clear state

The comparison-stage flag word is 0x80000001 and the owner’s stored value is zero.

Masking the flag produces zero. Expected behavior is one F-8bb3020950cd call, no explicit tracked-state write, no embedded-object dispatch, and return zero.

### Discriminator 21 with equal set state

The comparison-stage flag word is 0xC0000001 and the owner’s stored value is 0x40000000.

Only bit 30 survives masking. The values compare equal, so the function returns without a second F-8bb3020950cd call or dispatch.

### Stored value contains unrelated bits

The comparison-stage flag word is zero and the stored value is 0x00000001. The selected flag remains zero after the second F-8bb3020950cd call.

Expected behavior:

- The comparison is unequal.
- F-8bb3020950cd is called twice in total.
- Zero replaces the entire stored word.
- Embedded-object dispatch occurs.

Treating both operands as masked flags would incorrectly skip this dispatch.

### Flag changes during the second dependency call

The owner’s comparison-stage stored value is zero. The first flag read yields 0x40000000. After the second F-8bb3020950cd call, the flag word is zero.

The first comparison triggers the mismatch path. The function then writes zero and dispatches anyway. It does not reconsider dispatch based on the later flag value.

### Preparation changes the dispatch target

F-1c60e26b3436 changes the embedded region’s table address to another valid table.

The indirect invocation uses the first entry of the replacement table because table resolution occurs after preparation returns.

## Uncertainties

The following are **documented_only**:

- Semantic names and complete layouts of the records.
- Meaning of selector 0x3500 and discriminator values 6 and 21.
- Semantic meaning of bit 30.
- Calling conventions beyond the visible arguments and 32-bit representations.
- Dependency side effects, failure modes, ownership rules, and lifetime guarantees.
- Whether dependencies synchronize access or require synchronization from the caller.
- Whether the discriminator storage can change between its two represented reads.
- Exact machine-level load ordering beyond the source-visible sequencing described above.
- Target-specific consequences of invalid, misaligned, or wrapped addresses.
- Identity and semantics of the indirect operation.

Address calculations use byte offsets in the supplied 32-bit representation. No bounds or overflow checks are shown. The contract does not invent defined behavior for an address addition that fails to identify valid storage.

## Evidence

The established behavior comes solely from the supplied static function representation: its 32-bit accesses, constants, comparisons, calls, explicit store, indirect invocation, and zero-return exits.

The representation supports the important distinctions between:

- Unconditional record selection and conditional subsequent work.
- A single F-8bb3020950cd call on equality and two on mismatch.
- The flag read used for comparison and the later flag read used for storage.
- Masking the selected flag and comparing the owner’s complete stored word.
- Updating tracked state before dispatch preparation.
- Resolving the indirect target after preparation returns.

The interpretation as event handling or change notification is inferred intent, not independently established semantics. No implementation of a dependency, runtime trace, execution result, or test evidence was supplied.
