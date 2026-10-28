<!-- Trusted provenance: F-e6507aae2fa1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e6507aae2fa1

## Purpose

F-e6507aae2fa1 initializes a caller-provided result slot and conditionally constructs a small record. A zero primary value produces an empty result without dependency calls. A nonzero primary value causes a storage request and three record-field writes. An additional nonzero option causes a second storage request, initialization of two auxiliary fields, attachment of that storage to the first record, and a notification-like dependency call.

The function returns the address of the caller’s result slot, not the address of the constructed record.

This contract describes the supplied static representation. It does not establish that either dependency is an allocator or a notifier, that allocation succeeds, or that dependency calls preserve previously written memory.

## Interface

### Inputs and return value

| Item | Contract |
|---|---|
| Result-slot address | Address of writable storage for one result word. This storage is written before any conditional work. |
| Primary value | Integer value tested for equality with zero. A nonzero value is copied into the first field of the primary record. |
| Auxiliary option | Integer value tested for equality with zero, and only when the primary value is nonzero. Its magnitude is otherwise unused. |
| Return value | The original result-slot address, on every normally returning path. |

The representation uses integer-sized result and record fields, four-byte auxiliary fields, and pointer-to-integer conversions. The intended storage model is three consecutive 32-bit words for the primary record and two consecutive 32-bit words for the auxiliary record. It also treats a record address as representable in one integer word.

The supplied material does not independently establish the target ABI or native pointer width. Applying this contract to an implementation therefore requires confirming the 32-bit field model and the address representation. A wider-pointer environment cannot be assumed compatible merely because the logical fields fit in 32 bits.

No signed arithmetic is performed on either scalar input. Negative values, if representable by the input type, take the same nonzero branches as positive values. The primary value is copied without an explicit range check or transformation.

### Record layouts

Under the represented 32-bit word model, the primary record has the following layout:

| Byte offset | Width | Directly written content |
|---|---:|---|
| 0 | 32 bits | Primary input value |
| 4 | 32 bits | Integer value 1 |
| 8 | 32 bits | Initially zero; subsequently an auxiliary-storage address when the auxiliary option is nonzero |

The auxiliary record has this layout:

| Byte offset | Width | Directly written content |
|---|---:|---|
| 0 | 32 bits | Zero |
| 4 | 32 bits | Zero |

These are write descriptions, not unconditional guarantees about the fields at return. Dependencies invoked after particular writes may have effects that are not supplied.

### Preconditions

For a normal, fault-free execution:

- The result-slot address must permit a write of one result word.
- The first storage-provider call, when reached, must return an address permitting the three primary-record writes.
- The second storage-provider call, when reached, must return an address permitting the two auxiliary-record writes.
- Storage needed after an intervening dependency call must remain valid for the later accesses.
- Stored addresses must be representable in their destination fields if callers are to recover the intended addresses.

There are no checks enforcing these conditions. Neither a null result-slot address nor an unusable dependency result has an explicit recovery path.

## Behavior

### Initial result clearing

The first explicit memory operation is a zero write to the caller’s result slot. It occurs regardless of either scalar input.

This clearing precedes every dependency call. Consequently, a dependency that can observe the result slot may observe the cleared value. That observation is conditional on the dependency having access to the slot and on no intervening dependency effect changing it.

### Zero primary value

When the primary value is zero, execution performs no storage request and does not invoke the other dependency. The auxiliary option has no effect on this path.

The function returns the original result-slot address. The slot contains zero after the function’s explicit write; there are no later calls or direct writes on this path.

### Nonzero primary value

When the primary value is nonzero, the function calls F-8157a180d901 with the integer argument 12. Its returned value is used as the primary-record address without a null check or other validity test.

The function then directly writes the primary record in this order:

1. The primary input value at offset 0.
2. The integer value 1 at offset 4.
3. Zero at offset 8.

The order matters: the record is not initialized atomically, and its address has not yet been explicitly published through the caller’s result slot.

If the auxiliary option is zero, there are no further dependency calls. The function stores the primary-record address into the result slot and returns the slot’s original address.

### Nonzero primary value with auxiliary storage

When both scalar inputs are nonzero, the function first performs the primary-record initialization described above. It then calls F-8157a180d901 again, this time with the integer argument 8.

The returned address is used without validation. The function writes zero to the first auxiliary word and then zero to the second auxiliary word. It next stores the auxiliary address into the primary record’s field at offset 8.

Only after these writes does the function call F-b00260d1087a. No explicit arguments are supplied to that call, and no return value from it is consumed.

After that call returns normally, the function stores the saved primary-record address into the caller’s result slot. It then returns the original result-slot address.

Thus, the explicit publication of the primary-record address occurs after F-b00260d1087a, not before it. The function does not reinitialize either record after that call.

### Dependency interference and aliasing

The direct write sequence is established, but arbitrary noninterference is not.

For example, the second call to F-8157a180d901 occurs after primary-record initialization. Its behavior could affect that record if it has access to the storage. Likewise, F-b00260d1087a executes after both records have been initialized and linked. The caller must not assume those fields retain their assigned values unless the dependency contracts support that conclusion.

The supplied function also does not establish that returned storage is fresh, mutually disjoint, or disjoint from the result slot. If storage overlaps, later writes may replace earlier contents. Normal construction-like behavior therefore depends on storage validity and nonoverlap properties that are not established here.

