<!-- Trusted provenance: F-1acf7eeabd3d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1acf7eeabd3d

## Purpose

F-1acf7eeabd3d prepares an embedded record using an unsigned 32-bit value obtained through its containing object. Except for a sentinel path, it notifies a dependency, selects a configuration quantity, prepares the record, and conditionally inserts it into a global singly linked list. The scheduling branch ends with another dependency call and a conditional indirect invocation.

“Configuration,” “membership,” “duration,” and “deadline” are descriptive roles inferred from the visible operations. Timer semantics, ordering policy, synchronization guarantees, and dependency ownership rules are not established.

This contract describes direct accesses and call boundaries. It does not assume that any dependency preserves globals, object fields, links, or memory lifetime.

## Interface

### Arguments and result

| Argument | Established use |
|---|---|
| Object address | Locates an associated-object pointer, an embedded record, and other fields. |
| Second four-byte argument | No direct use is visible. |
| Third four-byte argument | Passed to the comparison dependency during list traversal. |
| Fourth four-byte argument | Passed to the comparison dependency during list traversal. |

No result value is specified. Normal exits return without a defined output value.

The supplied representation uses four-byte addresses and scalar accesses. The contract is therefore specific to the represented 32-bit address layout, not an arbitrary host pointer width. Byte order is not independently established.

### Object layout

Offsets are byte offsets from the supplied object address.

| Offset | Access width | Role |
|---|---:|---|
| `0x28` | 4 bytes | Associated-object address, loaded at several distinct boundaries. |
| `0x60` | 4 bytes | Embedded-record start; also its next-link during insertion. |
| `0x64` | 4 bytes | Duration-like quantity read after scheduling preparation. |
| `0x68` | 4 bytes | Computed deadline-like quantity; address passed as the new comparison key. |
| `0x78` | 4 bytes | Assignment from global-backed insertion data. |
| `0x7C` | 4 bytes | Membership-like field: zero permits scheduling; insertion writes one. |
| `0x80` | 4 bytes | Assignment from global-backed preparation data. |
| `0x8C` | 1 byte | Flag cleared initially and directly set to one on the above-threshold path. |

The associated object exposes a four-byte field at offset `0x48C`. Its initial value is interpreted as unsigned 32-bit.

Each traversed list node provides a four-byte next-link at offset `0x00` and a comparison-key address at offset `0x08`. For the embedded record, that key address is object offset `0x68`. The comparator’s actual key layout and access extent are unknown.

### Global-backed inputs

The following labels identify distinct global references in this contract. Their numeric contents and application meanings are unavailable. They are not assumed immutable.

| Label | Visible role |
|---|---|
| G1 | Four-byte multiplier read before notification. |
| G2 | Four-byte replacement configuration read on the above-threshold path. |
| G3 | Four-byte preparation value assigned to object offset `0x80`. |
| G4 | Shared address-valued slot used both for scheduling entry and final dispatch gating. |
| G5 | Address-valued slot identifying list-head storage. |
| G6 | Four-byte insertion value assigned to object offset `0x78`. |
| G7 | Global-backed argument for the post-scheduling dependency. |
| G8 | Global-backed indirect callable target. |

### Access and execution requirements

The body provides no checks for null object pointers, accessible memory, alignment, object size, valid list nodes, or executable targets. Every reached access must have suitable storage, including accesses made by dependencies.

Termination is not guaranteed. A valid cyclic list whose comparisons continually return zero can be traversed indefinitely. Reaching null or a nonzero comparison result is an additional condition for traversal termination, not a general requirement for defined execution.

## Behavior

### Initial snapshot and sentinel exit

The function first loads the associated-object address from object offset `0x28`, then reads the unsigned 32-bit value at that address plus `0x48C`. The resulting value is retained as a snapshot.

It next calls F-936a8728139d with the embedded-record address, object offset `0x60`. After that call returns, it writes zero to the byte at object offset `0x8C`.

