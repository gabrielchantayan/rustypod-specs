<!-- Trusted provenance: F-d66a32b7aeae; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d66a32b7aeae

## Purpose

F-d66a32b7aeae scans a source-associated sequence, tracks a position immediately after the most recently recognized separator, and selects one of two destination-directed operations.

The established distinction is:

- If the scan recognizes no separator, invoke F-b9acea4b06ad with the destination argument and source argument.
- If it recognizes a separator, obtain a scalar through F-d8cbf9aa79a9 and invoke F-67233aa84d28 with the destination, source, and that scalar minus the recorded separator position.

The separator values are `0x5C`, `0x2F`, and `0x3A`, corresponding to backslash, slash, and colon under an ASCII-compatible interpretation. The value `0x23`, conventionally `#`, changes how the following value is consumed and counted.

**Inferred intent:** the routine appears to select a trailing component from a separator-delimited representation while accounting for escape-like sequences. The supplied evidence does not establish that the final dependency copies a suffix, that the scalar is a length, or that the sequence is a pathname. Those interpretations are not required by this contract.

## Interface

The routine accepts four arguments and has no established return value.

| Argument | Established representation | Established use |
|---|---|---|
| First | Opaque 32-bit value | Passed unchanged as the first argument to exactly one final dependency on a normally completing invocation. Its apparent role is a destination or result handle. |
| Second | Opaque 32-bit value | Not read or forwarded by the routine. |
| Third | 32-bit integer representation, used both as an opaque argument and as an address base | Passed to the initializer and final dependency. On the separator-found branch, a four-byte value is read at byte offset 4 from this base. |
| Fourth | Opaque 32-bit value | Initially assigned to local state, then overwritten by the initializer’s result before any supplied operation reads that state. It has no established effect on the scan or final dependency arguments. |

The routine maintains an addressable, four-byte local scan-state slot. F-44362605ab79 supplies its initial effective contents. F-d3922c7fe0ec and F-13bcc08bc7a3 receive the address of this slot rather than the original source argument.

Two additional local quantities govern the scan:

- A signed 32-bit running position, initially zero.
- A signed 32-bit separator position, initially `−1`, used as the no-separator sentinel.

The source representation is otherwise opaque. The only established direct layout access is a four-byte load at source-base byte offset 4, performed only after a scan that records a separator. The available material does not specify alignment, byte order, the total object size, or the meaning of that field.

### Preconditions

For a normally completing invocation:

- Every invoked dependency must return normally.
- The initialized scan state must support the sequence of reader and adjustment calls described below.
- On a separator-found branch, the address at source-base byte offset 4 must permit the four-byte read.
- The arithmetic used for position updates and the final subtraction must have meaningful behavior for the actual execution environment.

No explicit null checks, bounds checks, object validation, or overlap checks are present. This does not establish that arbitrary arguments are valid; dependencies may impose additional requirements.

## Behavior

### Initialization

F-44362605ab79 is called once with the third argument. Its four-byte result becomes the effective scan state.

The running position starts at zero, and the separator position starts at `−1`. The fourth argument does not seed the effective scan because its local assignment is replaced by the initializer’s result.

Initialization can itself have externally visible effects. The routine does not establish that F-44362605ab79 is pure or preserves any global state.

### Outer reads and termination

The routine repeatedly calls F-d3922c7fe0ec with the address of the scan-state slot.

An outer read returning zero terminates the scan immediately. That zero does not increase the running position and does not update the separator position.

Every nonzero outer result enters one of two cases: ordinary handling or `0x23` handling. Returned values are compared as full integer results. No explicit truncation to eight bits occurs in the supplied routine.

### Ordinary values

For a nonzero outer result other than `0x23`:

- Increase the running position by one.
- If the result equals `0x5C`, `0x2F`, or `0x3A`, replace the separator position with the updated running position.
- Otherwise leave the separator position unchanged.

Consequently, an ordinary separator records the position immediately after its contribution to the running count. Later recognized separators replace earlier ones. Consecutive recognized separators therefore leave the position of the last one.

