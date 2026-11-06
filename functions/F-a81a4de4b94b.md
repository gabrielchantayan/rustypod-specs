<!-- Trusted provenance: F-a81a4de4b94b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a81a4de4b94b

## Purpose

F-a81a4de4b94b selects a 20-byte slot and partially initializes it. Selection has three alternatives, in priority order:

1. Remove the first slot from a reuse chain.
2. Consume the current cursor position when the cursor differs from its limit.
3. Obtain and publish a new storage region when the reuse chain is empty and the cursor equals the limit.

There is no explicit return value. Observable results consist of memory updates and, on the growth path, two calls to F-871e64ddc6de.

**Inferred intent:** The supplied state describes a region-backed pool of reusable slots. This interpretation does not establish the meaning of the objects stored in those slots, their ownership, or whether the partial initialization makes them ready for higher-level use.

## Interface

### Input and result

The sole explicit input is an address identifying state with four consecutive 32-bit fields:

| Byte offset | Width | Role |
|---|---:|---|
| 0 | 32 bits | Current region-descriptor address, or zero |
| 4 | 32 bits | Reuse-chain head address, or zero |
| 8 | 32 bits | Current slot-selection cursor |
| 12 | 32 bits | Limit compared with the cursor |

The accessed state spans 16 bytes. Address-bearing fields use the represented 32-bit address format.

The function returns no value and does not explicitly place the selected-slot address into a dedicated result field. How a caller subsequently identifies that slot is **documented_only**.

### Descriptor representation

Growth obtains storage used as a 12-byte descriptor:

| Byte offset | Width | Value written during growth |
|---|---:|---|
| 0 | 32 bits | Descriptor-head address read after both dependency calls |
| 4 | 32 bits | Newly calculated capacity |
| 8 | 32 bits | Second dependency result |

For a preexisting descriptor, this function directly consults only its capacity at offset 4. It does not traverse predecessor descriptors.

### Slot representation

The slot stride is 20 bytes. Accesses within a selected slot are:

| Byte offset | Width | Access |
|---|---:|---|
| 0 | 1 byte | Written with zero |
| 1–3 | 3 bytes | Not explicitly accessed |
| 4 | 32 bits | Written with zero |
| 8 | 32 bits | Written with zero |
| 12 | 32 bits | Read as a successor address on the reuse path; written with zero on every path |
| 16–19 | 4 bytes | Not explicitly accessed |

The function explicitly clears 13 bytes, not the entire slot.

### Preconditions for ordinary completion

The following are conditions for the accesses to succeed, not conditions checked by the function:

- The supplied state supports the indicated reads and writes.
- A nonzero reuse head supports a 32-bit successor read at slot offset 12 and the subsequent initialization writes.
- A cursor selected without growth supports the initialization writes.
- An existing descriptor consulted during growth supports a 32-bit capacity read at offset 4.
- Dependency results support the descriptor and slot accesses made through them.
- Addresses, alignment, and storage extents satisfy the execution environment’s memory rules.

The function does not validate chain membership, detect cycles, check available slot size, or establish that the cursor is below the limit.

Non-overlap between state, descriptors, and slots is not established by the evidence. Ordinary pool-style outcomes below assume such non-overlap where stated; overlapping storage must instead be interpreted using the ordered accesses.

## Behavior

### Selection priority and reuse

The first relevant state read is the reuse head at offset 4. A nonzero value selects that slot immediately, regardless of the cursor and limit.

The selected slot’s 32-bit field at offset 12 is read before initialization. Its value replaces the state’s reuse head. The selected slot is then partially cleared.

This path performs no explicit cursor, limit, or descriptor-head read and makes no dependency call.

For distinct state and slot storage, the result is removal of the first reuse-chain element. No check establishes that the successor is valid or distinct from the selected slot. A self-linked slot therefore leaves its own address in the reuse-head field, while the slot’s successor field is subsequently cleared.

### Cursor selection without growth

When the initial reuse head is zero, the function obtains the cursor at state offset 8 and compares it for equality with the limit at offset 12.

If they differ, the original cursor is the selected-slot address. The cursor field is updated to that address plus 20 before the selected slot is initialized.

Equality is the entire exhaustion test. The function does not compare address ordering or check whether 20 bytes remain. A cursor beyond the limit still selects a slot when the two values differ.

No descriptor access or dependency call occurs on this path.

### Capacity calculation during growth

Growth occurs only when the initial reuse head is zero and the cursor equals the limit.

A zero descriptor head selects a capacity of 32 slots.

Otherwise, the existing descriptor’s capacity is read as an unsigned 32-bit value. Two candidates are calculated:

- The old capacity plus 32.
- The old capacity plus its logical right shift by one bit, plus its logical right shift by three bits.

All capacity additions use unsigned 32-bit arithmetic, with reduction modulo \(2^{32}\). The comparison is unsigned. The larger wrapped candidate is selected; equality selects the shifted-sum candidate, producing the same numeric capacity.

