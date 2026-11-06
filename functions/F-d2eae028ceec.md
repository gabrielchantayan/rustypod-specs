<!-- Trusted provenance: F-d2eae028ceec; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d2eae028ceec

## Purpose

F-d2eae028ceec conditionally performs an operation for either of two distinguished selector values, 0 and 1. It calls a preliminary dependency for every selector. The selector, a flag byte, and the preliminary result determine whether it returns immediately or proceeds through preparation, submission, association, and a conditional final action.

The established contract concerns argument values, memory-access order, dependency-call order, and return-field construction. The application meanings of the operation, tables, contexts, and selectors are not established.

The 64-bit return contains two independently selected 32-bit fields. Its upper field is either the original selector or the address of temporary storage belonging to this invocation. That address is not an established durable output buffer.

## Interface

### Arguments

| Position | Representation | Established use |
|---|---|---|
| First | 32 bits | No explicit use in the supplied body. |
| Second | 32-bit pointer-shaped value | Passed unchanged to the preliminary dependency; compared numerically against address value 2; values 0 and 1 also select table offsets. |
| Third | 32 bits | No explicit use in the supplied body. |
| Fourth | 32 bits | Copied into a temporary 32-bit slot before the preliminary call. On the submission path, the slot is overwritten with zero before its address is passed onward. |

The second argument is called the **selector** below. The routine does not directly dereference it as an address. Its pointer-shaped representation does not establish that arbitrary values are acceptable to the preliminary dependency.

### Return layout

| Bits | Contents |
|---|---|
| 31–0 | A selected 32-bit result pattern. |
| 63–32 | The original selector bits on immediate-return paths, or the temporary slot’s address on every path reaching submission. |

The halves are concatenated. A low result such as `0xFFFFFFFF` occupies only bits 31–0; it does not sign-extend over the upper field.

The upper field changes to the temporary address before submission, regardless of whether submission subsequently returns zero or nonzero. The routine returns the address bits, not the slot’s contents.

### Preconditions for normal completion

The body contains no explicit argument validation, memory-access recovery, or dependency-failure recovery. Normal completion requires, as applicable:

- The preliminary dependency accepts the selector and constant 11.
- The flag-table base and selected byte are readable for selector 0 or 1.
- Address calculations and the addresses consumed by dependencies are valid under the execution environment’s rules.
- Both independently obtained entry-table bases support their respective accesses.
- Every invoked dependency returns normally and meets any external requirements.

These are conditional requirements, not checks performed by this routine. No alignment, bounds, ownership, or lifetime validation is visible.

## Behavior

### 1. Entry and preliminary call

The fourth argument is copied into a temporary 32-bit slot. The routine then calls F-c5cf4e75bd67 with:

1. The original selector.
2. The 32-bit constant 11.

This call occurs for every selector, before any flag-table access.

The dependency returns 64 bits. Its lower 32 bits control entry into the longer path in one branch. Its upper 32 bits supply the low return field on certain immediate-return paths. Neither half is inherently a success code under this contract.

### 2. Selector classification

The selector is compared as a pointer-shaped numeric address against address value 2. The distinguished values are 0 and 1.

For every other selector, the routine returns immediately:

- Low field: the preliminary result’s upper 32 bits.
- High field: the original selector.

There is no explicit flag-table or entry-table access on this path. The preliminary dependency has nevertheless already run and may have its own effects.

### 3. Flag decision

For selector 0 or 1, the routine obtains the flag-table base and reads one byte at the selector’s byte offset. This happens after the preliminary dependency returns, so that dependency’s changes to the base or contents can affect the read.

A nonzero flag byte causes an immediate return:

- Low field: zero.
- High field: the original selector.

The preliminary result’s two halves do not affect this branch’s returned fields.

If the flag byte is zero, the routine checks the preliminary result’s lower 32 bits:

- Zero: return its upper 32 bits in the low field, with the original selector in the high field.
- Nonzero: proceed to preparation.

Any nonzero 32-bit pattern passes this condition, including patterns that would represent negative signed integers.

### 4. Preparation

The routine calls F-unknown-01 with one 32-bit argument, zero.

It adds 4 to the returned 32-bit integer value and passes the resulting address-shaped value to F-b37968c9f644, together with the preliminary result’s lower 32 bits.

The visible arithmetic is 32-bit, not a widened address calculation. There is no explicit overflow, null, bounds, or alignment check. The supplied evidence does not establish a valid-address guarantee or portable signed-overflow behavior.