This position is not established to be a byte offset. It counts reader results according to this routine’s rules; the reader’s relationship to source storage is unknown.

### Handling `0x23`

A nonzero outer result equal to `0x23` causes an immediate second call to F-d3922c7fe0ec using the same scan-state address.

The second result determines the action:

| Second result | Running-position contribution for this outer iteration | Additional action |
|---|---:|---|
| `0x5C` | Two | No adjustment call |
| `0x2F` | Two | No adjustment call |
| `0x23` | Two | No adjustment call |
| Any other value, including zero and `0x3A` | One | Call F-13bcc08bc7a3 with the scan-state address |

Neither branch updates the separator position during this iteration.

In particular:

- A slash or backslash returned immediately after `0x23` does not become a separator in that iteration.
- A second `0x23` is consumed without initiating another nested special case.
- A colon is not among the three special second-result values. It triggers F-13bcc08bc7a3 rather than an immediate separator update.
- A zero returned by this inner read does not directly terminate the outer scan. The adjustment dependency is called, and the routine proceeds to another outer read.

The role of F-13bcc08bc7a3 is not established beyond this call pattern. Its placement suggests an undo or repositioning operation, but the contract must not assume that the next reader call repeats the second result. Any such behavior belongs to the dependency contract.

### Final dispatch without a recorded separator

If the scan ends while the separator position remains `−1`, the routine calls F-b9acea4b06ad with:

1. The first argument.
2. The third argument.

It then returns.

On this branch, the routine does not directly load the four-byte source field at offset 4, does not call F-d8cbf9aa79a9, and does not call F-67233aa84d28.

“No recorded separator” is determined by the sentinel comparison, not by a separate Boolean flag. The correspondence between the sentinel and absence of separators relies on position arithmetic not overflowing into the sentinel value.

### Final dispatch with a recorded separator

If the scan ends with a separator position other than `−1`, the routine:

1. Reads four bytes from source-base byte offset 4.
2. Passes that value to F-d8cbf9aa79a9.
3. Takes the signed 32-bit result returned by that dependency.
4. Subtracts the recorded separator position.
5. Calls F-67233aa84d28 with the first argument, third argument, and computed difference.

There is no explicit clamp or rejection if the difference is zero or negative. The routine does not check whether the scalar returned by F-d8cbf9aa79a9 is at least the separator position.

The field read occurs after the scan and all its dependency calls. It is not an initial snapshot. If those dependencies alter the source object or related global state, the final branch uses the subsequently read field value.

The final running position is not passed to the final dependency. Only the last recorded separator position participates in the subtraction.

## Side effects

The directly established operations, in order, are:

1. Initialization through F-44362605ab79.
2. Repeated reads through F-d3922c7fe0ec.
3. Conditional state-adjustment calls through F-13bcc08bc7a3.
4. Either:
   - F-b9acea4b06ad; or
   - a four-byte source-field read, F-d8cbf9aa79a9, and F-67233aa84d28.

The routine directly writes its local scan state and counters. The reader and adjustment dependencies receive a writable address to that local state and may change it.

No direct destination-memory write is established outside the final dependency calls. The first argument’s destination-like role does not prove any particular allocation, copying, formatting, or replacement behavior.

There is no explicit allocation, deallocation, cleanup, or release call attributable to this routine. This is not evidence that dependencies allocate nothing or release nothing.

Ownership and lifetime guarantees are **documented_only**:

- The routine does not explicitly transfer or reclaim ownership.
- Whether dependencies retain arguments, retain the scan-state address, allocate storage, or release existing storage is unknown.
- No aliasing guarantee is established between the first and third arguments.
- Global-state preservation cannot be assumed for any dependency.

