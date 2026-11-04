<!-- Trusted provenance: F-fd0e502ac9ba; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fd0e502ac9ba

## Purpose

F-fd0e502ac9ba conditionally invokes a dependency with mutable temporary storage, links a selected record into an object field, and drains a sequence through two dependency calls per detached record.

These are structural descriptions. The supplied body does not establish the application meaning of the object, records, payloads, or dependency operations. In particular, draining the sequence does not by itself establish deallocation or ownership transfer.

The function returns a 64-bit value. Its low 32 bits contain the original object address. Its high 32 bits contain either the second argument or a mutable temporary sampled at the terminating head check, depending on the initial selection field.

## Interface

### Arguments

| Position | Representation | Established use |
|---|---|---|
| First | Object address, represented by 32 bits in the observed operations | Base for object accesses; original address supplies the low return component. |
| Second | 32-bit integer | Supplies the high return component when the initial selection field is zero. |
| Third | 32-bit value | Not read or forwarded by the supplied body. |
| Fourth | 32-bit value | Copied into temporary storage passed by address to F-654faddb0d33 on the nonzero-selection path. |

The evidence does not establish a portable interface for 64-bit pointers.

### Object layout

Offsets are in bytes from the first argument. Each listed access is 32 bits wide.

| Offset | Observed role |
|---|---|
| +0 | Sequence head. Zero ends traversal. |
| +4 | Link value copied into a selected record, then replaced with that record’s address. |
| +16 | Selection field. Read for the initial branch, again for a record access, and again after F-654faddb0d33. |

The body contains no direct object-relative access at +8 or +12. Those locations can nevertheless be accessed or changed through overlapping record storage or dependencies.

### Selected-record layout

The address obtained from the selection field is used for these accesses:

| Offset | Width | Observed role |
|---|---:|---|
| +8 | 32 bits | Value copied into a temporary before F-654faddb0d33. |
| +12 | 32 bits | Destination for the object’s current +4 value after that dependency returns. |

The +8 read and +12 write need not concern the same record: the function rereads selection between them.

### Sequence-record layout

Each nonzero head is treated as a record with these fields:

| Offset | Width | Observed role |
|---|---:|---|
| +0 | 32 bits | Next head value. |
| +4 | 32 bits | Second argument to the first per-record dependency call. |
| +8 | 32 bits | First argument to that call. |

These layouts describe accesses, not distinct verified types. Selected records, sequence records, and the object are not required by the visible body to occupy disjoint storage.

### Result

On normal return, the result has:

- **Low 32 bits:** the original first-argument address.
- **High 32 bits:**
  - the second argument’s bit pattern, if the initial selection field was zero;
  - otherwise, the saved selection temporary’s value sampled during the final head check.

The two components are concatenated as bit patterns. The construction is not an arithmetic addition and does not sign-extend the high component. For example, a high component of `0xFFFFFFFF` remains exactly those 32 bits.

This describes significance within the 64-bit result, not its byte order in memory.

### Preconditions for normal completion

There is no explicit validation. Every address actually used must support its corresponding 32-bit access for the execution to complete normally.

On the nonzero-selection path, this includes:

- the selected-record +8 read before F-654faddb0d33;
- the selected-record +12 write using selection reread after that call;
- the object reads and writes;
- every sequence-record read reached through the evolving head.

An initially nonzero selection does not establish that the post-call selection remains nonzero or valid. There is no second validity check before the +12 write.

No blanket non-aliasing precondition is imposed. Aliasing changes which values later operations observe and can change termination or memory-access validity.

## Behavior

### Initial selection

The function first reads object +16 and saves its 32-bit value in a temporary.

If that value is zero, the function returns with the second argument as its high return component. On this path it does not:

- read a selected record;
- read or drain the sequence head;
- invoke either dependency; or
- explicitly write caller-provided storage.

A nonzero sequence head does not alter this branch.

### Preparation and initial dependency call

If the saved selection is nonzero, the function reads object +16 again and uses that address for a 32-bit read at record +8.

It prepares three distinct temporary values:

- the saved initial selection;
- the value read from selected-record +8;
- a copy of the fourth argument.

It calls F-654faddb0d33 with these arguments, in order:

1. Address of the fourth-argument copy.
2. Original object address.
3. Address of the selected-record value copy.
4. Address of the saved selection temporary.

The three temporary addresses expose writable storage. The visible caller does not explicitly copy modifications to the fourth-argument copy or selected-record value copy back to their original sources.

Modifications to the saved selection temporary can affect the high return component. They do not directly determine the target of the following link update.

