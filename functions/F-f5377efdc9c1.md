<!-- Trusted provenance: F-f5377efdc9c1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f5377efdc9c1

## Purpose

F-f5377efdc9c1 initializes a two-word output and, for each item in an input collection, obtains a byte, constructs a small descriptor, invokes two dependencies, and conditionally lowers the first output word and raises the second.

The established behavior is an ordered sequence of reads, calls, and conditional writes. Interpreting the result as a minimum/maximum interval is reasonable, but conditional: the dependencies must produce valid candidate words and must not independently interfere with the output in ways that invalidate that interpretation.

The first output word starts from a mutable global seed. The second starts at zero. The global seed’s value and semantic meaning are not supplied.

## Interface

### Inputs

The function receives three arguments:

| Argument | Role | Established accesses |
|---|---|---|
| Collection base | Address-like value identifying a collection-bearing object | A signed 32-bit count is read at byte offset 8. The address at byte offset 4 is passed to F-93ea0dab0e54. |
| Configuration base | Address-like value identifying configuration storage | A 16-bit quantity is read at byte offset 14 on every executed iteration, after the byte-producing dependency returns. |
| Output address | Writable storage for two adjacent unsigned 32-bit words | Word 0 is at byte offset 0; word 1 is at byte offset 4. |

The first two arguments appear as integer-valued addresses in the supplied representation. Their exact public pointer types, calling convention, and address-overflow rules are not established. The field widths and offsets above are supported by the supplied operations.

The collection count is captured once, before either output word is initialized. It is not reread from the collection during iteration.

### Outputs

There is no explicit return value.

The caller-visible result consists of writes to the two output words, together with any effects performed by the dependencies. The function does not return the descriptor, dependency result, or per-iteration scratch storage.

### Preconditions

For defined ordinary operation:

- The collection count location must support a 32-bit read.
- The output must support reads and writes of two consecutive 32-bit words.
- If the captured count is positive, the configuration field must support a 16-bit read on every iteration.
- Each dependency must accept the addresses, index values, and argument values passed to it.
- The scratch-producing dependency must establish valid candidate words before this function reads them.
- All effective addresses and storage accesses must satisfy the execution environment’s addressing, alignment, and lifetime requirements.

No null checks, length checks, alignment checks, or accessible-memory checks are visible.

For a nonpositive count, this function does not access the configuration field and does not call any dependency. The collection count read and output writes remain required.

## Behavior

### Initialization and iteration bounds

The observable initialization order is:

1. Read the signed 32-bit collection count at offset 8.
2. Read the global 32-bit seed and store it into output word 0.
3. Store zero into output word 1.
4. Process indices beginning at zero while the index is less than the captured count.

A zero or negative count causes no iterations. The function leaves the initialized output pair in place, absent external interference.

For a positive count, the function processes exactly that many indices if every dependency returns normally and no invalid memory operation interrupts execution. Indices are consecutive signed 32-bit values from zero through one less than the count.

The loop’s increment does not intrinsically require signed overflow: even for the largest positive signed 32-bit count, the final increment reaches that count and the next comparison terminates iteration.

### Per-item byte acquisition

For each index, the function first calls F-93ea0dab0e54 with:

- The collection base advanced by 4 bytes.
- The current index.

The result is used as an unsigned 8-bit value. Any wider result bits are not part of the byte subsequently placed in the descriptor.

The configuration field is read after this call returns. Consequently, changes made by F-93ea0dab0e54 to configuration storage can affect the descriptor for the same iteration.

### Descriptor construction

A temporary descriptor has the following established 12-byte layout:

| Byte offset | Width | Value supplied for the iteration |
|---|---:|---|
| 0 | 32 bits | The constant 16,777,216, numerically hexadecimal 01000000 |
| 4 | 16 bits | The acquired byte, zero-extended to 16 bits |
| 6 | 16 bits | The 16-bit value read from configuration offset 14 |
| 8 | 32 bits | Zero |

The source representation establishes the following preparation order:

1. Set the 32-bit field at offset 8 to zero.
2. Read the configuration quantity and establish the two adjacent 16-bit fields at offsets 4 and 6.
3. Set the 32-bit field at offset 0 to 16,777,216.
4. Pass the descriptor’s address to F-9a770d6f49ee.