If the saved value equals `0x0000FFFF`, the function returns immediately. Thus the sentinel path includes both the helper call and the subsequent flag clear. Changes made by that helper to the associated field do not change the sentinel decision, which uses the earlier snapshot.

### Configuration calculation and notification

For every non-sentinel snapshot:

1. G1 is read.
2. A 32-bit product of G1 and the saved unsigned value is formed.
3. The associated-object address is freshly loaded from object offset `0x28`.
4. F-f36c23c5ba7b is called with that address and the saved value.

The initial multiplication is unsigned 32-bit arithmetic. Its retained bit pattern is the product modulo \(2^{32}\), not an unrestricted mathematical product. It is formed even when the subsequent above-threshold branch discards it.

The G1 read and product precede notification. A change to G1 during F-f36c23c5ba7b does not change the already formed candidate.

The freshly loaded associated-object address need not match the address used for the initial snapshot: F-936a8728139d may have changed object state.

### Threshold selection

The saved value is compared with 30 as unsigned:

- **At most 30:** the previously formed product remains selected. This branch performs no further direct flag write and no direct write to the associated field.
- **Above 30:** the function directly writes one to object byte `0x8C`, then reads G2 as the replacement configuration. It subsequently reloads the associated-object address from object offset `0x28` and writes the saved value minus 30 to that associated object’s field at offset `0x48C`.

The sentinel has already returned and therefore does not enter either continuation.

On the at-most-30 path, the flag is **not guaranteed to remain zero**: F-f36c23c5ba7b runs after the initial clear and may modify it. Later dependencies may modify it as well. The established fact is the absence of another direct flag write on that branch.

On the above-threshold path, the G2 read occurs after notification and after the direct flag write. Its value is not an entry-time constant. The associated-field write uses the saved input, not the destination’s current contents, and the destination address can differ from the address passed to notification.

The selected four-byte configuration is passed with the embedded-record address to F-6904f7699031.

### Preparation and membership decision

After F-6904f7699031 returns, F-936a8728139d is invoked again. The supplied representation does not establish the arguments of this second invocation; it must not be specified as a proven repeat of the first call.

After the second invocation returns, G3 is read and its four-byte value is assigned to object offset `0x80`. Only then is the four-byte membership field at `0x7C` read.

If membership is nonzero, the function returns. This exit does not undo or bypass the preceding notification, configuration, helper invocation, or preparation assignment.

If membership is zero, the function enters scheduling. The membership decision reflects memory after the preceding dependencies, not necessarily the entry value.

### Scheduling entry and duration snapshot

The function reads G4 and passes its current four-byte address-valued contents to F-338d16f9ce72. It then calls F-add44d4e9aaa with the embedded-record address.

Only after both calls return does it read the four-byte duration-like quantity at object offset `0x64`.

A zero quantity skips deadline calculation and list insertion, but not the post-scheduling call or final dispatch gate.

For a nonzero quantity:

1. F-U1 is invoked with no explicit arguments and supplies a four-byte result.
2. G5 is read to obtain the address of list-head storage.
3. The saved duration is multiplied by 1,000 and added to the F-U1 result; the resulting four-byte deadline-like value is written to object offset `0x68`.
4. G6 is read and assigned to object offset `0x78`.
5. The current head link is read through the previously obtained list-head-storage address.

F-U1 is a contract-local opaque label for an unresolved dependency, not a resolved identity.

The duration is captured before F-U1. Changes to object offset `0x64` during that call do not replace the captured value. Conversely, G5 is read after F-U1, and G6 is read at its assignment boundary after the deadline write.

The later deadline calculation is separate from the initial unsigned configuration multiplication. Its operands and destination are represented as 32-bit quantities, with no visible widening, saturation, or overflow rejection. Exact signed-overflow semantics are unresolved. A mathematical unbounded result must not be substituted for the represented calculation.

Only zero is excluded. A duration bit pattern interpreted as negative is not rejected by a positivity check.

### List traversal and insertion

