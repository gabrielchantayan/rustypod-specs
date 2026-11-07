<!-- Trusted provenance: F-0398acce38ad; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0398acce38ad

## Purpose

F-0398acce38ad traverses an acquired descriptor, processes each indexed record, optionally traverses referenced descendants, and performs a final action selected by the caller.

The established behavior is an ordered sequence of dependency calls and descriptor accesses. Inferred intent is recursive processing of a structured object whose records can reference other objects. The meanings of those objects, the finalization operation, and the returned status codes are not established beyond the control flow described here.

This contract distinguishes:

- **Established static evidence:** behavior directly supported by the supplied function.
- **Inferred intent:** descriptive interpretation, not an additional requirement.
- **documented_only:** behavior that depends on unavailable dependency contracts or runtime conditions.

No runtime tests have been performed.

## Interface

The function takes four arguments and returns a 32-bit signed status value.

| Argument | Representation | Established use |
|---|---|---|
| State reference | Pointer to a 32-bit word | Its first word is read for an initial query. The original pointer is also passed to acquisition and recursive calls. |
| Requested identifier | Unsigned 32-bit value | Compared against the initial query result, then passed to acquisition. Recursive identifiers come from a dependency. |
| Acquisition context | Opaque 32-bit value | Passed unchanged to acquisition. Recursive calls instead obtain their context from the acquired descriptor. |
| Completion mode | Signed 32-bit value | Zero selects a status-query-and-update path. Every nonzero value selects an alternative completion dependency. |

The return value is not a Boolean. Zero permits processing to continue; nonzero values stop the current traversal or represent the final completion result. The only locally introduced nonzero status is decimal **11**, returned by the initial range check.

### Descriptor layout

Acquisition receives an output location initially containing a zero 32-bit value. On successful acquisition, the function treats the resulting value as a descriptor address with these accessed fields:

| Byte offset | Access width | Meaning established by use |
|---|---:|---|
| +0x04 | 8 bits | Zero enables recursive descent; any nonzero value suppresses it at that decision point. |
| +0x0E | Unsigned 16 bits | Offset from a data base to a two-byte-per-entry offset table. |
| +0x14 | Unsigned 16 bits | Entry count, read at each loop-condition evaluation. |
| +0x44 | 32 bits | Data-base address. |
| +0x48 | 32 bits | Opaque value supplied to the zero-mode status query. |
| +0x50 | 32 bits | Acquisition context for recursive calls. |

These are accessed fields, not a complete descriptor definition. No structure packing, allocation size, or meaning for intervening bytes is established.

Each table entry contains a **big-endian unsigned 16-bit displacement**. Its first byte is the high-order byte. Adding that displacement to the data-base address produces a record address.

The record format is otherwise unknown. A dependency interprets a record address to obtain a 32-bit recursive identifier. That same dependency also receives the data-base address plus eight for a distinct, post-loop recursive traversal.

### Preconditions

For a defined, successful execution:

- The state reference must support its initial 32-bit read.
- Successful acquisition must provide a descriptor supporting every field access reached by the execution.
- The table and resolved records must support the reads made by this function and its dependencies.
- Addresses retained across dependency calls must remain usable when subsequently passed onward.
- Recursive processing must eventually terminate if normal return is required.

The function does not locally validate null pointers, descriptor lengths, table bounds, record lengths, or recursion depth.

## Behavior

### 1. Initial query and rejection

The function first initializes its descriptor output storage to zero.

It reads the first 32-bit word through the state reference and passes that word to **F-f3192f39e4f6**. The returned value is used as an unsigned 32-bit quantity.

If that value is less than the requested identifier, the function returns **11** immediately.

Consequences of this strict comparison:

- Equality passes the initial check.
- A requested identifier of zero cannot be rejected by this unsigned comparison.
- This function makes no acquisition or cleanup call on this early-return path.
- Any effects performed by the query dependency have already occurred.

The query result’s semantic meaning—such as a size, maximum identifier, or limit—is **documented_only**. In particular, equality being accepted must not be replaced with a conventional “index must be less than length” rule.

### 2. Descriptor acquisition

If the initial check passes, the function calls **F-6293178c6e41** with:

1. The original state reference.
2. The requested identifier.
3. The address of the initially zero descriptor output storage.
4. The caller’s acquisition context.

A zero status enters traversal. A nonzero status skips traversal and completion processing.

Both outcomes subsequently reach cleanup. Thus, acquisition failure still causes cleanup of the descriptor output value left by acquisition. That value may remain zero or may have been changed; neither possibility is excluded by the visible function.

### 3. Indexed-record traversal

Traversal begins at entry index zero.