The adjacent 16-bit fields are represented as a combined value with the byte-derived value in its low 16 bits and the configuration quantity in its high 16 bits. The supplied overlapping field representation places them at offsets 4 and 6 respectively. It does not establish a portable external serialization format or justify reversing the byte order of the constant at offset 0.

The descriptor is temporary working storage. Its fields are re-established on each iteration. Any descriptor modifications made by a dependency are not independently returned to the caller.

### Intermediate result and scratch processing

F-9a770d6f49ee is called with the descriptor address. Its 32-bit result is stored in a separate temporary slot.

The function then calls F-202f56eb285c with:

- An address identifying temporary scratch storage.
- The address of the 32-bit result slot.
- The integer value one.

Passing the result by address is significant. F-202f56eb285c receives access to a slot containing the result, rather than merely receiving the result value.

After this call returns, the function reads two unsigned 32-bit candidate values from storage at byte offsets 20 and 24 relative to the scratch address.

The supplied representation does not initialize these two candidate locations before the call. Correct production of their values therefore depends on F-202f56eb285c or other behavior not shown here. A failure by that dependency to establish valid values is not detected locally.

### Output updates

After the scratch-processing call, the function advances the loop index and then performs two ordered comparisons:

1. Compare the candidate at scratch offset 20 against the current output word 0. If the candidate is smaller under unsigned 32-bit ordering, replace output word 0 with that candidate.
2. Compare the current output word 1 against the candidate at scratch offset 24. If output word 1 is smaller under unsigned 32-bit ordering, replace output word 1 with that candidate.

Equality does not cause a replacement write.

The second comparison occurs after any replacement of output word 0. This ordering matters where storage aliases or dependency effects make the accesses observable.

Neither candidate is treated as signed. For example, the value with all 32 bits set is larger than zero and larger than every other unsigned 32-bit value.

### Conditional aggregate interpretation

If dependencies establish both candidate words on every iteration and do not otherwise modify the output, the final result is:

- Output word 0: the unsigned minimum of the initial global seed and all first candidates.
- Output word 1: the unsigned maximum of zero and all second candidates.

This interpretation is conditional, not an unconditional guarantee about opaque dependencies. In particular, the first output is not necessarily the minimum of the candidates alone: a seed smaller than every candidate remains the result.

There is no explicit validation that the first candidate is less than or equal to the second, nor that the final output words describe a nonempty interval.

## Side effects

### Direct effects

The function directly:

- Reads the collection count once.
- Reads the global seed once during initialization.
- Writes both output words during initialization.
- Reads the configuration field once per executed iteration.
- Writes temporary descriptor and result storage.
- Reads candidate storage and the output words during aggregation.
- Conditionally writes output word 0, then output word 1, per iteration.

No direct allocation, deallocation, logging, explicit error reporting, or synchronization is visible.

### Aliasing and mutable state

No non-aliasing requirement is enforced locally.

If the output overlaps the collection count, the count used for iteration is still the value captured before output initialization. If the output overlaps the configuration field, initialization or an earlier output update may change the value read on a later descriptor construction.

The configuration value is not cached across iterations. A dependency’s changes to it can therefore affect subsequent reads.

The seed is not reread by this function after initialization. Later global-seed changes do not cause an explicit refresh of output word 0.

Dependencies may have access to globals, caller storage, or other shared state. Their preservation of that state is unknown. In particular, output comparisons use the output words present after the dependency calls, not a separately retained aggregate immune to dependency writes.

### Ownership and lifetime

No ownership transfer is explicitly expressed.

The descriptor address, scratch address, and address of the intermediate result slot identify temporary storage used during the invocation. Safe retention of those addresses after the function returns is not established.

The fact that the intermediate result is stored in a 32-bit slot does not establish whether it represents an integer, pointer, handle, or other token. The function itself performs no visible release operation for it. Neither ownership nor release behavior may be attributed to either dependency without additional evidence.

### Errors and interruption

There is no explicit status return or failure branch around any dependency.

If execution is interrupted after initialization, the output may already contain initialized or partially aggregated values. No rollback or transactional guarantee is visible.

Faults, nonlocal exits, dependency-specific error conventions, and invalid-access behavior are not defined by the supplied function body.

## Dependencies