Without overflow, this chooses the greater of additive growth by 32 and approximately 1.625-fold growth. That mathematical interpretation is not a replacement for the specified wrapped arithmetic.

The slot-storage extent is the selected capacity multiplied by 20, again reduced to 32 bits. The multiplication is unchecked. A wider implementation must not replace these operations with unbounded or checked arithmetic if that changes the observable request or comparison result.

### Growth calls and publication order

After capacity calculation, the following order is established:

1. F-871e64ddc6de is called with first argument 12 and second argument zero. Its result is retained as the new descriptor address.
2. F-871e64ddc6de is called with the wrapped slot-storage extent and second argument zero.
3. The second result is written to new-descriptor offset 8.
4. The current descriptor-head field is read from state offset 0 and written to new-descriptor offset 0.
5. The calculated capacity is written to new-descriptor offset 4.
6. The new descriptor address is written to state offset 0.
7. New-descriptor offset 8 is read to obtain the selected-slot address.
8. The selected-slot address plus the wrapped storage extent is written to state offset 12 as the new limit.
9. The selected-slot address plus 20 is written to state offset 8 as the new cursor.
10. The selected slot is partially initialized.

The predecessor link is therefore based on a state read after both dependency calls. It is not necessarily the descriptor head used earlier for capacity calculation. The dependency’s preservation of caller-accessible memory is unknown.

Selection also uses a read of the new descriptor’s storage-address field after descriptor publication. With distinct storage and no intervening mutation, this retrieves the second dependency result. Aliasing can invalidate that equivalence: an earlier write may affect the field read later.

There is no explicit reuse-head reset during growth.

The address additions represent 32-bit address calculations. Successful access requires that the resulting addresses be valid in the execution environment; the function performs no wraparound or range validation.

### Initialization shared by all paths

Every selected slot receives these writes in this order:

1. A 32-bit zero at offset 4.
2. A 32-bit zero at offset 8.
3. A 32-bit zero at offset 12.
4. A one-byte zero at offset 0.

Initialization follows the relevant state changes. In particular, the reuse head or cursor has already advanced when slot clearing begins; on growth, the descriptor and limit have also been published.

The final byte write does not establish a synchronization protocol or an object-validity flag. The meaning of that byte is unknown.

## Side effects

### Direct memory changes

The path determines the direct writes:

| Path | State writes | Other direct writes |
|---|---|---|
| Reuse | Reuse head | Selected-slot initialization |
| Existing cursor | Cursor | Selected-slot initialization |
| Growth | Descriptor head, limit, cursor | Three descriptor fields and selected-slot initialization |

With non-overlapping storage and no external mutation:

- Reuse preserves the descriptor head, cursor, and limit.
- Cursor selection without growth preserves the descriptor head, reuse head, and limit.
- Growth preserves the reuse head.
- Slot bytes 1–3 and 16–19 retain their previous contents.

These preservation statements are conditional. An access through one logical object can affect another when their storage overlaps.

### Ownership and lifetime

**Established:** The function retains dependency results in the descriptor/state structure. It makes no explicit release call and does not remove or release an older descriptor.

**Inferred intent:** Each new descriptor records a backing storage region and links to the preceding descriptor.

**documented_only:** The allocation domain, ownership transfer, release obligations, lifetime of returned storage, and existence of other references.

Removing a slot from the reuse chain does not prove exclusive ownership.

### Failure and partial completion

There is no explicit failure result, null-result check, recovery branch, or rollback.

Both dependency calls precede the first explicit write through the new descriptor address. If the first call returns an unusable address but returns normally, the second call is still attempted before that descriptor address is dereferenced.

A zero second result is stored without being recognized as an error. Subsequent address calculations and slot writes still follow the normal sequence.

A dependency might fail through a mechanism not visible here, such as an exception, trap, or termination. Its behavior is **documented_only**. Invalid results or invalid caller state may cause later accesses to fail under the execution environment’s rules.

Previously completed effects are not rolled back if a later operation fails. No atomicity or transactional guarantee is established. Concurrent access is not synchronized by this function.

## Dependencies

### F-871e64ddc6de

This is the only explicit dependency. A normally continuing growth path calls it twice:

| Order | First argument | Second argument | Result use |
|---|---:|---:|---|
| 1 | 12 | 0 | New descriptor address |
| 2 | Selected capacity × 20, modulo \(2^{32}\) | 0 | Stored at descriptor offset 8 and subsequently used for slot selection |

Neither non-growth path calls it.

**Inferred intent:** The request sizes and result uses suggest allocation of descriptor storage and slot storage.

**documented_only:** Whether it actually allocates memory; whether results are distinct, aligned, initialized, or nonzero; its failure convention; its ownership contract; and its side effects on globals or caller-accessible memory.

The second argument’s meaning is unknown. It must not be interpreted as an allocation flag or initialization guarantee without an independent contract.

