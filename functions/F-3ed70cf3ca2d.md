<!-- Trusted provenance: F-3ed70cf3ca2d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3ed70cf3ca2d

## Purpose

F-3ed70cf3ca2d coordinates a multi-stage operation over a mutable context:

1. Capture an original position and a separate phase boundary.
2. Invoke a setup dependency.
3. Process a fixed number of indexed entries.
4. Replace the active position and another context field with stored values.
5. Process from the replacement position toward the captured phase boundary.
6. If necessary, use a planning dependency and count-controlled processing to reach the original position.
7. On normal completion, restore the active position to the captured original position.

The supplied static evidence establishes the control flow, context accesses, dependency arguments, and return-value propagation. It does not establish what the positions measure, what the indexed entries represent, or whether processing involves storage, allocation, or external I/O.

Inferred intent: the function appears to reconcile or replay work from a stored position toward an original endpoint. That interpretation is not part of the required behavioral contract.

## Interface

### Input and result

The function accepts one context address and returns a 32-bit integer.

The context must provide accessible storage for every field read or written along the executed path. No null-address check, extent check, alignment check, or context validation is visible.

Dependency status values are tested only for zero versus nonzero. Nonzero values are propagated without a visible mapping to a different error domain. A nonzero result can therefore represent any dependency-defined status; its meaning is unknown.

### Context layout

Offsets are byte offsets relative to the context address. All listed fields are four bytes wide.

| Offset | Contract role | Interpretation |
|---|---|---|
| +0x2C | Setup argument | Opaque 32-bit value |
| +0x34 | Working field replaced during initialization | Opaque 32-bit value |
| +0x38 | Indexed-entry count | Signed 32-bit integer |
| +0x40 | Arithmetic parameter | Signed 32-bit integer |
| +0x70 | Position-processing argument | Opaque 32-bit value |
| +0x74 | Indexed-processing argument | Opaque 32-bit value |
| +0x98 | Active position, low word | Unsigned 32-bit word |
| +0x9C | Active position, high word | Signed 32-bit word |
| +0xA8 | Optional phase boundary, low word | Unsigned 32-bit word |
| +0xAC | Optional phase boundary, high word | Signed 32-bit word |
| +0xB0 | Replacement value for +0x34 | Opaque 32-bit value |
| +0xB8 | Replacement active position, low word | 32-bit word |
| +0xBC | Replacement active position, high word | 32-bit word |

A position consists of a low word and a high word. Position ordering uses signed high-word ordering and unsigned low-word ordering, equivalent to signed 64-bit ordering of the paired words. The low word is at the lower context offset; byte endianness within each word is not established here.

The function captures two positions before calling any dependency:

- **Original position:** the initial contents of +0x98 and +0x9C.
- **Phase boundary:** the initial contents of +0xA8 and +0xAC, unless both words are zero, in which case the original position is used.

A boundary with only one zero word is not treated as absent. In particular, an explicitly stored all-zero boundary cannot be distinguished from the fallback sentinel.

### Ownership and lifetime

The function does not explicitly allocate or release memory.

It supplies F-9838ae3a5083 with addresses of two temporary four-byte storage areas. One is subsequently read as a signed 32-bit count. The other is not read by this function. Their lifetime is limited to this invocation.

No evidence establishes that dependencies preserve unrelated context fields, avoid global mutation, or refrain from retaining supplied addresses. Correct use requires dependency behavior compatible with the temporary storage lifetime.

## Behavior

### 1. Capture positions and invoke setup

The initial visible field reads capture, in order:

1. Original low word at +0x98.
2. Original high word at +0x9C.
3. Boundary high word at +0xAC.
4. Boundary low word at +0xA8.

The boundary-zero test includes another read of +0xAC and, if that word is zero, another read of +0xA8. If the test succeeds, the captured original position replaces the captured boundary.

This is not an atomic snapshot of the context.

The function next reads +0x2C and calls F-11effff0926d with the context and that value. Its result becomes the initial retained status. The result is not immediately checked, and a nonzero setup result does not itself stop processing.

After this call, the function reads +0x38 once to obtain the indexed-entry count. Thus setup-side changes to +0x38 affect this phase.

### 2. Process indexed entries

For a positive captured count, entries are processed in ascending index order, starting at zero and stopping before the captured count. Zero or negative counts skip this phase.

For each entry:

- The arithmetic parameter at +0x40 is read again.
- An offset is formed from the entry index multiplied by the arithmetic parameter plus four.
- F-5569dd87c920 receives the context, the current value at +0x74, the offset’s low word, its sign-extension high word, and mode zero.

Any nonzero result is returned immediately. No subsequent initialization or original-position restoration is performed by this function on that path.

Every successful indexed call replaces the retained status with zero. Consequently, at least one successful indexed call discards the setup status, even if setup returned nonzero.

The captured entry count is not reread. The arithmetic parameter and +0x74 argument are reread for each call, so dependency mutations to those fields can affect later entries.

The arithmetic expression uses 32-bit signed intermediates in the supplied representation. An overflow-free contract requires the addition and multiplication to remain representable. Behavior outside that range is documented_only; a portable overflow or truncation rule is not established.

### 3. Install replacement state

After the indexed phase completes or is skipped, the function performs these assignments in order:

1. Read +0xB8 and write its value to +0x98.
2. Read +0xBC and write its value to +0x9C.
3. Read +0xB0 and write its value to +0x34.

These replacement values are read at this stage, not captured at function entry. Earlier dependencies may therefore affect them.

The position replacement consists of separate word writes. No atomic update or rollback is visible.

### 4. Process toward the phase boundary

The function compares the current active position against the captured phase boundary.

While the active position is strictly below that boundary, it calls F-5569dd87c920 with:

- The context.
- The current value at +0x70.
- The current active-position low and high words.
- Mode one.

A nonzero result is returned immediately without the function restoring the original position.

After a zero result, the retained status is set to zero and the active position is compared against the boundary again. The function itself does not advance the position between these calls.

Once the active position is at or above the boundary, control moves permanently to the next phase. The phase boundary is not checked again, even if a later dependency moves the active position backward.

Termination therefore depends on dependency behavior when this loop is entered. A dependency that repeatedly returns zero without making sufficient progress can cause an unbounded loop.

### 5. Reconcile with the original position

The function repeatedly compares the active position against the captured original position.

If the active position is already at or above the original position, the function:

1. Writes the captured original low word to +0x98.
2. Writes the captured original high word to +0x9C.
3. Repeats those two writes if the retained status is zero.
4. Returns the retained status.

This is an unconditional clamp to the original position on the normal completion path. A position above the original is reduced to the original; equality still produces writes.

If the active position is below the original, the function calls F-9838ae3a5083 with:

- The context.
- The high-word component of current position minus original position, including the low-word borrow.
- The captured original low word.
- The captured original high word.
- The address of a temporary signed 32-bit count.
- The address of a second four-byte temporary area.

A nonzero result is returned immediately. The function performs no original-position restoration on that error path.

On a zero result, the dependency-produced count is read. A successful dependency call must initialize that output before it is consumed. No such requirement can be inferred for the second temporary area, whose contents are not read here.

If the count is nonzero, it is used directly. If it is zero, the function invokes F-7071f254f784 to obtain a replacement count. Its four arguments are:

- The low word of original position minus the active position, read after the planning call.
- The corresponding high word, including the low-word borrow.
- The 32-bit value of the current +0x40 field plus eight.
- The sign-extension high word of that value.

The result of F-7071f254f784 is used as a signed 32-bit count. It is not checked as a separate error status.

The count-controlled processing then resets the retained status to zero and decrements the working count before each iteration decision. For an ordinary positive count of N, it permits at most N calls to F-5569dd87c920. Each permitted call additionally requires the active position still to be below the original position.

These calls use the current +0x70 argument, current active-position words, and mode one. A nonzero result returns immediately without restoration.

After the count is exhausted, or the original position is reached or passed, reconciliation begins again with a fresh active-position comparison. If the position remains below the original, another planning call occurs.

An ordinary zero or negative count permits no position-processing call during that batch. Repeated nonpositive counts without other progress can therefore cause repeated planning calls indefinitely. Decrementing the minimum signed 32-bit count crosses the signed range; its behavior is documented_only rather than assigned a portable meaning.

## Side effects

Direct context writes are limited to:

- Installing the replacement active position.
- Replacing +0x34 from +0xB0.
- Restoring or clamping the active position to the original on normal completion.

The function does not restore +0x34. It does not undo successful earlier dependency operations when a later call fails.

All dependency mutations remain observable unless subsequently overwritten. In particular:

- Captured original and boundary positions do not change when context fields change.
- Active-position checks use context state observed during processing, not a private advancing cursor.
- Early nonzero returns preserve the state left by prior operations and the failing dependency.
- Normal completion guarantees the captured original words are written, but not restoration of the rest of the context.

The supplied expressions do not establish a total order among every argument-field read within an individual dependency call. The phase ordering and explicit sequence of context assignments are established.

No thread-safety, reentrancy, volatile-access, synchronization, or concurrent-mutation guarantees are established.

## Dependencies

| Opaque ID | Observed use | Required distinction |
|---|---|---|
| F-11effff0926d | Setup using context and +0x2C | Result is retained initially, not checked immediately |
| F-5569dd87c920 | Indexed mode-zero processing and position-based mode-one processing | Zero permits continuation; nonzero returns immediately |
| F-9838ae3a5083 | Planning below the original position; receives two temporary output addresses | Nonzero returns immediately; zero leads to reading the count output |
| F-7071f254f784 | Supplies a count when the planner returns a zero count | Return is consumed as a signed count, not as a status |

No dependency’s internal algorithm, memory ownership, global-state behavior, or complete side effects are established.

In particular, the arithmetic argument pattern of F-7071f254f784 is not sufficient evidence to require division, rounding, saturation, or a particular result for a zero arithmetic parameter.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Positions below are small nonnegative values unless stated otherwise.

1. **Zero boundary selects the original position.**  
   Given original position 20 and both boundary words zero, the first position-processing phase uses 20 as its boundary. If replacement state is already at 20, the count is zero, and setup returned zero, no processing or planning call is required. The replacement +0x34 value is installed, the original position is written twice, and zero is returned.

2. **A setup failure can be returned only after state replacement.**  
   Given a skipped indexed phase, setup result 7, and a replacement position at or above both the phase boundary and original position, the replacement fields are installed before return. The original position is written once and the result is 7.

3. **Successful indexed processing discards the setup result.**  
   Given setup result 7 and one indexed entry whose processing result is zero, the retained status becomes zero. If later state requires no position-processing calls, normal completion returns zero rather than 7.

4. **Indexed failure prevents replacement-state installation.**  
   Given a nonzero result 9 from the first indexed call, the function returns 9 immediately. It does not perform the explicit writes from +0xB8, +0xBC, or +0xB0. Any setup or dependency mutations remain.

5. **Boundary crossing is checked after each successful processing call.**  
   Given replacement position 5, phase boundary 8, original position 10, and successful mode-one calls that each advance by one, first-phase processing receives positions 5, 6, and 7. At position 8, reconciliation begins.

6. **A planner count limits a batch, not the whole reconciliation.**  
   Continuing from position 8 with original position 10, suppose the planner returns count two and each successful processing call advances by one. Calls at positions 8 and 9 reach 10. Normal completion then restores position 10 and returns zero.

7. **An already-overshot position is clamped.**  
   Given original position 10, phase boundary 8, replacement position 12, and zero retained status, no position-processing or planning call is needed. The final active position is 10, not 12.

8. **A planning error does not trigger restoration.**  
   Given a position below the original and planner result 13, the result is 13. The function does not overwrite the active position with the original after that error.

9. **Low-word ordering is unsigned.**  
   With equal high words, low word 0x80000000 is greater than low word 0x7FFFFFFF. With different high words, signed high-word ordering takes precedence.

## Uncertainties

The following are documented_only:

- Semantic units and valid ranges of all context fields.
- Full signatures and side effects of the dependencies.
- Whether dependency calls advance, rewind, or leave the active position unchanged.
- Whether the count-producing dependency performs arithmetic division.
- Behavior for overflowing signed intermediate arithmetic.
- Behavior if a successful planner fails to initialize its count output.
- Effects of concurrent mutation, invalid storage, or incompatible dependency aliasing.
- Ownership or retention behavior inside dependencies.
- Error-code meanings and any external recovery requirements.

Progress, bounded execution, and restoration after errors are not guaranteed.

## Evidence

The supplied static body establishes the field offsets, word widths, captured positions, fallback boundary rule, dependency call sites, zero/nonzero branches, explicit writes, and return paths described above.

The position-ordering interpretation follows from low-word borrow handling combined with signed high-word subtraction and overflow correction.

The high-level reconciliation purpose is inferred intent. Dependency internals, domain meanings, and arithmetic edge semantics not established by the supplied body remain unknown.

No runtime tests have been performed.
