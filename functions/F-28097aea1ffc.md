<!-- Trusted provenance: F-28097aea1ffc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-28097aea1ffc

## Purpose

F-28097aea1ffc conditionally invokes two object-associated dependencies, then performs a sequence of indirect dispatches selected through a global table-address location.

A zero primary input causes immediate return. For a nonzero primary input, an object field determines whether the two direct dependency calls occur. The function then reads the global table address twice: the first value is retained for later dispatches; the second value selects the first optional target. There is no visible call between those two global reads, but their values are not established to be equal.

The call pattern is compatible with lifecycle or resource-management behavior. That interpretation is inferred intent, not an established contract. The evidence does not establish destruction, resource release, locking, notification semantics, or ownership transfer.

This contract describes the supplied static representation. It does not supply missing behavioral contracts for dependencies or claim runtime verification.

## Interface

### Incoming values

The supplied representation exposes four declared argument positions:

| Position | Represented width | Established use |
|---|---:|---|
| Primary input | 32 bits | Tested against zero; otherwise used as an object address and passed to indirect targets. |
| Second input | 32 bits | No explicit use is visible. |
| Third input | 32 bits | No explicit use is visible. |
| Fourth input | 32 bits | Passed to the first optional indirect target, if selected. |

The primary input is represented as a signed integer. Its initial test is equality with zero, not a positive-value check. A nonzero bit pattern is not rejected because its signed interpretation is negative; subsequent address validity remains a separate precondition.

An additional machine-state value appears as the fifth represented argument to the first optional target. Its represented width is **32 bits**. Its origin, ABI role, semantic meaning, and status as an API parameter are unresolved. It must not be promoted to a declared fifth API input without further evidence.

### Object layout

For a nonzero primary input, the following object-relative locations are visible:

| Byte offset | Access width | Use |
|---|---:|---|
| +4 | 32 bits | Initial condition, first dependency argument, and fresh second dependency argument. |
| +8 | 32 bits | Second argument to the first dependency, only on the initially nonzero-field path. |

No access at offset +0 is visible. The complete object size, type, packing, alignment requirements, and ownership model are not established.

### Global and table layout

A global location supplies represented 32-bit table addresses. It is read twice after the object-associated calls have completed or been skipped.

The resulting addresses have distinct roles:

| Address source | Table-relative location | Represented access | Use |
|---|---:|---:|---|
| First global read, retained | +0x18 | 32-bit callable address | Mandatory indirect dispatch. |
| First global read, retained | +0x30 | 32-bit callable address | Final optional indirect dispatch. |
| Second global read | +0x30 | 32-bit callable address | First optional indirect dispatch. |

The offsets are byte offsets: +0x18 is 24 bytes and +0x30 is 48 bytes. These are partial layouts, not evidence that the tables contain only those entries.

The first and second global reads may produce the same address under a stability assumption. The function itself does not establish that assumption.

### Outputs

No explicit return value is established. Dependency return values are not visibly consumed.

Observable behavior consists of reached memory reads, dependency invocations, argument values, their ordering, and effects caused by the invoked dependencies. No explicit write to the object, global table-address location, or table entries appears in the supplied representation.

### Preconditions for normal completion

For each reached operation:

- Object words must be readable when loaded.
- The global table-address location must be readable for both reads.
- The table address from the second global read must permit the initial +0x30 entry read.
- The retained address from the first global read must permit the later +0x18 and +0x30 entry reads.
- Every nonzero optional target must be callable with the represented arguments.
- The mandatory +0x18 target must be callable; it is not checked for zero.
- Dependencies must return normally for subsequent steps to be reached.

Neither global table-address value is checked for zero. Required storage lifetime extends to the actual time of each access, not merely to function entry.

## Behavior

### Zero primary input

If the primary input is zero, the function returns immediately.

This path performs no visible object-field read, global table-address read, or dependency invocation. The remaining declared inputs do not alter this result.

### Conditional object-associated calls

For a nonzero primary input, the function reads the 32-bit word at object offset +4 and compares it with zero.

If that initial word is zero, both direct dependency calls are skipped. Object offset +8 is not read by this function on that path. Execution continues to the global reads and indirect dispatches.

If the initial word is nonzero:

1. F-EXT-01 is invoked with values loaded from object offsets +4 and +8, in that argument order.
2. After F-EXT-01 returns, object offset +4 is loaded again.
3. F-5990a26eaaa3 is invoked with that newly loaded value.