The first selection read controls branching; the second supplies the address for the +8 read. They are separate accesses. Under stable ordinary memory they agree, but the contract does not replace them with a single indivisible read.

### Link update

After F-654faddb0d33 returns, the function:

1. Rereads object +16 and retains that selected address.
2. Reads the current value at object +4.
3. Writes that value to selected-record +12.
4. Writes the retained selected address to object +4.

Both the selected address and the copied link value therefore reflect state after the dependency call. The saved selection temporary is not substituted for the object’s selection field.

The retained selected address is used for the final object +4 write even if the preceding +12 write aliases and changes object +16.

These writes occur before the first head check. Their aliasing effects are part of the behavior. In particular:

- a zero head before the link update need not remain zero afterward;
- sequence-record next or payload fields can be changed before traversal reaches them;
- the selection field itself can be overwritten by an overlapping record destination.

The function performs no overlap check or corrective rewrite.

### Sequence draining

After the link update, each loop check:

1. Reads the current object head at +0.
2. Samples the saved selection temporary.
3. Continues only if the head read was nonzero.

For a nonzero head, that address is retained as the detached record. The iteration then:

1. Reads the detached record’s next value at +0.
2. Writes that value to object +0.
3. Reads the detached record’s fields at +8 and +4.
4. Calls F-ff1bdc376dcd with the +8 value, the +4 value, and zero.
5. Calls F-ff1bdc376dcd with the retained detached-record address, one, and zero.
6. Returns to the head check.

The head write precedes both payload-field reads and both calls. The relative order of the +8 and +4 argument reads is not established by the supplied expression.

Aliasing remains significant within an iteration. If writing object +0 overlaps a detached record’s payload field, the subsequent payload read observes the resulting state, not necessarily the value present at iteration entry.

The second call uses the retained record address. It does not reread the head to choose that argument, and there is no further explicit record-field read between the two calls.

Every iteration rereads the object head after both calls return. Dependencies can therefore redirect, extend, shorten, or replenish the sequence. Traversal is not restricted to records reachable before the initial dependency call or before the link update.

### Termination and return

Normal traversal ends when a head check reads zero. The saved selection temporary is sampled during that terminating check, and that sample supplies the high return component.

If nothing changes that temporary after its initialization, the high component equals the initial selection value. If F-654faddb0d33 changes it, the changed value can be returned. Later changes through retained addresses are not ruled out by the available dependency information.

There is no iteration bound, cycle detection, or explicit error exit. A cycle or continual replenishment can prevent normal return.

## Side effects

### Direct writes

On the nonzero-selection path, the visible body writes:

- selected-record +12 once;
- object +4 once;
- object +0 once per entered iteration;
- invocation-local temporary storage.

Because addresses may overlap, a selected-record write can also change an object field or another record field. This does not constitute a separate explicit object-relative write, but its effect must be included when predicting subsequent behavior.

The body does not explicitly clear detached records’ next or payload fields. It contains no direct assignment to object +16.

### Ownership and lifetime

No allocation or deallocation operation is established by the supplied body.

The two per-record calls are consistent with processing a payload and then its containing record, but release, destruction, and ownership transfer remain inferred possibilities.

Neither dependency is assumed to:

- preserve object fields or global state;
- preserve allocations or ownership;
- free or retain any supplied object;
- discard supplied addresses; or
- return normally.

If the first per-record call invalidates the detached record, the caller still passes its retained address to the second call. The validity of that use depends on the dependency contract; no intervening record-field read is visible.

The addresses passed for the three temporaries refer to invocation-local storage. No valid lifetime for their use after this function returns is established.

### Errors and external state

There is no explicit error result, rollback, or translation of dependency failures. Dependency return values are not consumed.

If an access is invalid, a dependency does not return, or a dependency leaves memory unsuitable for a later access, the supplied body establishes no recovery behavior. Earlier writes are not explicitly reversed.

No direct global-variable access, locking, or atomic operation is visible. Dependencies may access additional state. The body establishes no concurrency guarantee.

## Dependencies

### F-654faddb0d33

Called once on the nonzero-initial-selection path, before the link update. Not called on the zero-initial-selection path.

Its arguments expose three 32-bit temporaries and the original object address. Its implementation and return contract are unavailable.

Relevant possible consequences include:

- changing the saved selection temporary and thus the eventual high return component;
- changing object +16 and thus the target of the +12 write;
- changing object +4 and thus the value copied into that target;
- changing object +0 or reachable records and thus subsequent traversal.