Before each iteration, the function reads the descriptor’s unsigned 16-bit entry count. The loop continues while the index is smaller than that count. The count is not established as a snapshot taken at acquisition time.

For each admitted entry, processing is ordered as follows:

1. Read the unsigned 16-bit table offset from the descriptor.
2. Compute that offset plus twice the current index.
3. Read the data-base address from the descriptor.
4. Read the table entry’s two bytes relative to that base.
5. Interpret those bytes as a big-endian unsigned 16-bit displacement.
6. Add the displacement to the loaded data-base address, producing the current record address.
7. Read the descriptor’s recursion-control byte.
8. If that byte is zero, perform the recursive processing below.
9. If no recursive error stopped processing, call **F-d153bcc89350** with the descriptor address and the already resolved record address.
10. If that call succeeds, increment the index and reevaluate the loop condition.

The arithmetic used for the table position has a bounded mathematical range. An index admitted by a 16-bit count is at most 65,534, so twice the index plus a 16-bit table offset is at most 196,603. This calculation does not itself require more than 32 signed bits. Whether the resulting addresses refer to valid storage is a separate precondition.

The relative order of the two individual table-byte loads is not independently established. Both contribute to the displacement before recursion or record processing occurs.

#### Recursive processing for an indexed record

When the recursion-control byte is zero:

- Call **F-d1e995cfb2d3** with the resolved record address.
- Use its 32-bit result as the requested identifier for a recursive call to **F-0398acce38ad**.
- Pass the original state reference.
- Obtain the recursive acquisition context from descriptor offset +0x50.
- Use completion mode **1**, regardless of the caller’s completion mode.

If this recursive call returns nonzero, the parent immediately proceeds to cleanup. The parent does not call F-d153bcc89350 for that record and does not process later entries.

If recursion succeeds, F-d153bcc89350 receives the record address resolved **before** the recursive call. The parent does not recompute that address afterward.

Any nonzero result from F-d153bcc89350 likewise stops traversal and becomes the status returned after cleanup.

### 4. Post-loop recursive traversal

After the indexed loop finishes normally, the function reads the recursion-control byte again.

If it is zero:

- Read the current data-base address.
- Pass that address plus eight to **F-d1e995cfb2d3**.
- Recursively invoke **F-0398acce38ad** using the returned identifier, the original state reference, the descriptor’s acquisition context at +0x50, and completion mode 1.

A nonzero result stops processing and leads to cleanup.

This traversal occurs even when the indexed entry count is zero, provided the recursion-control byte is zero. It is separate from indexed-record processing: the parent makes no corresponding F-d153bcc89350 call for the data-base-plus-eight address.

The recursion-control byte is reread at each decision point. If dependencies change it, later decisions follow its later value. The function does not establish one fixed recursive/nonrecursive classification for the entire invocation.

### 5. Completion

Completion occurs only after all reached indexed processing and the optional post-loop recursion have succeeded.

#### Caller completion mode is zero

The function calls **F-81c906bf6e3a** with the 32-bit value at descriptor offset +0x48.

- If the result is nonzero, it becomes the return status and no update call follows.
- If the result is zero, the function reads the descriptor’s current data-base address and then its first byte. It calls **F-f6da3c4a0ea5** with the descriptor and the unsigned byte value bitwise-ORed with **0x08**.

The second argument to that update call is in the range 8–255. It preserves the byte’s existing bits and ensures bit 3 is set in the value passed to the dependency.

The visible function does not directly store that value into the data buffer. Any actual mutation is the dependency’s responsibility. Its return value, if any, is not used as the function’s status; the retained status is zero.

#### Caller completion mode is nonzero

The function calls **F-ff6acbee5e19** without explicit arguments and uses its result as the return status.

All nonzero completion-mode values select this same branch, including negative values. Recursive calls always select it by passing 1.

### 6. Cleanup and return

Except for the initial rejection, every normal return path calls **F-6cc778185718** with the descriptor output value.

Cleanup follows:

- Acquisition failure.
- A recursive failure.
- A record-processing failure.
- Either completion branch.

The cleanup dependency’s return value, if any, is not used. The function returns the status selected before cleanup.

## Side effects

Directly visible writes are limited to local bookkeeping and the output storage exposed to acquisition. Other externally observable effects occur through dependencies.

Important ordering and state properties:

- Descendant processing precedes processing of its corresponding parent record.
- Indexed records are processed in increasing index order, subject to the freshly read count.
- The data-base-plus-eight traversal follows all successful indexed processing.
- Completion follows successful traversal.
- Cleanup follows acquisition and any reached processing, including failure paths.
- There is no visible rollback of completed child traversals or earlier record processing when a later operation fails.