The result of F-b37968c9f644 is forwarded to submission without a local success check.

### 5. Submission

Before calling the submission dependency, the routine selects the temporary slot’s address as the eventual high return field and writes zero into that slot. The original fourth-argument value is therefore not the slot value presented to submission.

F-cd7e1905512f receives six arguments, in order:

1. The result of F-b37968c9f644.
2. Zero.
3. Zero.
4. The 32-bit constant 39.
5. The temporary slot’s address.
6. The 32-bit constant 1.

The dependency can access the slot through the supplied address. Whether it reads, writes, or retains that address is not established.

If submission returns zero, the routine returns:

- Low field: decimal 28.
- High field: the temporary slot’s address.

No association, follow-on, or final-action dependency is called on this branch. The meaning of 28 beyond its literal result value is unknown.

Any nonzero 32-bit submission result enters the next phase.

### 6. Context association: two independent base reads

After nonzero submission, the routine calls F-unknown-02 with no arguments.

After that dependency returns, the routine performs these accesses in order:

1. Reads the global entry-table base and saves that value for a possible later access.
2. Independently reads the same global entry-table base again.
3. Reads the selected 32-bit entry using the base obtained by the second read.

The entry’s byte offset is 0 for selector 0 and 4 for selector 1.

There is no explicit dependency call between the two global-base reads. They are nevertheless distinct reads with distinct uses. The first supplies the saved base for the optional final action; the second supplies the base for the association entry. Equality of their values requires a stability assumption not established by the supplied body.

The routine calls F-23aadb16b673 with:

1. The context returned by F-unknown-02.
2. The entry read through the second global-base value.

No result from this association call is used.

### 7. Follow-on result and optional final action

The routine next calls F-256e8a77aa59 with:

1. The nonzero submission result.
2. Zero.

Its 32-bit result becomes the low return field.

If this result is nonzero, no further explicit dependency call occurs. The high return field remains the temporary slot’s address.

If the result is zero, the routine:

1. Calls F-unknown-02 again with no arguments.
2. Reads the selected 32-bit entry through the base saved by the first global-base read in the association phase.
3. Calls F-8becc2a0337c with the newly obtained context and that newly read entry.

This final entry access occurs after the second context call. It does not reload the global entry-table base, does not necessarily use the base used for association, and does not reuse the earlier entry value.

Consequently:

- Later changes to the global base do not redirect this final access.
- Changes to the entry at the saved base may affect the value passed to the final action.
- If the two earlier global-base reads differed, association and final action can obtain their entries from different tables.

The final action’s result, if any, is ignored. If it returns normally, the low return field remains zero.

## Side effects

### Direct memory accesses

The supplied body establishes this ordered access structure:

- A 32-bit temporary-slot write containing the fourth argument, before the preliminary call.
- For selector 0 or 1, a global flag-base read and one selected byte read, after the preliminary call.
- On the submission path, a zero write to the temporary slot before submission.
- After nonzero submission and the first context call:
  - One global entry-base read whose value is saved.
  - A second, independent read of that global base.
  - One selected 32-bit entry read through the second base value.
- If the follow-on result is zero, one further selected 32-bit entry read through the saved first base, after the second context call.

There are exactly two explicit global entry-base reads on the successful-submission path, not one. The optional final action adds an entry-content read, not another global-base read.

No explicit write to either table or either global base appears. Dependencies must not be assumed to preserve them.

The routine does not explicitly read the temporary slot after submission. Its contents do not select the low return field.

### Ownership and lifetime

No allocation or release semantics are established by the body. Preparation results and table entries are forwarded to dependencies, but their ownership is unknown.

In particular, the evidence does not establish that:

- Preparation creates an owned resource.
- Submission consumes or releases its first argument.
- Association acquires an entry.
- The final action releases, restores, or cleans up anything.
- Either context call returns the same context as an earlier call.

The temporary slot belongs to this invocation. Returning its address bits does not extend its lifetime. Any dependency retaining that address would need an external lifetime contract.

### Failure and interference

No explicit retry, rollback, or exception handling is visible. If a dependency fails to return normally, the subsequent accesses, calls, and packed return are not established.