The initial condition read is distinct from the argument reads. The supplied representation does not establish the relative evaluation order of the +4 and +8 loads used to prepare F-EXT-01.

The later +4 read occurs after F-EXT-01 returns. Its value need not equal either the initial tested word or the value passed as F-EXT-01’s first argument. In particular, if F-EXT-01 changes the field to zero and leaves it readable, F-5990a26eaaa3 receives zero. There is no repeated condition suppressing that call.

The function does not visibly clear either field.

### Two global reads

After both object-associated calls return, or after that path is skipped, the function reads the global table-address location and retains the resulting address.

It then reads the same global location again. The address from this second read is used to read the first optional target at table offset +0x30.

The established ordering is:

- First global read, producing the retained address.
- Second global read.
- +0x30 target read through the second address.

There is no visible dependency call between these global reads. Nevertheless, equality of their results is not established without an additional stability assumption. The contract does not infer a synchronization mechanism, exclude external mutation, or assume ordinary-memory behavior sufficient to collapse them into one read.

Both reads occur after the object-associated dependencies. Those dependencies may therefore affect either observed global value. Their contracts do not establish global-state preservation.

### First optional indirect dispatch

The first optional target is selected from +0x30 of the table addressed by the **second** global read.

If the selected callable address is zero, this invocation is skipped.

If it is nonzero, F-IND-01 is invoked with five represented arguments:

| Position | Value |
|---|---|
| First | Primary input |
| Second | Zero |
| Third | The selected callable address itself |
| Fourth | Fourth declared input |
| Fifth | Additional 32-bit machine-state value |

The third argument is the target address, not the table base and not the address of the table entry.

The exact source-level prototype and ABI interpretation are unresolved. The representation supports these argument positions without establishing their semantic names or purposes.

### Mandatory indirect dispatch

After F-IND-01 returns, or immediately if it was skipped, the function reads the callable address at +0x18 from the table addressed by the **first**, retained global value.

F-IND-02 is invoked with the primary input as its represented argument.

There is no zero check for this target. A zero entry does not cause an intentional skip; the represented behavior attempts the indirect invocation.

This entry is read after the first optional dispatch. Consequently, changes made by F-IND-01 to the retained table’s +0x18 entry can affect target selection. Replacing only the global table-address value does not replace the retained address.

### Final optional indirect dispatch

After F-IND-02 returns, the function reads +0x30 from the table addressed by the retained first global value.

If the entry is zero, the function returns. Otherwise, F-IND-03 is invoked with the represented arguments zero and one. No further work is visible after that transfer.

The initial and final +0x30 selections are separate reads and may use different table bases. Their targets can therefore differ even without a mutation to either table entry, if the two global reads supplied different addresses.

No third global table-address read is visible. Later changes to that global location do not themselves redirect either retained-address selection. Changes to the retained table’s entries may affect later selections.

## Side effects

### Direct effects

The visible body performs reads and calls, with no explicit updates to the represented object or dispatch data.

It does not visibly:

- Clear object offset +4 or +8.
- Replace the global table address.
- Change a table entry.
- Set a completion flag or error status.
- Allocate or release memory directly.

This is not a claim that the overall operation is read-only. Every invoked dependency may have side effects whose contracts are unavailable.

### Ownership and lifetime

No ownership transfer is established for the object, its field values, either table address, or any resource represented by those values.

The object must remain readable through every reached field access. On the initially nonzero-field path, this includes the +4 read after F-EXT-01 returns. If that dependency invalidates the object, the following read has no established safe outcome.

The retained table must remain suitable for its later entry reads even if a dependency replaces the global selector. The function shows no lifetime protection or synchronization.

Whether dependencies consume, retain, release, or invalidate arguments is documented_only.

### Errors and abnormal completion

There is no visible explicit error return, validation-failure branch, exception handler, retry, rollback, or guaranteed cleanup path.

Invalid object storage, invalid table storage, or an invalid callable address can prevent normal completion. The exact platform response is not established.

If a dependency does not return normally, later operations are not guaranteed to occur. In particular, the final optional dispatch is not established as failure-safe cleanup.

## Dependencies

