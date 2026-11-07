<!-- Trusted provenance: F-0f32acfe4c6d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0f32acfe4c6d

## Purpose

F-0f32acfe4c6d conditionally traverses a counted collection of fixed-stride records associated with an object. For each visited record, it can invoke a two-call preparation sequence and a selector-controlled operation. It then examines two 32-bit fields, either of which can lead to a dependency annotated as non-returning. Paths that continue perform ordered clearing writes. Normal traversal completion clears two object-level bytes.

**Inferred intent:** the arrangement resembles conditional cleanup. Neither resource types nor ownership, destruction, or memory-release semantics are established.

**Evidence boundary:** this contract describes the supplied static representation. Dependency internals and runtime behavior are unavailable. In particular, the non-returning property is an annotation, not an independently verified fact. No runtime tests have been performed.

## Interface

### Inputs and result

The function has four input positions and no explicit result.

| Input | Established use |
|---|---|
| Object address | Represented by a 32-bit value and used to form object and record addresses. |
| Second input | A 32-bit value with no observed use in this body. |
| Third input | A 32-bit value with no observed use in this body. |
| Fourth input | A 32-bit value copied into an addressable local slot only when the entry gate is nonzero. |

No explicit result means that the function provides no visible status value. It does not imply that every invocation returns: dependencies may terminate execution or otherwise fail to return.

### Observed memory layout

All offsets below are byte offsets. Object-level fields are addressed directly from the object address.

| Object offset | Access width | Role |
|---|---:|---|
| `0x468` | 32 bits, unsigned | Traversal count, read at each loop condition. |
| `0x73c` | 8 bits | Entry gate; cleared after normal traversal completion. |
| `0x73e` | 8 bits | Additional byte cleared after normal traversal completion. |

Records use a stride of `0x3c`, or 60 bytes. For unsigned index \(i\), the accessed address is the object address plus \(60i\) plus the indicated offset.

| Record offset | Access width | Role |
|---|---:|---|
| `0x485` | 8 bits | First preparation condition. |
| `0x484` | 8 bits | Second preparation condition, read only when the first is nonzero. |
| `0x49c` | 32 bits | Primary value: an optional dependency argument, a zero/nonzero decision, and a clearing destination. |
| `0x4a4` | 8 bits | Operation selector; exactly `1` selects a dependency call. |
| `0x488` | 32 bits | Secondary zero/nonzero decision and clearing destination. |

“Primary,” “secondary,” and “selector” are descriptive labels, not established application types. A 32-bit value passed to a dependency is not thereby proven to be a pointer, allocation, or owned resource. Endianness is not established by these accesses.

### Preconditions

No explicit null, alignment, capacity, address-overflow, or count validation is visible.

Execution requires valid storage for every access actually reached and satisfaction of the invoked dependencies’ preconditions. When the gate is zero, only the gate access is reached; neither a readable count nor readable records are required by this body on that path.

The record stride does not establish allocation capacity. Safe operation requires that all reached address calculations identify appropriate storage. Behavior for invalid addresses or violated dependency preconditions is **documented_only**.

## Behavior

### Entry gate

The first explicit object-memory operation is an 8-bit read at `0x73c`.

If the observed byte is zero, the function returns. It does not read the count, initialize the fourth-input slot, access records, invoke dependencies, or clear `0x73e`.

If the byte is nonzero, the function copies the fourth input into a local 32-bit slot and starts traversal at unsigned index zero.

The slot is initialized once per enabled invocation. It is not reinitialized for each record.

### Traversal and arithmetic

Before each record, the current index is compared with a newly read unsigned 32-bit count at `0x468`. A record is processed only while the index is strictly less than that observed count.

The count is not captured once at entry. A dependency can change it and thereby change which later iterations are reached. A count change does not independently interrupt the record currently being processed; the next count-based decision occurs after that record completes.

Each completed record is followed by an index increment and another count read. The index is 32-bit unsigned. Index `0xffffffff` cannot be less than any unsigned 32-bit count, so ordinary progression through this comparison does not execute an iteration at that index or increment it back to zero.

Record-address calculations are a separate limitation. The supplied representation uses 32-bit address-related values. Multiplication by 60 and addition of the object address and field offsets can exceed that width. No overflow protection is visible. This contract does not establish valid storage or portable execution semantics for overflowing address calculations.

### Preparation conditions and calls

For a visited record, the function first reads the byte at `0x485`.

- If it is zero, the byte at `0x484` is not read, and both preparation calls are skipped.
- If it is nonzero, the byte at `0x484` is read.
- If both observed bytes are nonzero, F-360ac6ff643e receives the local-slot address, the object address, and the current unsigned index.
- If F-360ac6ff643e returns, F-9b718979506d receives the address of the same slot.