| Opaque ID | Invocation | What is established | What remains unknown |
|---|---|---|---|
| F-93ea0dab0e54 | Once per iteration, with collection base plus 4 and the current index | Its result supplies an 8-bit descriptor value | Collection interpretation, bounds handling, side effects, and failure behavior |
| F-9a770d6f49ee | Once per iteration, with the temporary descriptor address | Its 32-bit result is stored for the next dependency | Descriptor semantics, result interpretation, ownership, mutations, and failure behavior |
| F-202f56eb285c | Once per iteration, with scratch storage, the intermediate result’s address, and one | Candidate words are subsequently read at scratch offsets 20 and 24 | Required scratch extent, guarantees of initialization, token handling, ownership effects, and failure behavior |

The call order is fixed for every executed iteration:

F-93ea0dab0e54, then F-9a770d6f49ee, then F-202f56eb285c, followed by aggregation.

No dependency may be assumed to preserve globals, refrain from mutating reachable storage, or release memory merely because the caller does not show those actions.

## Acceptance examples

These are conditional acceptance cases derived from static evidence. They are not runtime test results.

### Nonpositive count

Given a readable count of zero or a negative signed 32-bit value, a readable seed, and valid output storage:

- Output word 0 receives the seed.
- Output word 1 receives zero.
- No dependency is called.
- The configuration field is not read.

### Single item with ordinary candidates

Given a count of one, seed 100, and a scratch-processing result whose candidates are 12 and 40, with dependencies otherwise leaving the output unchanged:

- The byte-producing dependency receives index zero.
- Output word 0 becomes 12.
- Output word 1 becomes 40.

### Seed limits the first aggregate

Given a count of two, seed 5, and candidate pairs of 12/40 and 8/30, under the same noninterference assumptions:

- The final output is 5/40.
- The first result is not 8, because the seed participates in the minimum.

### Unsigned comparison boundary

Given a count of one, seed equal to the largest unsigned 32-bit value, and candidates 2,147,483,648 and 4,294,967,295:

- Output word 0 becomes 2,147,483,648.
- Output word 1 becomes 4,294,967,295.

These values are not treated as negative.

### Equality suppresses replacement

Given a count of one, seed 9, and candidates 9 and zero, with dependencies leaving the output unchanged:

- Initialization writes still occur.
- Neither conditional replacement write occurs.
- The final pair remains 9/0.

### Descriptor field widths

Given an acquired byte of 255 and a configuration field of 46,608:

- The descriptor’s 16-bit field at offset 4 is 255, not a sign-extended negative value.
- Its 16-bit field at offset 6 is 46,608.
- Its 32-bit fields at offsets 0 and 8 are 16,777,216 and zero.

### Configuration changes during byte acquisition

If F-93ea0dab0e54 changes the configuration field before returning, and no intervening change occurs, the descriptor uses the changed field value for that same iteration.

### Unordered candidate pair

Given seed 100 and a single candidate pair of 80/20, with valid candidate production and no output interference, the final pair is 80/20. This function does not reject or reorder it.

## Uncertainties

The following are **documented_only**:

- The global seed’s actual value, intended sentinel role, and initialization policy.
- The collection’s full layout and the meaning of its offset-4 subobject.
- The semantic meanings of the descriptor constant and configuration quantity.
- The public argument types, exact ABI, alignment rules, and address-arithmetic behavior.
- The full scratch-storage extent and F-202f56eb285c’s write guarantees.
- Whether the intermediate 32-bit result is a pointer, handle, numeric value, or ownership-bearing token.
- All dependency error conventions and side effects beyond the call interface shown.
- Concurrency guarantees and behavior under unsynchronized external mutation.
- Whether overlapping caller storage is supported by the surrounding API.
- The intended domain meaning of the two final words.

An extrema calculation over per-item measurements is **inferred intent**. The unsigned comparison behavior, initialization, ordering, and field accesses are **established static evidence**.

## Evidence

This contract is based solely on the supplied static function representation and dependency-ID mapping.

Established evidence includes:

- The count read precedes output initialization.
- The count comparison is signed; candidate aggregation comparisons are unsigned.
- The global seed and zero initialize the two output words.
- Every executed iteration makes the three dependency calls in the stated order.
- Descriptor field widths, offsets, constants, and preparation order are visible.
- Candidate reads occur at scratch offsets 20 and 24.
- Output replacement is conditional and ordered.
- No explicit error handling, ownership release, or candidate-pair validation is present.

No runtime tests have been performed. Dependency implementations and external type definitions were not available; their behavior is not asserted beyond the visible interface.
