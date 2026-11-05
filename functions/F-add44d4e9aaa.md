<!-- Trusted provenance: F-add44d4e9aaa; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-add44d4e9aaa

## Purpose

F-add44d4e9aaa operates on a mutable record according to two ordered state conditions. It either invokes a dependency and conditionally waits, performs list-link replacement, or bypasses both operations. Every normally returning path writes an entry-time terminal-state value into the record, except an explicit early return on the list-processing path when a gate is zero.

The first state condition takes precedence over the second. A zero gate has no effect unless the second condition is selected after the first condition fails.

The supplied static representation establishes these operations, not their application-level meaning. Descriptions such as cancellation, completion, scheduling, or destruction remain inferred intent.

## Interface

### Input and output

The function accepts one pointer to a mutable record. It returns no value and provides no explicit error result.

There is no visible validation of the pointer, record extent, list membership, or list integrity. Storage must remain valid for all accesses reached during the invocation, including accesses after dependency calls.

### Record layout and widths

The supplied representation supports the following 32-bit-word layout. These offsets describe the recovered representation; the original machine access details are not independently established.

| Byte offset | Width | Role |
|---|---:|---|
| 0 | 32 bits | Successor address, with zero representing the end of a list. |
| 12 | 32 bits | First argument to F-77781c8e0ff9. |
| 24 | 32 bits | State used by dispatch and waiting, then overwritten on normal completion. |
| 28 | 32 bits | Gate checked only on the selected list-processing path and cleared after that processing. |

A traversed list node must expose a readable successor word at offset 0. A nonzero replacement head used by the head-change comparison must also expose a readable 32-bit value at offset 8.

Addresses carried in successor words are interpreted as pointers by the supplied representation. This does not establish portability to a different pointer width.

### External state

The following labels identify roles, not source symbols or additional parameters:

| Label | Established use |
|---|---|
| Terminal-state source | Supplies the value captured at entry for waiting and the final state write. |
| Base state | Participates in each dispatch expression. |
| State adjustment | Participates only in the second dispatch expression. |
| List-head pointer | Identifies the storage containing the current first-node address. |
| Comparison-reference pointer | Identifies the storage supplying the head-change comparison reference. |
| Invocation argument | Supplies the second argument to F-77781c8e0ff9. |
| Wait argument | Supplies the argument to each F-162a41bac6a7 call. |
| Head-change argument | Supplies the argument to F-5bcae2eff9a7. |

The gate is not validated against list membership. A nonzero gate permits list processing; it does not establish that the input record is present in the list.

## Behavior

### Entry capture and ordered dispatch

The first established operation captures the terminal-state source. That captured value remains the wait target and the value used by the final state write. A later change to its external source does not replace the capture.

Dispatch then proceeds in this order:

1. The function evaluates the record state minus the base state. A zero result selects invocation and waiting.
2. Only if the first result is nonzero does it evaluate the record state minus the base state, followed by addition of the state adjustment. A zero result selects list processing.
3. If neither expression is zero, neither specialized path is taken; execution proceeds to the final state write.

The two expressions contain separate accesses to the record state and base state. The representation does not establish a shared snapshot across both decisions.

For stable values and arithmetic that does not overflow, the first condition corresponds to equality with the base state; the second corresponds to equality with the base state minus the adjustment. These mathematical descriptions do not replace the represented arithmetic where overflow matters.

If both expressions would be zero for stable values, the first path wins. In particular, a zero adjustment does not make the gate relevant when the first condition succeeds.

### Invocation and waiting

On the first dispatch path, F-77781c8e0ff9 is called once with these arguments, in these positions:

1. The record’s 32-bit value at offset 12.
2. The external invocation argument.
3. The original input-record pointer.

The dependency result is classified only as zero or nonzero.

If the result is nonzero, no wait-state check or wait call is required by this path. Execution proceeds to the final state write, regardless of the record state left by the dependency.

If the result is zero, the function checks the record state against the captured terminal value:

- Equality ends waiting immediately.
- Inequality causes one call to F-162a41bac6a7 with the external wait argument.
- After that call returns, the state is checked again.

The wait argument is obtained for each call rather than explicitly captured at function entry. Changes to its external source between calls can therefore affect later arguments.

There is no visible timeout, iteration limit, cancellation condition, or alternative returning exit from the loop. If every check observes a state different from the captured target, execution does not reach the final state write. A wait call alone does not establish progress or synchronization.

After equality is observed, the final state write still occurs. The function does not merely rely on the state value it observed.

### List-processing entry and early return

List processing is selected only when the first dispatch expression is nonzero and the second is zero.

The function then reads the gate at offset 28:

- A zero gate causes an immediate return.
- A nonzero gate permits traversal and link replacement.

The zero-gate return omits list-head access, traversal, link replacement, dependency calls, gate clearing, and the final state write. Entry capture and dispatch have already occurred.

### Traversal and successor acquisition

With a nonzero gate, traversal begins at the node address stored in the external head location. Nodes are compared with the input record by pointer identity.

Traversal stops on either:

- Encountering the input-record address; or
- Reaching a zero candidate address.

Each nonzero, nonmatching node contributes its successor as the next candidate. The most recently visited nonmatching node is retained as the predecessor.

After traversal stops, the function reads the successor from offset 0 of the input record. This read occurs whether or not traversal found the record. The link replacement uses this acquired value.

A cycle that never reaches the input record or a zero link can prevent traversal from completing.

### Replacement without a predecessor

If no predecessor was retained, the external head location is overwritten with the acquired input-record successor.

This includes both:

- The input record being the original head.
- The original list being empty.

The empty-list case is not protected by a membership check. An absent record with a nonzero gate can therefore supply a new head through its successor field.

After the head write:

- A zero replacement head suppresses the comparison and head-change call.
- A nonzero replacement head causes a comparison using the external reference and the replacement node’s value at offset 8.

The tested expression subtracts the replacement-node value from the reference and checks whether the signed result is negative. If so, F-5bcae2eff9a7 is called once with the external head-change argument.

The head write precedes the comparison and any resulting call. The relative order of the two comparison operand reads is not established.

### Replacement with a predecessor

If a predecessor was retained, its successor word is overwritten with the acquired input-record successor.

When the input record was found, this bypasses it in the list. When traversal instead reached zero without finding it, the last visited node receives the same write. The function does not distinguish these outcomes or report absence.

This branch makes no direct F-5bcae2eff9a7 call, regardless of the replacement link’s value.

### Gate clearing and final state write

After link replacement and any head-change dependency call have completed, the function writes zero to the input record’s gate.

It then writes the captured terminal value to the record state.

The function does not directly clear the input record’s successor. With nonaliasing storage and no relevant dependency mutation, that field retains its previous value.

The final state write also occurs after:

- Bypassing both specialized paths.
- A nonzero invocation result.
- A zero invocation result followed by immediate or eventual wait-target equality.

Only the explicit zero-gate return omits that write among normally returning paths. Nonreturning dependencies, invalid accesses, or nonterminating traversal or waiting can also prevent later operations from being reached.

## Side effects

### Direct writes and their order

The established direct writes are limited to:

- The external head location or a retained predecessor’s successor.
- The input record’s gate.
- The input record’s state.

On a completing list-processing path, their order is:

1. Head or predecessor-link replacement.
2. Conditional head-change call, applicable only after head replacement.
3. Gate clear.
4. Final state write.

On the invocation path, the invocation call and any wait calls precede the final state write.

These are direct effects only. Dependency bodies are unavailable and may have additional effects.

### Ownership and lifetime

No allocation, release, reference-count change, or ownership-transfer operation is directly visible. Link replacement must not be interpreted as freeing the record.

Dependencies are not assumed to preserve globals, list contents, record fields, or ownership. The function continues accessing the input record after dependency calls. If a dependency invalidates that storage, safe continuation is not established.

Aliasing can affect later reads and the apparent set of changed fields. For example, a head-location write may affect another logical field if their storage overlaps. Nonaliasing is a condition for the ordinary field-by-field interpretation, not a checked guarantee.

### Concurrency and errors

No lock, atomic operation, barrier, or explicit fault handling is shown.

Repeated state checks suggest an expectation of change, but the representation does not identify the writer or establish a valid synchronization model. Concurrent access safety cannot be concluded from the loop alone.

Invalid pointers and malformed nodes can cause invalid accesses. No defined diagnostic or recovery behavior is supplied.

## Dependencies

| Opaque function ID | Established interaction | Unresolved behavior |
|---|---|---|
| F-77781c8e0ff9 | One call on the first dispatch path, with the offset-12 value, external invocation argument, and record pointer. Result tested for zero. | Result meaning, mutation, ownership, progress, and whether it returns. |
| F-162a41bac6a7 | One call for each unequal wait-state check, with the external wait argument. Result unused. | Whether it waits, yields, updates state, or provides synchronization. |
| F-5bcae2eff9a7 | Conditional call after a nonzero head replacement and negative signed subtraction result. Receives the external head-change argument. Result unused. | Notification meaning, mutation, ownership, and whether it returns. |

Argument positions are established. The relative evaluation order of individual argument-value reads is not independently established.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Unless otherwise stated, assume valid nonaliasing storage, stable external configuration, returning dependencies, and arithmetic without overflow. Any dependency mutation relevant to an example is stated explicitly.