Traversal begins at the loaded head with no predecessor.

For each non-null current node, F-0d70d6f9010c receives:

- The current node’s key address, node offset `0x08`.
- The new record’s key address, object offset `0x68`.
- This function’s third and fourth arguments.
- Three additional apparent four-byte inputs whose origins and meanings are unresolved.

A zero result continues traversal. Any nonzero result stops before the current node; its sign and magnitude receive no further local interpretation.

After a zero result, the current node’s next-link is read **after the comparison call returns**. That node becomes the predecessor, and traversal proceeds to the just-read successor. Comparator mutations can therefore affect the next traversal step.

When traversal stops, the direct writes occur in this order:

1. Publish the embedded-record address through either the saved list-head storage, if there is no predecessor, or the predecessor’s next-link.
2. Write four-byte one to object offset `0x7C`.
3. Write the stopping node address, possibly null, to the embedded record’s next-link at object offset `0x60`.

The list-head-storage address obtained from G5 is retained for this insertion. A later change to G5 does not redirect that retained publication location.

Publication precedes membership and next-link finalization. The visible body alone does not establish atomic insertion or safe observation by concurrent readers. It also does not prove sorted order, duplicate prevention, or equality handling beyond the comparator’s zero/nonzero rule.

### Post-scheduling call and shared-indirection dispatch gate

After either a zero duration or completed insertion, G7 is read and its current value is passed to F-5bcae2eff9a7.

After that dependency returns, final gating uses **G4 again**:

1. Reload the current address-valued contents of G4.
2. Dereference that address to obtain a dispatch-state pointer.
3. If the dispatch-state pointer is null, skip its contents and skip dispatch.
4. Otherwise, read the four-byte state value at that pointer.
5. If that state value is nonzero, obtain the indirect target through G8 and invoke it.

The address loaded from G4 is not itself null-checked before being dereferenced. The null check applies to the dispatch-state pointer obtained from that dereference.

The scheduling-entry argument and the final gate therefore share a global slot, but do not share a cached value. Intervening dependencies may change G4, the storage it addresses, or the dispatch-state contents. The final gate follows the slot’s current value after F-5bcae2eff9a7, even if it differs from the earlier scheduling-entry argument.

The indirect invocation, labeled F-U2 here, has no explicitly established arguments or checked result. Its target is not independently checked for null or callability. Neither the sentinel exit nor the nonzero-membership exit reaches this sequence.

## Side effects

Direct writes can affect:

- Object byte `0x8C`: initial zero; later one only on the above-threshold path.
- The freshly selected associated object’s `0x48C` field: saved input minus 30.
- Object field `0x80`: current G3 value.
- Object field `0x68`: computed deadline-like value.
- Object field `0x78`: current G6 value.
- Global list-head storage or a predecessor’s next-link.
- Object field `0x7C`: one during insertion.
- Embedded next-link at object offset `0x60`: stopping successor.

Opaque dependencies may cause additional changes. Consequently, a direct assignment does not establish the field’s final value after subsequent calls.

No allocation, copying of a node, or deallocation is explicit. The inserted node is storage embedded in the supplied object. Insertion exposes that storage through the list; safe subsequent list use requires adequate object lifetime. Ownership transfer, removal obligations, and any dependency-managed release are unknown.

The membership check occurs before F-338d16f9ce72 and is not explicitly repeated afterward. The surrounding calls do not by themselves prove locking or lock release.

No explicit error result, rollback, exception translation, or recovery mechanism is visible. A non-returning dependency, invalid access, or nonterminating traversal can leave only a prefix of the effects completed.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-936a8728139d | First called with the embedded-record address; later invoked with unresolved arguments. |
| F-f36c23c5ba7b | Receives a freshly loaded associated-object address and the saved unsigned value. |
| F-6904f7699031 | Receives the embedded-record address and selected four-byte configuration. |
| F-338d16f9ce72 | Receives the scheduling-entry value loaded from G4. |
| F-add44d4e9aaa | Receives the embedded-record address before the duration read. |
| F-0d70d6f9010c | Controls traversal through zero versus nonzero results; receives two key addresses and five additional apparent inputs. |
| F-5bcae2eff9a7 | Receives the current G7 value before G4 is reloaded for dispatch gating. |
| F-U1 | Unresolved call supplying the four-byte deadline base; no explicit arguments shown. |
| F-U2 | Unresolved indirect invocation reached through G8 when the final gate succeeds. |

