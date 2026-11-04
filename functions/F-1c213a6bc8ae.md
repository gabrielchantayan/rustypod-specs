<!-- Trusted provenance: F-1c213a6bc8ae; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1c213a6bc8ae

## Purpose

F-1c213a6bc8ae conditionally dispatches a normalized binary mode to objects obtained through an opaque traversal mechanism, and passes the same mode to a separate dependency associated with the primary object.

The defining observable distinction is ordering:

- For a **nonzero request**, the primary-object dependency is called with mode **1 before** the primary object's eligibility field is read and before any traversal or callback.
- For a **zero request**, eligibility checking, any traversal, and traversal finalization occur **before** the primary-object dependency is called with mode **0**.

Traversal is attempted only when a 32-bit field in the primary object satisfies a specific masked comparison.

These are established control-flow properties. Interpreting the operation as enabling or disabling a parent and its children is plausible intent, not an established ownership or object-model contract.

## Interface

### Inputs

The function takes four arguments in the supplied representation:

| Argument | Representation | Established use |
|---|---|---|
| Primary object address | 32-bit integer-shaped address | Passed to F-1e945672712d; used to locate a flags word and a traversal anchor. |
| Requested mode | 32-bit integer | Tested for zero versus nonzero; copied into local traversal state before any dependency call. |
| Auxiliary value | Opaque 32-bit word | Copied into local storage. No direct interpretation or subsequent direct read is shown. |
| Initial candidate pointer | Pointer to a 32-bit-word representation | Copied into a local pointer slot whose address is supplied to F-b38a3aba0e18. |

The supplied representation uses integer-shaped object addresses and pointer-based indirect dispatch. The contract describes its apparent 32-bit address model; it does not establish portability to a different pointer width or native ABI.

The requested mode is normalized for externally visible mode arguments:

- Zero becomes 0.
- Every nonzero value becomes 1, including negative values in the signed 32-bit interpretation.

The original requested-mode value is nevertheless copied unchanged into the first local state word before normalization is used for calls. Thus, for example, requests 1 and −1 produce the same callback mode, but their initial local state words differ.

### Output

The function has no explicit return value.

It does not return a traversal count, status, previous mode, or error indication. Return values from the primary-object dependency and object callbacks are not consumed by the visible logic.

### Addressed layouts

The following locations are directly relevant:

| Base | Offset | Interpretation |
|---|---:|---|
| Primary object | 0x48, or 72 bytes | 32-bit unsigned flags word |
| Primary object | 0xA8, or 168 bytes | Address passed as the traversal anchor |
| Current candidate | 0 | Word used as the base address of an indirect-dispatch table |
| Indirect-dispatch table | 0xC4, or 196 bytes | Callable entry invoked for the current candidate |

The traversal-anchor address is passed to a dependency. This function does not itself dereference the anchor at offset 0xA8.

The object and table offsets are byte offsets. They are not element indices. On the apparent 32-bit layout, the callable entry occupies a pointer-sized location at byte offset 196; its position does not establish the meanings of any preceding entries.

### Preconditions for defined execution

For a normal, defined execution:

- The primary object must permit the flags read at offset 0x48 when that read occurs.
- F-1e945672712d must accept the primary address and normalized mode.
- If traversal is selected, the traversal dependencies must accept the supplied local-state address and anchor address.
- After every nonzero result from F-b38a3aba0e18, the candidate slot must contain a usable object pointer. That object must provide a usable table-base word, and the table must provide a callable entry at offset 0xC4.

The function performs no visible null, bounds, alignment, lifetime, or callable-entry validation.

## Behavior

### Initial local state

Before branching on the requested mode, the function initializes three local values:

1. A 32-bit word containing the original requested mode.
2. A 32-bit word containing the auxiliary value.
3. A pointer slot containing the initial candidate pointer.

The traversal initializer and finalizer receive the address of the first word. The traversal-advance dependency receives that same address and, separately, the address of the candidate slot.

The auxiliary word has no explicit consumer in the visible body. Its presence does not establish that it is irrelevant: an opaque dependency may access surrounding storage under an external layout contract. The permitted extent of such access is not established here.

### Nonzero-request ordering

For every nonzero requested mode, the function proceeds in this order:

1. Call F-1e945672712d with the primary object and integer mode 1.
2. After that call returns, read the 32-bit flags word at primary-object offset 0x48.
3. Evaluate the traversal eligibility condition.
4. If eligible, initialize, consume, and finalize the traversal.
5. Return without another call to F-1e945672712d.