The function contains no visible global-variable access. This does **not** imply that dependencies preserve global state or that the no-argument completion dependency is independent of preceding calls.

**Ownership is documented_only.** Cleanup is visibly called, but the supplied evidence does not establish whether it frees memory, decrements a reference, unlocks a resource, or performs another action. Likewise, successful acquisition does not by itself establish exclusive ownership.

Dependencies may affect shared state. The contract therefore preserves the distinction between fields reread later and values already retained, especially the record address retained across recursion.

## Dependencies

| Opaque ID | Established role and arguments |
|---|---|
| **F-f3192f39e4f6** | Receives the first state word; returns the unsigned value used in the initial comparison. |
| **F-6293178c6e41** | Receives the state reference, requested identifier, descriptor output location, and acquisition context; returns a status. |
| **F-d1e995cfb2d3** | Receives a record-like address; returns a 32-bit value used as a recursive identifier. |
| **F-0398acce38ad** | Recursive invocation with the original state reference, derived identifier, descriptor-provided context, and mode 1. |
| **F-d153bcc89350** | Receives the descriptor and a resolved indexed-record address; returns a status. |
| **F-81c906bf6e3a** | Receives the descriptor field at +0x48; returns the zero-mode completion status. |
| **F-f6da3c4a0ea5** | Receives the descriptor and a byte-derived value with bit 3 set; its result is ignored. |
| **F-ff6acbee5e19** | Receives no explicit arguments; supplies the nonzero-mode completion status. |
| **F-6cc778185718** | Receives the acquisition output value during cleanup; its result is ignored. |

No dependency’s internal memory behavior, global-state behavior, or failure semantics are established beyond these uses.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests. Unless stated otherwise, each example assumes valid storage and dependencies that leave unrelated fixture fields unchanged.

1. **Initial rejection:** If the initial query returns 4 and the requested identifier is 5, the result is 11. Acquisition and cleanup are not called by this invocation.

2. **Equality passes:** If both values are 4, acquisition is reached. The initial check does not itself determine the final result.

3. **Acquisition failure:** If acquisition returns 7 and leaves its output zero, cleanup is called with zero and the function returns 7. No descriptor fields are read for traversal.

4. **Big-endian displacement:** If a table entry’s bytes are 0x01 and 0x02, its record address is the loaded data base plus 258, not plus 513.

5. **Nonrecursive indexed processing:** With two entries and a nonzero recursion-control byte throughout, F-d153bcc89350 is called for entry zero and then entry one, assuming both calls succeed. No recursive traversal is initiated by this invocation.

6. **Recursive failure precedes record processing:** If the first entry enables recursion and its recursive call returns 6, that entry’s F-d153bcc89350 call is skipped. Later entries and completion are skipped; cleanup runs; the result is 6.

7. **Empty recursive descriptor:** With a zero entry count and a zero recursion-control byte, the function still performs the data-base-plus-eight recursive traversal before completion.

8. **Zero-mode completion:** If the completion query returns zero and the current first data byte is 0x21, the update dependency receives 0x29. The retained result is zero.

9. **Completion-query failure:** If that query returns 9, the update dependency is not called. Cleanup runs and the result is 9.

10. **Any nonzero mode:** With mode −1 and successful traversal, the no-argument completion dependency is selected. Its status is returned after cleanup.

11. **Changed count:** If a successful first record-processing call changes the count from two to one, the next loop-condition read ends indexed traversal. A cached original count would not satisfy this behavior.

## Uncertainties

The following remain **documented_only**:

- The semantic domain and validity rules of identifiers and acquisition contexts.
- The meaning of status 11 and dependency-specific nonzero statuses.
- Complete descriptor and record formats.
- Native byte order and alignment requirements of multibyte descriptor fields. Only the table-entry byte order is explicitly established.
- Validity and overflow rules for address additions near address-space boundaries.
- Dependency ownership, allocation, release, synchronization, and global-state contracts.
- Behavior after invalid memory access, dependency nonreturn, or stack exhaustion.
- Whether object references can form cycles. No visited-set or recursion-depth guard is visible.
- Whether cleanup accepts zero or partially acquired descriptors safely.
- Whether dependencies may invalidate addresses retained across recursion.

These unknowns must not be replaced with assumed validation, rollback, cycle detection, or memory-management guarantees.

## Evidence

The supplied static function supports the stated field widths, offsets, unsigned initial comparison, explicit big-endian table decoding, recursive call arguments, failure propagation, completion branches, and cleanup placement.

Traversal and finalization terminology describes inferred intent only. Dependency semantics beyond their visible call interfaces are unavailable.

No runtime execution, dependency inspection, or behavioral tests have been performed.