The capacity remains the previously calculated value across these calls. The descriptor-head value used for the new predecessor link is read afterward. Consequently, examples that require stable surrounding memory explicitly assume that the dependency does not mutate it.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Unless an example states otherwise, assume valid, suitably aligned, non-overlapping storage, successful dependency returns, and no dependency-induced mutation of relevant memory. Address sums in the ordinary examples are assumed representable without address wrap.

### Reuse takes priority over exhaustion

Given a reuse head identifying slot A, A’s successor field identifying B, and equal cursor and limit:

- A is selected.
- The reuse head becomes B before A is cleared.
- No dependency calls occur.
- Descriptor head, cursor, and limit remain unchanged.
- A receives the four initialization writes in the specified order.
- A’s bytes 1–3 and 16–19 remain unchanged.

The equal cursor and limit do not cause growth because the nonzero reuse head takes priority.

### A self-linked reuse slot

Given a reuse head identifying A and A’s successor field also identifying A:

- The reuse head is written with A.
- A’s successor field is subsequently cleared to zero.
- The function does not reject the cycle.

Thus, the head still identifies A after this invocation, but A’s former self-link is gone. No cycle-repair behavior is implied.

### One cursor slot remains

Given an empty reuse chain, cursor P, and limit P + 20:

- P is selected.
- The cursor becomes P + 20 and therefore equals the limit.
- P is partially initialized.
- No descriptor access or dependency call occurs.

A later invocation reaches growth if the reuse head remains empty and the state remains otherwise unchanged.

### Initial growth

Given an empty reuse chain, equal cursor and limit, and a zero descriptor head:

- Capacity is 32.
- The two calls have first arguments 12 and 640, respectively.
- Both second arguments are zero.

If the results are distinct valid addresses H and D:

- H contains predecessor zero, capacity 32, and storage address D.
- The descriptor head becomes H.
- The limit becomes D + 640.
- The cursor becomes D + 20.
- The slot at D receives the partial initialization.

This does not imply that the remaining storage is initialized.

### Later growth

For an existing capacity of 32:

- Additive candidate: 64.
- Shifted-sum candidate: 32 + 16 + 4 = 52.
- Selected capacity: 64.
- Storage request: 1,280 bytes.

For an existing capacity of 128:

- Additive candidate: 160.
- Shifted-sum candidate: 128 + 64 + 16 = 208.
- Selected capacity: 208.
- Storage request: 4,160 bytes.

### Wrapped capacity arithmetic

For an existing capacity of 4,294,967,280, hexadecimal 0xFFFFFFF0:

- The right-shifted values are 2,147,483,640 and 536,870,910.
- Adding 32 wraps to 16.
- The shifted sum wraps to **2,684,354,534**, hexadecimal **0x9FFFFFE6**.
- Unsigned comparison selects **2,684,354,534**.
- Multiplication by 20 wraps to a storage request of **2,147,483,128 bytes**, hexadecimal **0x7FFFFDF8**.

No overflow error is reported. This arithmetic result does not establish that the requested extent can hold the represented number of 20-byte slots or that the resulting accesses are safe.

### Cursor beyond the limit

Given an empty reuse chain and a cursor numerically greater than the limit, the function still selects the cursor because it differs from the limit. It advances the cursor by 20 and attempts initialization.

There is no automatic repair, rejection, or growth based solely on the cursor being beyond the limit.

### Descriptor-head mutation during a dependency call

Suppose growth calculates capacity using an existing descriptor, and a dependency then changes the state’s descriptor head to another valid descriptor address.

Provided no other relevant memory is changed:

- The calculated capacity is unchanged.
- The new descriptor’s predecessor is the changed head read after the calls.
- It is not necessarily the descriptor used for capacity calculation.

This example describes the observed read timing, not a guarantee that such dependency mutation is supported or safe.

## Uncertainties

The following remain **documented_only**:

- The higher-level meaning of the state and slot fields.
- How callers obtain or use the selected slot without an explicit returned address.
- Whether callers guarantee non-overlap and valid address arithmetic.
- Any external capacity bound preventing overflow.
- The full contract of F-871e64ddc6de, including failure, ownership, alignment, initialization, reentrancy, and memory side effects.
- Initialization and interpretation of untouched slot bytes.
- External synchronization and permitted concurrent access.
- Eventual reclamation of descriptors and backing storage.

The pool interpretation is inferred intent. These unknowns do not justify adding full-slot clearing, validation, allocation recovery, ownership transfer, or stronger publication guarantees.

## Evidence

The supplied static representation of F-a81a4de4b94b establishes the branch priority, accessed layouts, 20-byte stride, ordered state updates, unsigned capacity arithmetic, two-call growth sequence, and four initialization writes. The supplied dependency mapping identifies F-871e64ddc6de as the sole explicit callee.

The overflow example follows directly from the unsigned shifts, modulo-\(2^{32}\) additions, unsigned comparison, and modulo-\(2^{32}\) multiplication. It does not rely on an assumed allocator contract.

No dependency implementation or broader caller contract is supplied. No runtime tests have been performed.
