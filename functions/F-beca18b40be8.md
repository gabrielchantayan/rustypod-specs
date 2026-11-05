<!-- Trusted provenance: F-beca18b40be8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-beca18b40be8

## Purpose

F-beca18b40be8 performs a bounded sequence of candidate-processing attempts against a mutable object. It starts only when the object’s signed 32-bit pending-count field is positive. Each attempt prepares a candidate, invokes an object-selected callback, conditionally decrements the pending count and updates a retained result pair, then invokes a second object-selected callback to obtain the next candidate state.

The invocation performs at most **10,000 attempts**. Once the processing branch has been entered and its loop has terminated normally, the function writes the retained result pair into the object and unconditionally clears its pending count. Reaching the attempt limit therefore clears any remaining count; it does not preserve an unfinished remainder for a later invocation.

“Pending count,” “candidate,” and “result pair” describe apparent roles, not established application-level meanings. The established contract is the access and call behavior below.

## Interface

### Inputs and return

The function accepts three arguments:

| Argument | Established representation and use |
|---|---|
| Object | Address of mutable storage accessed through 32-bit fields. Its first word supplies a dispatch-table address. |
| Input-word address | Address from which one 32-bit word is initially copied. The original address is also passed to every attempt callback. |
| Auxiliary value | A 32-bit value used as the initial second retained result, as the initial first word of a temporary result area on every attempt, and as an argument to F-7551853739b9. |

The function has no explicit return value. Completion is observable through object writes and any effects of its dependencies.

### Object layout

The supplied representation establishes these byte offsets relative to the object:

| Offset | Width | Use |
|---|---:|---|
| +0 | 32 bits | Dispatch-table address |
| +8 | 32 bits | First retained-result field |
| +12 | 32 bits | Second retained-result field |
| +16 | 32 bits | Signed pending count |

No meaning is established for the word at +4 or for other object storage.

Two indirect callbacks are selected through the dispatch table:

- **Attempt callback:** slot at byte offset +20.
- **Advance callback:** slot at byte offset +24.

These are byte offsets in the supplied 32-bit representation, not slot numbers or a portable layout for another pointer width.

### Temporary storage

For an active invocation, the function uses:

- A mutable 32-bit candidate-state word.
- A 64-byte candidate workspace.
- A temporary result area exposing two adjacent 32-bit words.
- Two retained 32-bit result values.
- An unsigned 32-bit attempt counter.

The first word of the temporary result area is initialized before each attempt callback. The second word is not visibly initialized by this function before its possible consumption. Neither the initial contents of the 64-byte workspace nor complete initialization by a dependency is established.

### Preconditions

A nonpositive initial pending count takes the inactive path. On that path, this function does not access the input word or dispatch table and does not write the object.

For an active invocation, defined execution requires valid storage for every access actually made, including the input-word read, object-field reads and writes, dispatch-table reads, and dependency accesses. Compatible indirect-call targets are also required.

The supplied material does not establish non-aliasing requirements. If the input-word address aliases object storage, or dependencies can modify that storage, the actual ordering of reads and writes matters. No null checks, range validation, or recovery behavior are visible.

## Behavior

### Entry gate and initial retained values

The function first reads the pending count as a signed 32-bit value.

If it is zero or negative, the function returns without entering the processing branch. In particular, it does not clear a negative count.

If it is positive, the function:

1. Reads one 32-bit word through the input-word address and copies it into the mutable candidate-state word.
2. Reads the object field at +8 into the first retained result.
3. Initializes the second retained result from the auxiliary argument.
4. Initializes the unsigned attempt counter to zero.

The object field at +12 is **not** read to initialize the second retained result. If no attempt callback subsequently returns nonzero, the eventual write to +12 uses the auxiliary argument rather than the field’s original contents.

### Loop eligibility and limit

Before each attempt, the function checks the current object pending count again. Processing continues only while that signed value is positive and the attempt counter is below the effective limit.

The supplied unsigned limit expression admits counter values from 0 through 9,999 inclusive:

- Values below 8,192 satisfy its first alternative.
- Values from 8,192 through 9,999 satisfy its second alternative, which subtracts 8,192 and compares the result against 1,808.
- Value 10,000 fails both alternatives.

The counter starts at zero and increases once after each completed attempt. No counter wraparound is reachable during ordinary completion of this loop.