The second call does not depend on the slot’s contents. It follows the first whenever the first returns.

Neither preparation-condition byte is explicitly cleared by this function. The dependencies may change them or other state.

Changes to the local slot persist for later preparation calls within this invocation. The fourth input is not copied again. The body does not establish whether the dependencies read, overwrite, or retain the slot address.

### Selector-controlled operation

After preparation returns, or immediately when preparation is skipped, the function reads the selector byte at `0x4a4`.

If and only if the observed value is exactly `1`, it reads the 32-bit primary value at `0x49c` and passes that value to F-fceaba74ea88.

Other selector values, including nonzero values other than `1`, do not select this call. A zero primary value does not suppress the call when the selector equals `1`.

Preparation precedes the selector read. Consequently, preparation dependencies can affect which selector value is observed and which primary value is passed.

If the selected dependency returns, the function reads the primary field again for the next decision. The argument used for F-fceaba74ea88 is not reused as that decision’s value.

### Primary-value decision

The function reads the 32-bit primary field at `0x49c` and tests it for zero.

For a nonzero value:

1. F-1bf9985dc121 is called without explicit arguments.
2. If it returns, F-feee9156a257 is called with the saved primary value.

The argument to F-feee9156a257 is the value read before F-1bf9985dc121. The primary field is not reread to construct that argument, even if the intervening dependency changes it.

F-feee9156a257 is annotated as non-returning. Under that annotation, this branch does not reach this function’s subsequent primary, selector, or secondary clearing writes. It also does not reach another record or final object-byte clearing. Dependencies may independently write those locations; the body establishes no preservation guarantee.

When the primary decision observes zero, the function performs:

1. A 32-bit zero write at `0x49c`.
2. An 8-bit zero write at `0x4a4`.

These writes are performed even though the primary field was just observed as zero. If the non-returning annotation is inaccurate, continuation from the annotated call is not established by the supplied representation and remains **documented_only**.

### Secondary-value decision

After the primary and selector clearing writes, the function reads the 32-bit secondary field at `0x488`.

For a nonzero value:

1. F-de99289388aa is called without explicit arguments.
2. If it returns, F-feee9156a257 is called with no explicit argument shown at this call site.

The secondary value is tested, but the representation does not show it being passed to F-feee9156a257. Its presence as an implicit argument must not be assumed.

Under the non-returning annotation, the secondary clearing write, later records, and final object-byte clearing are not reached on this branch.

When the secondary decision observes zero, the function writes a 32-bit zero to `0x488`. It then advances the index and evaluates the next traversal condition.

### Normal traversal completion

When the unsigned loop condition fails, the function performs two ordered byte writes:

1. Zero to object offset `0x73c`.
2. Zero to object offset `0x73e`.

It then returns without an explicit result.

The entry gate is not reread as an authorization condition during traversal. A dependency changing it does not itself cancel processing or suppress the final writes.

An initially nonzero gate and an initial count of zero still cause slot initialization followed by both final writes. No record is visited.

## Side effects

### Explicit writes and ordering

On paths reaching them, the function explicitly writes:

- The fourth input into one local 32-bit slot.
- Zero to a record’s primary field.
- Zero to that record’s selector byte.
- Zero to that record’s secondary field.
- Zero to the object gate and then the additional object byte after traversal completes.

For a completed record, the explicit record-write order is primary, selector, then secondary. The secondary decision read occurs between the selector write and the secondary write.

The function does not explicitly write the traversal count or either preparation-condition byte. Dependencies may write any state permitted by their actual behavior; this contract imposes no preservation assumption on them.

### Partial completion and later mutation

Processing is incremental. Earlier completed records have received the specified explicit clearing writes before later records are processed. No rollback is visible in this function.

Those facts do **not** establish that earlier records remain cleared. A later dependency can modify an earlier record before returning or terminating. The contents of earlier records after subsequent dependency calls are unknown unless separate dependency guarantees are supplied.

Under the non-returning annotation:

- A nonzero primary decision prevents this function’s three explicit clearing writes for that record.
- A nonzero secondary decision occurs after this function’s primary and selector clearing writes, but prevents its secondary clearing write.
- Either branch prevents this function’s final object-byte writes.

These statements describe which writes this function reaches, not the final contents of memory after dependency execution.

### Ownership and lifetime

The local slot has invocation-local storage. Two dependencies receive its address. Whether either retains that address, and whether doing so satisfies its own lifetime requirements, is unknown. The body supplies no guarantee of storage lifetime beyond the invocation.

