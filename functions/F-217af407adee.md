<!-- Trusted provenance: F-217af407adee; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-217af407adee

## Purpose

F-217af407adee traverses a sequence exposed by three opaque dependencies and stops at either:

- The first candidate whose two leading 32-bit words match a supplied ordered pair.
- The first traversal result whose 32-bit status is zero.

It invokes a finalization dependency on either normal exit path and returns a 64-bit value assembled from pointer-sized quantities. The return value must not be simplified to “the matching pointer”: its upper half is taken from a mutable output slot **after finalization**, while its lower half is either zero or the matching pointer captured **before finalization**.

The traversal-and-match behavior is established by the supplied static representation. Descriptions such as “iterator,” “candidate,” and “finalization” are behavioral labels, not claims about container type, allocation, ownership, or the dependencies’ broader purposes.

## Interface

### Inputs

The function accepts four arguments:

| Argument | Represented width | Role |
|---|---:|---|
| Context address | 32 bits | Used to construct the address passed to the initialization dependency. |
| Initial candidate-slot value | 32-bit pointer | Initially stored in a local pointer slot, then explicitly replaced with null after initialization. |
| First match value | 32 bits | Compared against the candidate word at byte offset 0. Also copied into the first local state word before initialization. |
| Second match value | 32 bits | Compared against the candidate word at byte offset 4. Also copied into the second local state word before initialization. |

The representation uses 32-bit integers and 32-bit pointer values. The return occupies 64 bits. These are properties of the supplied representation; portability to an environment with different integer or pointer widths is not established.

The initial candidate-slot argument is not an initial traversal position under the visible control flow. Although it is stored locally before initialization, the function resets that slot to null before making the first traversal call. Whether the initialization dependency can observe or alter the slot through behavior not described by its explicit arguments is unknown.

### Locally exposed storage

The visible local storage comprises:

- One 32-bit candidate-pointer slot.
- A first 32-bit state word initialized from the first match value.
- A second 32-bit state word initialized from the second match value.

The represented layout places the second state word immediately after the first, and the candidate-pointer slot immediately before the first state word. The initialization, traversal, and finalization dependencies receive the address of the first state word. The traversal dependency additionally receives the address of the candidate-pointer slot.

This establishes the storage and addresses exposed by the caller. It does not establish how much storage any dependency is entitled to access, or whether a dependency modifies neighboring storage.

### Output

The result is a 64-bit bit pattern with two 32-bit halves:

| Exit condition | Upper 32 bits | Lower 32 bits |
|---|---|---|
| Traversal returns zero | Candidate-slot contents after finalization | Zero |
| A matching candidate is found | Candidate-slot contents after finalization | Candidate pointer captured immediately after the successful traversal call |

Each pointer-derived half contributes its 32-bit representation. The failure-path upper half is explicitly zero-extended before shifting left by 32 bits. The successful path concatenates two 32-bit values.

The represented return type is a signed 64-bit integer, but the established construction is bit-oriented. If the final upper half has its most significant bit set, interpreting the entire result as signed produces a negative value. No separate meaning for that sign is established.

## Behavior

### Initialization and address construction

Before calling a dependency, the function initializes the local candidate-pointer slot from the second argument and initializes the two state words from the two match values.

It then invokes **F-8ca21d3a8cca** with:

1. The address of the first local state word.
2. The context address plus 8.

The addition is an eight-byte numeric displacement, not an eight-element displacement. The supplied representation performs the addition in a 32-bit integer expression. Behavior at arithmetic overflow, or where the resulting address is not usable by the dependency, is not established. A safe calling precondition is that this expression produces the intended address without overflow and that the dependency can validly use it.

The visible body does not directly dereference the context address. Any access to the context or its displaced address belongs to the dependency.

After the initialization call returns, the function writes null to the candidate-pointer slot. Consequently, a value left in that slot by initialization, if any, is overwritten before traversal begins.

No visible initialization result controls whether traversal starts. If initialization returns normally, the function always makes at least one traversal call.

### Traversal status and candidate capture

Each traversal attempt invokes **F-b38a3aba0e18** with:

1. The same first-state-word address.
2. The candidate-pointer-slot address.

After that call returns, the function captures the candidate-pointer slot’s current contents in a separate local value. It then decides whether the returned 32-bit status is zero.

All nonzero status values take the candidate-inspection path. The function does not distinguish positive from negative nonzero statuses and does not expose the traversal status in its own result.

The function does not explicitly clear the candidate-pointer slot between traversal calls. A subsequent traversal call therefore receives the slot as left by preceding execution. Whether traversal writes a fresh value, reuses a value, or relies on the existing value is dependency-defined.

### Zero-status exit

When traversal returns zero:

1. The function does not read either candidate word.
2. It invokes **F-d0d3f100effc** with the first-state-word address.
3. After that call returns, it obtains the candidate-pointer slot’s then-current contents.
4. It returns a 64-bit value with those contents in the upper half and zero in the lower half.

A zero traversal status does **not** by itself establish an all-zero function result. That requires the candidate-pointer slot to contain null after finalization.

The captured pre-finalization pointer is not used in this exit’s return value.

### Nonzero-status candidate inspection

When traversal returns nonzero, the function treats the candidate-pointer-slot contents as the address of a two-word record.

Candidate reads occur in this order:

1. Read the 32-bit word at byte offset 4.
2. Compare it with the original second match argument.
3. Only if that comparison is equal, read the 32-bit word at byte offset 0.
4. Compare that word with the original first match argument.

The offset-0 read is short-circuited: it is skipped whenever the offset-4 word differs from the second match argument.

The comparisons are equality comparisons on 32-bit values. There is no ordering, range comparison, masking, byte swapping, or visible normalization. The supplied representation does not establish the target’s byte order.

The match criteria remain the original argument values. Mutations to the local state words do not themselves replace those criteria.

A nonzero traversal status requires a readable candidate under the visible execution path. The function performs no explicit null check, bounds check, alignment check, or record-length check before reading the offset-4 word. If that word matches, offset 0 must also be readable.

### Mismatch and continuation

If either required equality fails, the function repeats traversal.

No candidate release, state reset, or finalization call appears between an unsuccessful comparison and the next traversal call. Traversal and any effects it performs determine how the sequence advances.

There is no iteration limit, cancellation check, or visible progress validation. If traversal continues returning nonzero while providing only nonmatching candidates, the function continues indefinitely.

### Matching exit

When both candidate words match:

1. The function retains the candidate pointer captured immediately after the traversal call.
2. It invokes **F-d0d3f100effc** with the first-state-word address.
3. It reads the candidate-pointer slot after finalization.
4. It returns the post-finalization slot value in the upper 32 bits and the retained pre-finalization pointer in the lower 32 bits.

Thus, if finalization leaves the slot unchanged, the result contains the same pointer in both halves. If finalization changes the slot, the halves differ.

No candidate data is read after finalization by the visible body. The return of pointer bits does not establish that the corresponding memory remains live or accessible.

## Side effects

The function directly writes its local candidate-pointer slot and its two local state words. It directly reads candidate memory but contains no visible direct write to candidate records or context storage.

The dependencies may introduce additional side effects. Their writes, allocations, releases, external I/O, global-state changes, and retained references are not established by this body. They must not be assumed absent.

On either normal exit path, finalization is invoked exactly once. It occurs:

- After the traversal call that returns zero; or
- After the reads that establish a matching candidate.

Finalization is not invoked between mismatching candidates.

The visible body provides no abnormal-exit handling. A fault during a candidate read, a nonreturning dependency, or another interruption is not shown to trigger finalization.

Ownership remains unspecified:

- Passing an address does not establish ownership transfer.
- A captured pointer is not an independent allocation.
- Returning pointer bits does not establish a retained reference.
- Finalization must not be assumed either to preserve or to release candidate storage.
- Dependencies must not be assumed to preserve global state.

The described ordering is the ordering of operations in the supplied static representation. It is not a claim about synchronization, atomicity, volatile access, or inter-thread memory ordering.

## Dependencies