1. **Neither dispatch condition matches.**  
   If the first expression is nonzero and the second is nonzero, the record state receives the captured terminal value. There is no dependency call, link replacement, or direct gate clear.

2. **Invocation returns nonzero.**  
   If the first expression is zero and F-77781c8e0ff9 returns nonzero, there is no wait call. The final state write occurs. The gate does not affect this outcome.

3. **Invocation reaches the target before waiting.**  
   If the first expression is zero, the invocation returns zero, and the first subsequent state check equals the captured target, no wait call occurs. The final state write still occurs.

4. **Exactly one wait call.**  
   If the invocation path returns zero, the first check differs from the target, and the check after the first returning wait call equals it, exactly one wait call occurs before the final state write.

5. **Waiting does not complete.**  
   If the invocation returns zero and every subsequent check differs from the captured target, returning wait calls continue indefinitely. No final state write is reached.

**Shared dispatch precondition for examples 6–11:** the first dispatch expression is nonzero, and the second dispatch expression is zero. Thus list processing is selected rather than merely satisfying the second expression in isolation. Example 6 requires a zero gate; examples 7–11 require a nonzero gate.

6. **Selected list path with a zero gate.**  
   Under the shared dispatch precondition and a zero gate, the function returns before list access and does not write the state. If both dispatch expressions were instead zero, this example would not apply: the invocation path would take precedence.

7. **Head replacement with a notification.**  
   Under the shared dispatch precondition and a nonzero gate, suppose the input record is the head and its successor is nonzero. That successor becomes the head. If the comparison reference is 10 and the replacement node’s offset-8 value is 20, the subtraction is negative and F-5bcae2eff9a7 is called. After it returns, the gate is cleared and the terminal value is written.

8. **Head replacement without a notification.**  
   Under the shared dispatch precondition and a nonzero gate, suppose the input record is the head. A zero successor becomes a zero head and suppresses the comparison. A nonzero successor also suppresses the call when the comparison difference is zero or positive. Both cases then clear the gate and write the terminal value.

9. **Interior record.**  
   Under the shared dispatch precondition and a nonzero gate, suppose traversal encounters a predecessor, then the input record, then a successor. The predecessor’s successor becomes the input record’s successor. No direct head-change call occurs. The gate is cleared, the terminal state is written, and the input successor is not directly cleared.

10. **Record absent from a nonempty list.**  
    Under the shared dispatch precondition and a nonzero gate, suppose a finite nonempty list reaches zero without encountering the input record. Its last node’s successor is overwritten with the input record’s successor. No “not found” result is produced; gate clearing and the final state write follow.

11. **Record absent from an empty list.**  
    Under the shared dispatch precondition and a nonzero gate, suppose the initial head is zero. The input record’s successor becomes the head. A nonzero replacement is subject to the head-change comparison. Gate clearing and the final state write follow.

12. **Terminal-state source changes.**  
    If the entry capture is 7 and a returning dependency changes its external source to 9, the invocation still uses 7 as its wait target and final written value, whenever those operations are reached.

## Uncertainties

- **documented_only — Arithmetic semantics.** Dispatch uses subtraction and addition; the head-change condition tests a signed subtraction result. Exact overflow behavior is not resolved. A mathematical less-than comparison is equivalent to the latter only where the subtraction is representable.
- **documented_only — Address arithmetic and ABI.** The layout reflects a 32-bit-word representation, including integer-carried addresses and an offset-8 access. Machine instructions, alignment constraints, address-overflow behavior, and original pointer representation are unavailable.
- **documented_only — Access mechanics.** Volatile qualification, compiler reload guarantees, and exact physical access widths are not independently verified.
- **documented_only — External state.** Initialization, mutability, aliasing, and identity relationships among external locations are unknown.
- **documented_only — Dependency effects.** No global-preservation, ownership, progress, or successful-return guarantee is established.
- **documented_only — Synchronization.** No mechanism establishing safe visibility of concurrent state changes is supplied.
- **Inferred intent.** The gate may indicate list participation, and the head-change call may react to an ordering condition. Neither interpretation is needed to describe the established behavior.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping. The prior draft and feedback are not independent behavioral evidence.

The representation establishes entry-time terminal capture, ordered dispatch, zero/nonzero invocation-result handling, repeated state checks, pointer-identity traversal, successor acquisition even after unsuccessful traversal, predecessor-dependent link replacement, the conditional head-change call, gate clearing, and the final state write.

The acceptance conditions explicitly account for first-branch precedence and the gate requirement. No dependency bodies, machine instructions, runtime traces, or execution results were supplied. No runtime tests have been performed.