Even if this dependency preserves all relevant memory, the caller’s own following link writes can alter the head or record fields through aliasing.

### F-ff1bdc376dcd

Called twice for each entered sequence iteration:

| Call | First argument | Second argument | Third argument |
|---|---|---|---|
| First | Detached-record +8 value | Detached-record +4 value | Zero |
| Second | Retained detached-record address | One | Zero |

The scalar values and constants occupy 32-bit argument components in the observed interface.

The first call must return before the second is issued. Both must return before the next head check. Their meanings, ownership effects, global effects, and failure behavior remain unknown.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Preservation assumptions below apply only to their stated scenarios; they do not add a general non-aliasing requirement.

### Initial selection is zero

Given a readable object +16 containing zero and a second argument of `0x87654321`:

- Neither dependency is called.
- The head and sequence records are not examined.
- No explicit write through caller-provided storage occurs.
- The result has high component `0x87654321` and low component equal to the original object address.

### Empty sequence after the link writes

Assume valid selection storage, normally returning F-654faddb0d33, and that object +0 is zero **after both link writes**.

The first head check terminates traversal, so no F-ff1bdc376dcd call occurs. If the saved selection temporary remains unchanged, the high return component equals the initial selection.

One sufficient way to satisfy the post-link condition is a zero head preserved by the dependency and no overlap between the selected-record +12 destination and the head. A zero head before the link update alone is insufficient.

### A link write creates a nonempty head

Assume the following valid, suitably accessible storage:

- Object address: `0x00001000`.
- Initial selection: `0x00000FF4`, twelve bytes before the object.
- Initial object +4: `0x00002000`.
- Initial head: zero.
- The record at `0x00002000` has next zero, +8 value P, and +4 value Q.
- The selected-record +8 location is readable.
- Dependencies return normally and preserve relevant storage and temporaries.

The selected-record +12 destination is the object’s head. The first link write therefore changes the head to `0x00002000`. The next link write changes object +4 to `0x00000FF4`.

Traversal occurs despite the initially empty head. The head becomes zero, and F-ff1bdc376dcd receives P, Q, zero, followed by `0x00002000`, one, zero. The next check terminates. The high return component is `0x00000FF4`.

### Dependency changes selection and return companion independently

Assume F-654faddb0d33 changes object +16 to another valid selected address, changes object +4 to a new link value, and changes the saved selection temporary to `0xFFFFFFFF`.

The newly selected record receives the new link value at +12, and object +4 receives the newly selected address.

If the head is zero **after those writes**, no per-record calls occur and the high return component is `0xFFFFFFFF`. A zero head merely left by the dependency does not establish this outcome if the subsequent +12 write overlaps it.

### One record after the link writes

Assume that, after both link writes, the head addresses a valid record with next zero, +8 value P, and +4 value Q. Also assume advancing the head does not overwrite those payload fields, and dependencies preserve relevant memory and return normally.

The head becomes zero before either call. F-ff1bdc376dcd receives P, Q, zero, then the retained record address, one, zero. The next head check terminates.

### Dependency replaces the remaining sequence

Assume an iteration’s next-field read produces valid record address A. The function first writes A to the head.

If either per-record call then replaces the head with valid record address B, the next check uses B. Visiting A is not required.

## Uncertainties

The following remain **documented_only**:

- Application-level meanings and complete structure layouts.
- Dependency implementations, ownership rules, failure behavior, and global effects.
- Whether retaining temporary addresses is intended or supported.
- Which overlapping layouts are valid application inputs.
- Endianness, alignment requirements, and concurrent or volatile memory semantics.
- Address-arithmetic behavior at representation boundaries.
- Whether cycles or replenishment occur in intended use.

The evidence establishes 32-bit fields, 32-bit address-bearing values, fixed byte offsets, and a 64-bit concatenated result. It does not establish portable signed-overflow rules or address-wrap guarantees. Scenarios requiring offset computation beyond valid addressable storage have no specified successful outcome here.

Record linking and head advancement are established behavior. Cleanup, destruction, and ownership transfer are inferred intent only.

## Evidence

This contract is based on the supplied static body: its field accesses, branch condition, exposed temporary addresses, dependency argument construction, ordered writes and calls, repeated head checks, and return composition.

The alias-dependent examples follow from the selected-record +12 write occurring before traversal and from the absence of a disjointness check. The one-record example separately accounts for the head write occurring before payload reads.

No dependency body, independent type definition, caller contract, or runtime trace was supplied. No runtime tests have been performed. The acceptance examples state conditional outcomes, not observed executions.