No ownership transfer, allocation, release, or destruction operation is established. Dependency argument shapes and call placement are insufficient to infer those semantics.

## Dependencies

| Opaque ID | Visible invocation |
|---|---|
| F-360ac6ff643e | Local-slot address, object address, and current unsigned index when both preparation conditions are nonzero. |
| F-9b718979506d | Same local-slot address after F-360ac6ff643e returns. |
| F-fceaba74ea88 | Current 32-bit primary value when the selector equals exactly `1`. |
| F-1bf9985dc121 | No explicit arguments after a nonzero primary decision. |
| F-de99289388aa | No explicit arguments after a nonzero secondary decision. |
| F-feee9156a257 | Annotated non-returning; receives the saved primary value at one call site and has no explicit argument shown at the other. |

No dependency result is visibly consumed. Returning dependencies can nevertheless affect later behavior through the slot, object memory, other reachable memory, or shared state.

No dependency is assumed to preserve globals, object contents, earlier records, or the traversal count. No dependency is assumed to own or release memory.

## Acceptance examples

These are conditional static expectations, not executed tests. All reached accesses are assumed valid. For each example, dependencies return and preserve relevant state except where a mutation or the non-returning annotation is expressly part of the example.

### Disabled object

With a zero gate at `0x73c`, the function returns without reading the count, accessing records, invoking dependencies, or clearing `0x73e`.

### Enabled object with zero count

With a nonzero gate and count zero, the local slot receives the fourth input. No record is visited. The function writes zero to `0x73c`, then to `0x73e`, and returns.

### Record requiring only clearing

With count one, byte `0x485` zero, selector zero, and both 32-bit fields zero:

- Byte `0x484` is not read.
- No dependency is invoked.
- The primary field, selector, and secondary field receive zero writes in that order.
- The two final object-byte writes follow.

### Shared preparation slot

With two visited records having both preparation conditions nonzero, the preparation pair runs for each record. If the first pair changes the slot, the next pair receives the same slot address containing the then-current value, not a fresh copy of the fourth input.

### Exact selector and primary reload

With selector `2`, F-fceaba74ea88 is skipped. With selector `1` and primary value zero, it receives zero.

If that dependency changes the primary field, the subsequent primary decision uses the newly read value. A change from nonzero to zero permits the clearing path; a change from zero to nonzero selects the primary nonzero branch.

### Saved primary argument

If the primary decision reads a nonzero value, F-1bf9985dc121 precedes F-feee9156a257. If the former changes the primary field and returns, the latter still receives the saved decision value.

Under the non-returning annotation, this function performs no subsequent record-clearing or final object-byte writes on that path. Dependency writes remain unconstrained.

### Secondary nonzero branch

With a zero primary decision and nonzero secondary decision, the primary and selector zero writes precede F-de99289388aa. F-feee9156a257 follows if that dependency returns, with no explicit argument shown.

Under the annotation, this function does not perform the secondary zero write.

### Changed count

If a dependency changes the count to zero and the current record completes, the next condition ends traversal. The final object-byte writes occur. No immediate interruption of the current record is implied.

### Earlier record modified by a later dependency

Suppose the first record completes and receives all three clearing writes. During processing of a later record, a dependency writes a nonzero value into the first record and then terminates execution.

This is compatible with the established body. The first record received the clearing writes, but its contents did not remain zero. No rollback or preservation guarantee follows from the earlier completion.

## Uncertainties

The following remain **documented_only**:

- Actual dependency behavior, including termination, failures, memory effects, and shared-state effects.
- The true non-returning behavior and calling interface of F-feee9156a257.
- Any implicit argument at its secondary call site.
- Meaning and ownership of the two 32-bit record fields.
- Allocation capacity and intended count limits.
- Platform behavior for invalid, unaligned, or overflowing addresses.
- Concurrency, synchronization, reentrancy, and signal interaction.
- Retention and lifetime requirements for the local-slot address.
- Memory contents after dependencies that may mutate previously processed records.
- Error-reporting conventions beyond the visible calls and control flow.

The body exposes no explicit status result, error-code write, or recovery mechanism. Failure behavior must not be inferred from the cleanup-like structure.

## Evidence

Established static evidence consists of the supplied representation’s access widths and offsets, 60-byte stride, unsigned count comparison, conditional reads, visible dependency arguments, repeated primary-field read, saved primary argument, and ordered clearing writes.

The body establishes that writes occurred on reached paths; it does not establish preservation of their results across later dependency calls.

Cleanup intent is an inference. Non-returning behavior is an annotation. Dependency internals, ownership rules, surrounding allocation constraints, and runtime behavior are unresolved.

No runtime tests have been performed.