The current pending count is tested before the limit expression. Consequently, after the 10,000th completed attempt, the next loop check still reads the pending count before rejecting further processing on the counter limit.

### Ordered actions within an attempt

Each admitted attempt has the following order.

**1. Prepare candidate workspace.**

The function calls F-202f56eb285c with:

- The address of the 64-byte workspace.
- The address of the mutable candidate-state word.
- A value of 1.

Any effects of this dependency occur before the temporary result initialization and the attempt callback. Its return value is not used.

**2. Initialize the first temporary result word.**

The first word of the temporary result area is set to the auxiliary argument. This initialization occurs on every admitted attempt, regardless of whether the attempt callback will return zero or nonzero.

There is no corresponding visible initialization of the second temporary result word.

**3. Invoke the attempt callback.**

The function selects the callback through the object’s dispatch table at byte offset +20 and passes:

- The object address.
- The original input-word address.
- The candidate-workspace address.
- A value of zero.

The callback’s 32-bit result is tested for zero. Every nonzero result takes the same branch; no distinct success codes are interpreted.

**4. On a nonzero result, decrement the current count and update retained results.**

The function reads the object’s pending count at this point, subtracts one, and writes it back.

This is a read of the count **after** the attempt callback. It is not necessarily a decrement of the value seen by the loop guard. If the callback changes the count, that changed value participates in the subtraction.

After writing the decremented count, the function calls F-7551853739b9 with:

- The address of the first temporary result word.
- The current candidate-state value, passed as a 32-bit value rather than as its address.
- The auxiliary argument.

After that call returns, the function reads the second temporary result word into the second retained result, then reads the first temporary result word into the first retained result.

Thus, each nonzero attempt replaces both retained results. A later zero-returning attempt does not itself replace them. Whether both words are validly initialized depends on behavior not shown for F-7551853739b9.

**5. Invoke the advance callback unconditionally.**

After either callback-result branch, the function selects the callback through the object’s dispatch table at byte offset +24. It passes:

- The object address.
- The address of the mutable candidate-state word.

The callback’s 32-bit return value replaces the candidate-state word.

If this callback also writes through the supplied address, the subsequent assignment of its return value determines the candidate-state value retained by this function.

The advance callback is invoked even when:

- The attempt callback returned zero.
- A successful attempt reduced the pending count to zero.
- This was the 10,000th and final permitted attempt.

There is no intervening count check that skips this call.

**6. Increment the attempt counter.**

The counter increases by one, after which the loop conditions are checked again.

### Normal completion of the active branch

When the loop terminates normally, the function performs these writes in order:

1. First retained result to object offset +8.
2. Second retained result to object offset +12.
3. Zero to object offset +16.

These writes occur whether termination resulted from a nonpositive count or the attempt limit.

If no attempt callback returned nonzero, the first retained result remains the value copied from +8 at entry, and the second remains the auxiliary argument. Direct dependency writes to those object fields do not, by themselves, update the retained copies; the final writes can overwrite such changes.

No final candidate-state value is explicitly copied back through the input-word address.

## Side effects

### Direct effects

The function can directly:

- Read the input word once on entry to the active branch.
- Read the object’s dispatch-table address for indirect calls.
- Read and write the pending count.
- Read the first result field at active entry.
- Write both result fields and clear the count at active completion.
- Modify its own temporary storage.

The source does not cache a callback selection for the entire invocation. Selection occurs at the respective call sites. If a dependency changes the object’s dispatch-table address or entries, a later indirect call may select a different target.

### Dependency effects and ownership

Dependencies receive addresses of caller-provided storage and temporary storage. Their effects cannot be treated as limited to the visible arguments: no supplied dependency contract establishes preservation of globals, object fields, aliases, or memory ownership.

This function contains no explicit allocation or release operation. That does not establish that its dependencies allocate nothing, release nothing, or preserve caller-owned memory.

Addresses of temporary storage are passed to dependencies. Those addresses must not be assumed usable after the invocation returns. Whether a dependency retains them is unknown, not evidence of a supported retained-reference interface.

### Errors and abnormal completion

No explicit error result, diagnostic, exception handling, rollback, or cleanup path appears.

A zero attempt-callback result is not an error return from F-beca18b40be8. It skips the decrement and result-update helper, but still permits advancement and later attempts.