Dependencies may change memory or global state. Concurrent and reentrant behavior is not established. The distinction between the two entry-base reads must be retained without asserting either that interference occurs or that the reads are stable.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-c5cf4e75bd67 | Always called first with selector and 11; returns the 64-bit preliminary value. |
| F-unknown-01 | Called with zero before preparation; supplies the 32-bit value to which 4 is added. |
| F-b37968c9f644 | Receives that adjusted value and the preliminary low field; its result is forwarded to submission. |
| F-cd7e1905512f | Receives six arguments including 39, the zero-initialized temporary address, and 1; zero selects result 28. |
| F-unknown-02 | Called without arguments before association and again before the optional final action. |
| F-23aadb16b673 | Receives the first context and an entry obtained through the second global entry-base read. |
| F-256e8a77aa59 | Receives the nonzero submission result and zero; its result supplies the final low field. |
| F-8becc2a0337c | Receives a fresh context and an entry reread through the saved first entry-base value; called only after a zero follow-on result. |

F-unknown-01 and F-unknown-02 are document-local opaque identifiers for dependencies without supplied public IDs. Their names assign no application semantics.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Unless otherwise stated, required memory is valid and all dependencies return normally.

1. **Selector outside the distinguished pair**

   With selector 7 and preliminary upper field 9, the return fields are low = 9 and high = 7. The preliminary lower field is irrelevant. No direct table access follows the preliminary call.

2. **Nonzero flag**

   With selector 1 and a nonzero selected flag byte, the return fields are low = 0 and high = 1, regardless of the preliminary result. The preliminary dependency still runs before the flag read.

3. **Zero preliminary lower field**

   With selector 0, flag zero, and preliminary fields lower = 0 and upper = 6, the return fields are low = 6 and high = 0. Preparation and submission do not occur.

4. **Submission failure-shaped branch**

   With selector 1, flag zero, and a nonzero preliminary lower field, submission receives the address of a slot initialized to zero. If submission returns zero, the low result is 28 and the high result is the slot’s address. The fourth argument’s original value does not determine either field.

5. **Nonzero follow-on result**

   After nonzero submission, association precedes the follow-on call. If the follow-on result is `0xFFFFFFFF`, that exact pattern occupies the low field, the temporary address occupies the high field, and the final action is omitted.

6. **Stable entry-base reads**

   Suppose both global entry-base reads yield address A. Association receives the selected entry at A as read before association. If later dependencies change that entry and the follow-on result is zero, the final action receives the entry’s later value at A.

7. **Different entry-base reads**

   Conditional on an execution environment permitting the global value to change between the two explicit reads, suppose the saved first read yields A and the second read yields B. Association receives the selected entry from B. If the follow-on result is zero, the final action receives a newly read entry from A, after the second context call. There is no explicit dependency call between the two base reads; this example does not assert concurrency support.

8. **Later global-base change**

   Suppose the first read saves A and the second yields B. A later dependency changes the global base to C. With a zero follow-on result, the final entry access still uses A, not B or C.

9. **Submission writes the slot**

   If submission writes 123 into the temporary slot and returns zero, the low result remains 28. The high field contains the slot’s address, not 123.

## Uncertainties

### Inferred intent

The two distinguished selectors, flag gate, submission, and paired entry-related calls suggest a conditional operation over a small predefined resource set. This is inferred intent only. It does not establish a resource type or justify describing the final action as cleanup, release, restoration, or rollback.

### documented_only

The following remain unresolved:

- Application meanings and valid domains of arguments, entries, contexts, and results.
- Meaning of result 28 and other returned bit patterns.
- Table allocation sizes, initialization, mutability, and synchronization.
- Stability of either global table base.
- Whether the two consecutive entry-base reads can observe different values in supported execution.
- Dependency effects on globals, entries, and temporary storage.
- Resource ownership and transfer obligations.
- Whether submission retains the temporary address.
- Concurrent or reentrant execution guarantees.
- Behavior for invalid addresses, overflowing arithmetic, or abnormal dependency completion.
- Whether the packed return is an intended public format or a lower-level calling-convention artifact.

No unknown dependency is assumed to preserve globals or acquire, consume, or release memory.

## Evidence

This contract derives from the supplied static body and opaque dependency mapping. The prior draft and reviewer feedback are not independent evidence.

Established observations include the 32-bit inputs, 64-bit field concatenation, branch conditions, constants 11, 39, 1, and 28, byte-sized flag access, four-byte entry spacing, temporary-slot writes, and dependency-call order.

The body specifically establishes two consecutive global entry-base reads after the first context call: the first is saved for the optional final access; the second addresses the association entry. No explicit dependency call separates those reads. The final entry read uses the saved first base after another context call.

Application semantics, ownership, synchronization guarantees, and abnormal-completion behavior are not established. No runtime tests have been performed.