F-U1 and F-U2 are local opaque labels only. Dependency bodies, failure modes, memory effects, calling conventions beyond visible arguments, and ownership effects are unavailable.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Outcomes assume reached accesses are valid and relevant dependencies return normally.

1. **Sentinel snapshot.** With initial associated value `0x0000FFFF`, the first helper runs, the flag is directly cleared, and the function returns without notification or scheduling. A helper change to the associated value does not change this decision.

2. **Threshold 30 with flag mutation.** With saved value 30, suppose notification writes seven to object byte `0x8C`. The at-most-30 branch does not restore zero. The selected configuration remains the pre-notification 32-bit product.

3. **Product width.** With saved value two and G1 equal to `0x80000001` at its read, the selected product has bit pattern `0x00000002`. This does not imply that a later helper preserves that configuration in object storage.

4. **Threshold 31 with changing replacement.** With saved value 31, notification may change G2. The function subsequently writes flag one, reads the then-current G2, and writes one to the associated field selected by the later object-pointer reload.

5. **Unsigned upper range.** Saved value `0xFFFFFFFF` takes the above-threshold path and produces associated-field value `0xFFFFFFE1`. It is not treated as negative for threshold selection. The initial 32-bit product is still formed before being discarded.

6. **Already marked.** If membership is nonzero when checked after preparation, scheduling entry is skipped. The preceding G3 read and assignment to offset `0x80` still occur.

7. **Zero duration.** If membership permits scheduling but the post-helper duration is zero, no direct deadline or insertion write occurs. The G7 call and shared-indirection dispatch gate still execute.

8. **Empty list.** With nonzero duration and a null head, no comparator call occurs. Publication of the new head precedes writing membership one, which precedes writing the new next-link as null.

9. **Comparator changes successor.** If a comparison returns zero after replacing the current node’s next-link, traversal follows the replacement link read after that call.

10. **Cyclic traversal.** A readable cycle with comparisons that always return zero can run indefinitely without reaching insertion, the post-scheduling dependency, or dispatch.

11. **Changed shared slot.** Suppose scheduling entry receives address A from G4, but F-5bcae2eff9a7 leaves G4 containing address B. Final gating dereferences B, not cached A. A null dispatch-state pointer read through B suppresses dispatch; a non-null pointer to nonzero state permits F-U2, subject to a valid target.

## Uncertainties

The following remain `documented_only`:

- Numeric contents, units, and application meanings of global-backed values.
- The second F-936a8728139d invocation’s argument interface.
- F-U1’s identity, time domain, units, monotonicity, and wraparound interpretation.
- Exact overflow semantics of the signed-looking deadline expression.
- Comparator ordering, equality policy, key extent, and three unrecovered inputs.
- Dependency effects on globals, object fields, links, ownership, and control flow.
- Synchronization, visibility of partial insertion, and reentrancy guarantees.
- Meanings of the dispatch state and F-U2’s full interface.
- Whether helpers remove an existing record or normalize membership.

## Evidence

Established static evidence is limited to the supplied function representation and opaque dependency mapping. It supports the access widths and offsets, unsigned snapshot and threshold decisions, initial 32-bit product, global-read boundaries, shared G4 indirection, traversal rule, publication order, and dispatch conditions described above.

Scheduling terminology and helper roles are inferred intent, not established dependency semantics. No dependency bodies, resolved global contents, complete calling-convention specification, or concurrency model were supplied.

No runtime tests have been performed.