| Opaque ID | Invocation and established role | Unspecified behavior |
|---|---|---|
| **F-8ca21d3a8cca** | Called once before traversal, with the first-state-word address and the context address displaced by 8. | Initialization protocol, state footprint, context layout, failure reporting, and side effects. |
| **F-b38a3aba0e18** | Called at least once after normal initialization. Receives state and candidate-slot addresses. A zero status ends traversal; every nonzero status permits candidate inspection. | How candidates are selected, whether the slot is always written, meaning of zero, meaning of different nonzero statuses, ownership, and progress guarantees. |
| **F-d0d3f100effc** | Called once on each normal exit, with the first-state-word address, before the final candidate-slot value is incorporated into the result. | Whether it changes the slot or other storage, releases resources, invalidates candidates, or affects global state. |

The labels “initialization,” “traversal,” and “finalization” summarize call position and visible usage. They do not substitute for dependency contracts.

## Acceptance examples

These examples are conditional behavioral checks, not reports of executed tests. Pointer names denote 32-bit address representations.

### First candidate matches

Given that initialization returns normally and traversal returns a nonzero status with candidate pointer **P**:

- The word at **P + 4** equals the second match argument.
- The word at **P** equals the first match argument.
- Finalization leaves the candidate-pointer slot equal to **P**.

The function calls traversal once and finalization once. The result has **P** in both 32-bit halves.

### Finalization changes the successful result’s upper half

Under the same matching conditions, suppose finalization changes the candidate-pointer slot from **P** to **Q**.

The result has **Q** in the upper half and **P** in the lower half. Replacing this with a duplicated **P**, duplicated **Q**, or a single pointer would violate the visible behavior.

### Immediate zero traversal status

Suppose the first traversal call returns zero and finalization leaves the candidate-pointer slot null.

The function reads no candidate words, calls finalization once, and returns zero.

### Zero status with a nonnull final slot

Suppose traversal returns zero and the candidate-pointer slot contains **Q** after finalization.

The result has **Q** in the upper half and zero in the lower half. It is nonzero if **Q** is nonzero, despite the zero traversal status.

### Second word rejects a candidate

Suppose traversal returns nonzero with pointer **P**, and the word at **P + 4** differs from the second match argument.

The function reads that word but does not read the word at **P** during this inspection. It invokes traversal again without first invoking finalization.

### First word rejects a candidate

Suppose the word at **P + 4** matches, but the word at **P** differs from the first match argument.

Both reads occur, in offset-4-then-offset-0 order. Traversal resumes without intermediate finalization.

### Negative nonzero status

Suppose traversal returns a negative 32-bit status and supplies a valid candidate.

The function inspects the candidate exactly as it would for a positive nonzero status. It does not treat negativity as a separate error condition.

### Null candidate with nonzero status

Suppose traversal returns nonzero while leaving the candidate-pointer slot null.

The visible body attempts the offset-4 candidate read. It does not convert this situation into a clean failure result. The resulting fault or platform-specific behavior is outside the established normal contract.

## Uncertainties

The following are **documented_only** unknowns:

- The actual context, state, and candidate types.
- The dependencies’ complete readable and writable memory footprints.
- Whether adjacent local-slot access by dependencies is intentional.
- Whether dependencies retain addresses of local storage after returning.
- Whether zero status means exhaustion, an error, or another stopping condition.
- Whether candidates can repeat or traversal is guaranteed to terminate.
- Candidate ownership and lifetime before and after finalization.
- The intended semantic interpretation of the two-part return value.
- Calling convention, platform byte order, and exceptional behavior.
- Exact overflow behavior of the context-address addition.
- Concurrency, reentrancy, and global-state interactions.

An iterator-like search is inferred intent. The status branch, comparison order, finalization placement, and return-half construction are established static behavior, not inferred API conventions.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping.

The representation establishes the three dependency calls, local initialization and reset, repeated traversal, zero-versus-nonzero status handling, short-circuit candidate reads, both finalization paths, and 64-bit return construction.

No dependency implementation, independent ABI description, caller contract, or runtime observation was supplied. No runtime tests have been performed.