No explicit error result, diagnostic, retry mechanism, or recovery branch is present. Reader result zero is treated as outer-scan termination, but the evidence does not distinguish ordinary end-of-input from an error encoded that way.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-44362605ab79 | Called once with the third argument. Returns the four-byte value used to initialize effective scan state. |
| F-d3922c7fe0ec | Receives the address of the four-byte scan-state slot and returns an integer. Called for every outer scan step and additionally after each outer `0x23`. Outer result zero ends scanning. |
| F-13bcc08bc7a3 | Receives the scan-state address after an inner result other than `0x5C`, `0x2F`, or `0x23`. Its return value is not used. |
| F-b9acea4b06ad | Final operation when the separator sentinel remains unchanged. Receives the first and third arguments. |
| F-d8cbf9aa79a9 | Called only after a separator is recorded. Receives the four-byte source field read at offset 4 and returns the scalar used in the final subtraction. |
| F-67233aa84d28 | Final operation after a separator is recorded. Receives the first argument, third argument, and computed scalar difference. |

No dependency’s internal validation, object layout, error handling, ownership behavior, or global effects are established by the supplied material.

## Acceptance examples

These are conditional contract examples, not executed tests. Unless stated otherwise, arithmetic is representable, dependencies return normally, and the stated reader results occur in the stated order.

1. **Immediate termination.**  
   The first outer read returns zero. The routine calls F-b9acea4b06ad with the first and third arguments. It does not directly read the source field at offset 4.

2. **Ordinary nonseparators only.**  
   Outer results are `0x41`, `0x42`, then zero. The running position reaches two, but no separator position is recorded. Final dispatch is F-b9acea4b06ad.

3. **One ordinary slash.**  
   Outer results are `0x41`, `0x2F`, `0x42`, then zero. The recorded separator position is two. If F-d8cbf9aa79a9 returns three, F-67233aa84d28 receives one as its third argument.

4. **Latest separator wins.**  
   Outer results are `0x2F`, `0x41`, `0x3A`, `0x42`, then zero. The recorded separator position changes from one to three. A scalar result of four produces a final third argument of one.

5. **Slash immediately following `0x23`.**  
   An outer read returns `0x23`; its inner read returns `0x2F`; the next outer read returns zero. The running position becomes two, but no separator is recorded. No F-13bcc08bc7a3 call occurs, and final dispatch is F-b9acea4b06ad.

6. **Colon following `0x23`, conditional on adjustment semantics.**  
   An outer read returns `0x23`; its inner read returns `0x3A`. The routine increases the running position by one and invokes F-13bcc08bc7a3. If the next outer read then returns `0x3A`, that outer read records separator position two. This example does not assert that the adjustment dependency necessarily causes that repeated result.

7. **Zero from an inner read.**  
   An outer read returns `0x23`; its inner read returns zero. The routine counts one, invokes F-13bcc08bc7a3, and performs another outer read. If that next outer read returns zero, scanning ends with no separator recorded.

8. **Zero or negative final difference.**  
   A separator position of three and scalar result of three produce a final third argument of zero. A scalar result of two produces `−1`. Neither value is rejected locally; downstream acceptance is unknown.

9. **Unused arguments.**  
   Changing only the second or fourth argument does not change any established dependency argument or branch condition in this routine. This observation does not imply that dependencies are free of unrelated external-state effects.

## Uncertainties

The following remain **documented_only**:

- The semantic types of the first and third arguments.
- The encoding and storage unit represented by each reader result.
- Whether the scan-state value is a pointer, index, handle, or other representation.
- The precise effect of F-13bcc08bc7a3, especially after an inner zero.
- Whether F-d8cbf9aa79a9 returns a length and whether its units match the running position.
- The operation performed by either final dependency.
- Validity rules for empty, null, malformed, aliased, or overlapping inputs.
- Dependency ownership, allocation, retention, release, and global-state behavior.
- Distinction between termination and reader failure.
- Behavior when signed 32-bit additions or subtraction overflow. The supplied representation contains no explicit widening, saturation, or overflow handling; portable mathematical results outside the representable range must not be assumed.
- Termination if dependencies repeatedly return nonzero values or continually reposition the scan.

## Evidence

The established behavior is derived solely from the supplied static routine representation and opaque dependency mapping: argument use, integer widths, comparisons, local state transitions, direct field access, branch conditions, and dependency-call order.

Path-component interpretation, escape intent, length interpretation, and repositioning semantics are inferences, not established dependency behavior.

No runtime tests have been performed. The acceptance examples specify conditional observable behavior rather than report execution results.