The flags value used for eligibility is therefore the value available **after** F-1e945672712d returns. The function does not preserve an earlier flags snapshot.

No assumption is made that this dependency preserves the flags, traversal anchor, other object contents, or global state.

### Zero-request ordering

For a zero requested mode, the order is:

1. Read the 32-bit flags word at primary-object offset 0x48.
2. Evaluate the traversal eligibility condition.
3. If eligible, initialize, consume, and finalize the traversal.
4. Call F-1e945672712d with the primary object and integer mode 0.
5. Return.

The eligibility decision therefore precedes the primary-object mode-0 call. Changes caused by that final call do not cause a new eligibility check or another traversal within this invocation.

### Eligibility condition

Traversal is selected exactly when:

**flags AND 0x00000600 equals 0x00000200**

Equivalently:

- Bit 9 must be set.
- Bit 10 must be clear.
- All other bits are ignored by this condition.

Having bit 9 set is not sufficient if bit 10 is also set. The masked value 0x600 is ineligible, as are 0x000 and 0x400.

The function performs one explicit flags read per invocation along a normally returning path. It does not recheck eligibility between candidates.

### Traversal lifecycle

When eligible, the function first calls F-8ca21d3a8cca with:

- The address of the local state word initially containing the original requested mode.
- The primary-object address plus 0xA8.

After initialization returns, it repeatedly calls F-b38a3aba0e18 with:

- The same local-state address.
- The address of the local candidate-pointer slot.

Every nonzero result means that the function proceeds to dispatch through the candidate slot. Zero ends traversal.

The continuation result is tested only for zero versus nonzero. A negative result, if supplied by the dependency in the represented integer return type, takes the dispatch path. There is no separate visible negative-error branch.

For each nonzero result, the function:

- Uses the candidate pointer currently stored in the slot.
- Reads the table-base word at the beginning of that candidate.
- Resolves the callable entry at table-base offset 0xC4.
- Invokes that entry with the candidate pointer and the normalized mode.

The candidate is not checked against the initial pointer or earlier candidates. Duplicate candidates are not suppressed. The function also does not require the advance dependency to have changed the pointer slot: if it returns nonzero without changing that slot, the slot's existing value is used.

After a callback returns normally, the function asks F-b38a3aba0e18 for another continuation result. The callback's return value, if any, is ignored.

Once F-b38a3aba0e18 returns zero, F-d0d3f100effc is called with the local-state address. Only after that finalizer returns does the function continue to its branch-specific completion step.

There is no visible iteration limit. Normal termination depends on the advance dependency eventually returning zero and on all intervening calls returning.

### Dispatch freshness and mutation

For every dispatched iteration, candidate and callable resolution follow that iteration's successful advance call. There is no visible caching of the first candidate's table or callable entry across iterations.

Callbacks may therefore influence later observable behavior if they change state later consumed by traversal or dispatch. The function provides no visible snapshot, locking, rollback, or reentrancy guard.

## Side effects

The function's explicit writes are to local storage used for traversal state and the candidate pointer.

It does not visibly assign to the primary object's flags word, traversal anchor, candidate objects, or dispatch tables. Changes to those objects can nevertheless occur through dependencies or callbacks.

The externally significant calls are:

- Exactly one call to F-1e945672712d on every normally completing invocation.
- One traversal-initializer call if and only if the eligibility condition succeeds.
- One candidate callback for each nonzero advance result.
- One traversal-finalizer call after the first zero advance result, provided all preceding calls return normally.

When eligibility fails, the initializer, advance dependency, candidate callbacks, and finalizer are all skipped.

### Ownership and lifetime

No allocation, reference-count operation, explicit release, or ownership transfer appears directly in the function.

This does **not** establish that any dependency is allocation-free or ownership-neutral. In particular:

- The finalizer's name or position does not prove that it releases a particular resource.
- The initializer's acceptance of local storage does not establish its complete storage layout.
- Passing a candidate pointer to a callback does not establish whether the callback retains, releases, or invalidates that candidate.

Addresses of local storage are supplied to traversal dependencies. Safe retention beyond the invocation is not established; those addresses do not describe durable caller-owned state.

### Errors and abnormal completion

There is no visible error-reporting path, exception handler, retry, rollback, or guaranteed-cleanup construct.

If initialization, advancement, a candidate callback, or finalization does not return normally, the remaining normal sequence is not established. In particular:

- The traversal finalizer is not visibly guaranteed after a callback failure.
- In the zero-request branch, the primary-object mode-0 call occurs only after traversal and finalization return normally.
- In the nonzero-request branch, the primary-object mode-1 call has already occurred before any traversal failure.

## Dependencies

| Opaque function ID | Arguments supplied | Result handling and established role |
|---|---|---|
| F-1e945672712d | Primary address; normalized mode | Called once on normal completion, before eligibility for mode 1 and after traversal handling for mode 0. No result is used. |
| F-8ca21d3a8cca | Local-state address; primary address plus 0xA8 | Called once at the start of an eligible traversal. No result is used. |
| F-b38a3aba0e18 | Local-state address; address of candidate-pointer slot | Zero ends traversal; any nonzero result causes one candidate dispatch. |
| F-d0d3f100effc | Local-state address | Called after advance returns zero. No result is used. |

A further dependency is selected dynamically through each candidate's table entry at offset 0xC4. No opaque function ID is provided for that callable. Its visible call arguments are the candidate pointer and normalized mode.

These descriptions specify call relationships, not internal semantics. They do not establish global-state preservation, resource ownership, callback effects, or traversal membership.

## Acceptance examples

The following are conditional static acceptance examples, not executed tests. Each assumes valid memory and normally returning calls unless stated otherwise.

### Nonzero request with post-call eligibility

Suppose the request is 7, and the flags word is 0x200 after F-1e945672712d returns.

The required observable sequence is:

1. F-1e945672712d receives mode 1.
2. The flags word is read.
3. Traversal is initialized.
4. Each nonzero advance result causes a candidate callback with mode 1.
5. A zero advance result is followed by finalization.

The original local state word starts as 7, not 1.

### Nonzero request changes eligibility

Suppose the initial flags are 0x000, but F-1e945672712d changes them to 0x200 before returning.

Traversal is selected because the eligibility read occurs after that call. Conversely, changing flags from 0x200 to 0x600 before returning prevents traversal.

### Zero request with two candidate dispatches

Suppose flags are 0x200 and the advance results are nonzero, nonzero, then zero.

The two candidate callbacks receive mode 0. Finalization follows the zero result. Only then is F-1e945672712d called with mode 0.

### Ignored flags bits

Flags 0x201 are eligible because the additional low bit is outside mask 0x600. Flags 0x600 are ineligible even though bit 9 is set.

### Empty eligible traversal

If flags are eligible but the first advance result is zero, initialization and finalization still occur. No candidate or dispatch-table dereference is required by the visible loop body.

### Unchanged candidate slot

If advance returns nonzero without changing the candidate slot, the existing pointer is dispatched. On the first iteration, this can be the initial candidate pointer, unless initialization or another permitted dependency effect has changed the slot.

### Negative continuation result

If advance returns −1, the function treats it as continuation and dispatches through the current candidate. It does not treat that value as an error.

### Invalid candidate after continuation

If advance returns nonzero while the candidate slot contains a null or otherwise invalid pointer, the function has no visible rejection path. Defined successful completion is not established for that case.

## Uncertainties

The following remain **documented_only**:

- The semantic meaning of the primary object, flags, anchor, candidate objects, and mode operation.
- The full ABI, pointer widths outside the apparent 32-bit representation, and permissible address ranges.
- The complete traversal-state layout and whether dependencies access the auxiliary word or surrounding local storage.
- The meaning and necessity of the auxiliary value and initial candidate pointer.
- Traversal order, membership, mutation tolerance, and duplicate behavior supplied by the dependencies.
- Allocation, retention, release, global-state mutation, and ownership behavior inside dependencies and callbacks.
- Callback return type, exception behavior, reentrancy, and concurrency requirements.
- Behavior when offset address calculations exceed the valid address range. No checked arithmetic or explicit overflow handling is shown.
- Whether a nonzero advance result is intended to distinguish errors from items. The visible caller makes no such distinction.

The parent/child interpretation and enable/disable terminology are inferred intent only. They are not required to explain the established call ordering.

## Evidence

This contract is based solely on static inspection of the supplied function representation.

Established evidence includes the zero/nonzero branch, normalized call arguments, 32-bit masked flags comparison, relative object offsets, traversal call order, continuation test, indirect dispatch, and placement of finalization.

Dependency implementations and dynamically selected callbacks were not supplied. Their internal effects are not established.

No runtime tests, memory observations, or execution traces have been performed. The acceptance examples describe consequences of the visible control flow under their stated conditions; they are not empirical results.