| Opaque identifier | Selection and represented invocation |
|---|---|
| F-EXT-01 | Direct external dependency, invoked only when the initial object +4 word is nonzero; receives values from object +4 and +8. |
| F-5990a26eaaa3 | Mapped direct dependency; receives a fresh object +4 value after F-EXT-01 returns. |
| F-IND-01 | First optional role; selected from second-global-read table +0x30 and invoked with five represented arguments. |
| F-IND-02 | Mandatory role; selected from retained-first-global-read table +0x18 and invoked with the primary input. |
| F-IND-03 | Final optional role; selected from retained-first-global-read table +0x30 and invoked with zero and one. |

F-EXT-01 and the indirect-role identifiers are document-local opaque labels. They do not establish recovered identities.

The indirect identifiers distinguish invocation roles, not necessarily different functions. Their callable addresses may coincide.

Dependency side effects, return behavior, memory ownership, global preservation, and reentrancy properties remain unknown.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests. Unless an example concerns failure, assume all reached accesses and calls are valid and dependencies return normally.

### Zero primary input

Given a zero primary input, the function returns without object reads, global reads, or dependency calls.

### Initially zero object field

Given a nonzero primary input whose initial +4 word is zero:

- F-EXT-01 and F-5990a26eaaa3 are skipped.
- Object +8 is not read.
- Both global reads still occur.
- The mandatory target is selected from the table address obtained by the first global read.

### Field changed to zero

Given an initially nonzero +4 word, if F-EXT-01 changes that field to zero while preserving readable storage, F-5990a26eaaa3 is invoked with zero. It is not skipped.

### Global reads agree

If both global reads return the same table address, all three indirect selections use that table base. The two +0x30 entry reads remain separate and occur on opposite sides of the mandatory invocation.

### Global reads differ

If the first global read yields one table address and the second yields another:

- The first optional target is selected from +0x30 of the second table.
- The mandatory target is selected from +0x18 of the first table.
- The final optional target is selected from +0x30 of the first table.

No intervening visible call is required by this criterion. It is conditional on the differing read results; the mechanism producing them is not established.

### First optional target changes global selection

If F-IND-01 replaces the global selector, later target selections still use the retained first-read address. If it instead modifies entries in that retained table, those modifications can affect later dispatches.

### Optional target appears only at the final selection

If the initial optional selection yields zero, but the retained table’s +0x30 entry is nonzero when the final selection occurs, the first optional invocation is skipped and the final one occurs with arguments zero and one.

This can result from different table bases or from an intervening entry change; neither explanation is required universally.

### Mandatory entry is zero

If the retained table’s +0x18 entry is zero when read, the function does not treat it as optional. It attempts the indirect invocation. The resulting platform behavior is documented_only.

## Uncertainties

### Established static evidence

The representation establishes:

- A zero-primary-input early return.
- Conditional direct calls based on the initial object +4 word.
- A fresh +4 read after F-EXT-01.
- Two ordered global reads after the direct-call stage, with no visible intervening call.
- First optional selection through the second global value.
- Mandatory and final selections through the retained first global value.
- An unchecked mandatory invocation.
- A 32-bit represented additional machine-state value.
- No explicit represented object, table, or global writes.

### Inferred intent

The sequence may represent lifecycle work with optional surrounding hooks. That interpretation is not contractual. Terms such as destruction, release, lock, unlock, or completion notification would add unsupported semantics.

### Documented-only unknowns

The following remain documented_only:

- Exact ABI and source-level prototypes.
- Meaning of the fourth declared input.
- Origin and ABI role of the additional 32-bit value.
- Whether the final indirect transfer is a conventional call or a tail transfer.
- Actual identities of unmapped dependencies.
- Dependency ownership and global-state behavior.
- Global-read stability, synchronization, concurrency, and reentrancy guarantees.
- Full object and table layouts and required alignment.
- Fault behavior and supported address ranges.

The representation uses 32-bit values and addresses with fixed byte-offset addressing. No application-level arithmetic beyond these offsets is visible. The evidence does not establish a supported wraparound rule for address addition, safe handling of signed overflow, or valid accesses near address-space boundaries.

## Evidence

This contract is derived from the supplied static function representation and opaque dependency mapping.

The representation supports the distinction between the first retained global read and the second read used for initial optional selection. It also establishes the additional machine-state value’s represented 32-bit width, while leaving its meaning unresolved.

The dependency mapping identifies F-5990a26eaaa3. Other dependency labels here are opaque local identifiers rather than recovered identities.

Indirect-call prototypes and final-transfer mechanics are not fully established by the supplied material. Those limits are retained rather than resolved by assumed intent.

No runtime execution, smoke checks, or tests have been performed.