The final result writes and count clearing are guaranteed only for normal completion of the active branch. A dependency that does not return normally prevents that normal sequence from completing.

## Dependencies

| Dependency | Established call contract at this site | Not established |
|---|---|---|
| F-202f56eb285c | Receives a 64-byte workspace address, a mutable 32-bit state-word address, and 1; called once per admitted attempt | Workspace format, initialization coverage, state transformation, other side effects |
| Attempt callback, dispatch offset +20 | Receives object, original input-word address, workspace address, and zero; its result is tested for zero versus nonzero | Application meaning of the result, preservation guarantees, ownership effects |
| F-7551853739b9 | Called only after a nonzero attempt result and the count decrement; receives the temporary result address, current state value, and auxiliary value | Whether it initializes both result words, result interpretation, other side effects |
| Advance callback, dispatch offset +24 | Receives object and mutable state-word address; its returned word becomes the next state | Advancement algorithm, relationship between pointed-to writes and returned value, other side effects |

The indirect callbacks have no supplied opaque function IDs. Their dispatch locations identify the dependency sites, not fixed target identities.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence. They are not runtime observations.

### Inactive count

Given an initial count of zero or a negative signed value:

- No dependency is called.
- The input word is not read by this function.
- Neither result field nor the count is written by this function.

### One successful attempt exhausts the count

Given an initial count of 1, dependencies that otherwise preserve the count, and an attempt callback returning nonzero:

- Candidate preparation precedes the attempt callback.
- The count is decremented to zero before F-7551853739b9 runs.
- The helper-produced result words become the retained results, provided both words are defined.
- The advance callback still runs.
- Normal completion writes those retained results, then writes a zero count.

### No successful attempts

Given a positive initial count and dependencies that leave the count positive while every attempt callback returns zero:

- Exactly 10,000 attempts and 10,000 advance calls occur.
- F-7551853739b9 is never called.
- The final first result is the entry value from +8.
- The final second result is the auxiliary argument.
- The pending count is cleared despite no decrement in the nonzero-result branch.

### A zero-returning callback stops later attempts through mutation

Given that the first attempt callback changes the count to zero and returns zero, and the advance callback leaves it zero:

- No decrement or result-helper call occurs.
- The advance callback still runs once.
- The next loop check terminates processing.
- The initial retained results are written, followed by count clearing.

### Callback mutation affects the decrement

Given a positive loop-entry count, an attempt callback that sets the count to 7 and returns nonzero, and a result helper that leaves it unchanged:

- The direct decrement writes 6, regardless of the count observed by the loop guard.
- Later execution depends on the count after all intervening dependency effects.

### Last nonzero attempt determines retained results

Given multiple nonzero attempt results interspersed with zero results, with valid helper-produced result words:

- Each nonzero attempt replaces both retained results.
- Zero results do not replace those retained values.
- Normal completion commits the pair from the last nonzero attempt, not the first.

## Uncertainties

The following are **documented_only**:

- Application meaning of the object, count, state word, workspace, and result pair.
- Workspace layout and required initialized bytes.
- Whether F-7551853739b9 initializes the second temporary result word before it is read.
- Dependency preservation, ownership, allocation, release, reentrancy, and global-state effects.
- Calling-convention details beyond the supplied argument and word representation.
- Behavior under concurrent mutation of object storage.
- Exceptional or nonreturning dependency behavior.

The count decrement is represented as signed 32-bit subtraction. With an unchanged positive count, subtracting one is within range. Dependencies can change that count before subtraction; if it becomes the minimum signed 32-bit value, the supplied material does not establish a portable overflow outcome. This contract does not invent wrapping, saturation, or recovery behavior.

Names such as “pending” and “advance” are inferred intent. They do not establish that one count unit corresponds to one real resource, that the state progresses monotonically, or that candidates are unique.

## Evidence

This contract is based solely on the supplied static representation of F-beca18b40be8 and the supplied opaque dependency mapping.

Established evidence includes the signed positive-count gate, 32-bit field accesses, dispatch offsets, dependency arguments, zero/nonzero result branch, decrement placement, temporary-result read order, unconditional advance call, effective 10,000-attempt limit, and ordered final writes.

No dependency bodies, external interface documentation, or runtime observations were supplied. No runtime tests have been performed.