## Side effects

### Direct effects

The function directly:

- Clears the caller’s result slot.
- Conditionally writes three primary-record words.
- Conditionally writes two auxiliary-record words.
- Conditionally replaces the primary record’s zero auxiliary field with an address.
- Publishes the primary-record address through the result slot after all reached dependency calls return.

It performs no explicit reads from the newly obtained record contents. The record operations shown are writes, including replacement of the auxiliary-address field.

There is no explicit access to global variables in the supplied representation. This does not imply that the dependencies are global-state-free.

### Ownership and lifetime

The function contains no explicit release operation. On normal nonzero-primary completion, it exposes the primary-record address through the caller’s slot. On the auxiliary path, it also writes an auxiliary address into the primary record.

This establishes address publication, not an ownership policy. The following remain unspecified:

- Whether the storage is dynamically allocated, pooled, borrowed, or otherwise supplied.
- Whether the caller owns either record.
- Which operation, if any, releases either record.
- Whether either dependency retains references.
- Whether F-b00260d1087a changes storage lifetime or ownership.

In particular, the later publication of the primary address does not prove that F-b00260d1087a preserved the storage.

### Errors and abnormal completion

There is no explicit error return, status field, rollback, or cleanup branch.

An unusable result-slot address can fail before any dependency call. An unusable storage-provider result can fail on a subsequent direct write. The function does not convert either condition into an empty successful result.

If a dependency does not return normally, the later publication and return do not occur. Earlier writes may already be observable. The supplied material does not specify whether abnormal completion means a trap, exception, termination, or another mechanism.

## Dependencies

### F-8157a180d901

Established use:

- Called once with 12 when the primary value is nonzero.
- Called a second time with 8 when both scalar inputs are nonzero.
- Each result is treated as an address to writable storage.
- Neither result is checked before use.

Inferred intent: a storage-provisioning operation, with requests corresponding to the apparent 12-byte and 8-byte record layouts.

Documented only: argument units, allocation mechanism, alignment, freshness, initialization, failure convention, global effects, ownership, lifetime, and preservation of existing storage. The use of the arguments as byte counts is consistent with the layouts but is not independently established by a dependency contract.

### F-b00260d1087a

Established use:

- Called exactly once on the path where both scalar inputs are nonzero.
- Receives no explicit arguments in the supplied representation.
- Runs after auxiliary initialization and attachment.
- Runs before publication of the primary-record address.
- Has no consumed return value.

Inferred intent: an ancillary action associated with enabling the auxiliary record.

Documented only: its actual purpose, implicit inputs, global reads and writes, record mutations, ownership effects, and failure behavior. It must not be assumed to preserve globals or either record.

## Acceptance examples

These examples are conditional static expectations, not executed tests. Successful-storage examples assume valid, suitably aligned, nonoverlapping storage and normally returning dependencies.

### Empty result

Given a writable result slot initially containing any value, a primary value of zero, and any auxiliary option:

- The slot is overwritten with zero.
- Neither dependency is called.
- The returned address is the original result-slot address.

A nonzero auxiliary option does not independently trigger construction.

### Primary record without auxiliary storage

Given a primary value of 37 and a zero auxiliary option, with F-8157a180d901 returning suitable storage:

- Its sole invocation receives 12.
- The primary fields are written as 37, 1, and zero, in that order.
- F-b00260d1087a is not called.
- The result slot receives the primary-record address.
- The function returns the slot address, not the record address.

### Primary record with auxiliary storage

Given a primary value of 37 and an auxiliary option of −1:

- The negative option is treated as enabled because it is nonzero.
- F-8157a180d901 receives 12 and then 8.
- The auxiliary words are zeroed before its address is attached.
- F-b00260d1087a runs after attachment.
- The final direct result-slot write publishes the saved primary address after that call returns.

If both dependencies preserve the initialized storage, the resulting primary fields are 37, 1, and the auxiliary address, with two zero auxiliary fields. Without that preservation condition, those final field contents are not guaranteed.

### Mutation by the final dependency

If F-b00260d1087a changes a record field and returns normally, this function does not restore that field. It still performs the subsequent result-slot write using the saved primary address.

### Invalid storage result

If F-8157a180d901 returns an address that cannot support the immediately following writes, the function has no explicit failure-handling branch. Acceptance must not require graceful null handling, rollback, or an error code.

## Uncertainties

The following are documented_only:

- Exact target ABI, native pointer width, and address-to-integer representation.
- Dependency implementations and their preservation guarantees.
- Whether storage requests are measured in bytes.
- Allocation failure behavior.
- Storage uniqueness, overlap, lifetime, and release rules.
- The semantic meaning of the constant primary-field value 1.
- The purpose of the auxiliary record and the final dependency call.
- Behavior under concurrent access, reentrancy, or asynchronous observation.

No synchronization or atomic-publication guarantee is established. The ordered direct operations describe the supplied representation; they do not establish a concurrency protocol.

## Evidence

Established static evidence consists of the supplied function representation: an unconditional result-slot clear, two zero/nonzero branch conditions, dependency calls with arguments 12 and 8, ordered record-field writes, a conditional no-explicit-argument dependency call, final address publication, and return of the original result-slot address.

Construction, storage allocation, and notification are inferred roles, not verified dependency semantics. No external dependency contracts, ABI documentation, or implementation access were provided. No runtime tests have been performed.
