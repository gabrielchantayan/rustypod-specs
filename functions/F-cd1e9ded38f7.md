<!-- Trusted provenance: F-cd1e9ded38f7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cd1e9ded38f7

## Purpose

F-cd1e9ded38f7 selects one of two paths using a 32-bit mode argument:

- **Zero mode:** run two prerequisite checks, invoke object-processing dependencies, and issue an indirect notification if two object fields differ.
- **Nonzero mode:** update an object field, inspect an activation-like field, and, when that field is zero, prepare an embedded node for possible list insertion before a final dispatch check.

The function has no established uniform success or failure convention. Its result may be a prerequisite result, zero, a freshly read object field, or the result of an indirect call.

“Notification,” “activation,” “interval,” and “deadline” describe inferred roles. The established contract consists of the accesses, calls, conditions, and result propagation below; application-level meanings remain unresolved.

## Interface

### Arguments and result

| Item | Representation | Requirements and interpretation |
|---|---|---|
| Object address | 32-bit address-valued argument | Must support the selected path’s direct accesses and any additional requirements imposed by dependencies. |
| Mode | 32-bit integer | Zero selects change processing. Every nonzero value selects the other path; its magnitude and sign have no further visible use. |
| Result | 32-bit integer-valued result | Determined by the executed path. No universal error encoding or Boolean normalization is established. |

The supplied representation uses 32-bit addresses and words. It does not establish an equivalent interface for wider addresses. Valid address arithmetic and suitably accessible storage are preconditions for normal execution; no local validation establishes them.

### Object-relative locations

Offsets are byte offsets from the object address. **Direct word accesses are 32 bits. The reference at `+0x38` is address-only:** this function forms and passes that address without establishing a direct read, direct write, or subobject extent there.

| Offset | Access category | Visible role |
|---|---|---|
| `+0x30` | Direct 32-bit reads and conditional write | Recorded value used in zero-mode comparison and update. |
| `+0x34` | Direct 32-bit reads | Current value compared with, then separately read for copying into, `+0x30`. |
| `+0x38` | Address formation only | Argument to F-59222578e3f0; layout and storage extent are unknown. |
| `+0x44` | Address formation and direct 32-bit write | Embedded-node address; its first word becomes the next-node link. |
| `+0x48` | Direct 32-bit read | Interval-like value; zero suppresses insertion work. |
| `+0x4c` | Direct 32-bit write and address formation | Deadline-like value; its address is supplied to the comparator. |
| `+0x5c` | Direct 32-bit write | Receives an opaque global word during insertion preparation. |
| `+0x60` | Direct 32-bit reads and conditional write | Activation-like field; insertion writes one. The already-active path reads it separately for the branch and return. |
| `+0x64` | Direct 32-bit write | Receives an opaque global word on every nonzero-mode invocation after the initial dependency returns. |

This is an access map, not a complete object definition. In particular, passing the address at `+0x38` does not establish that this subobject is one word long.

### External storage

Each traversed list node supplies:

- A 32-bit next-node address at node offset `+0`.
- A comparison-key address at node offset `+8`.

The comparator’s access width and key extent are not established. For the embedded node at object offset `+0x44`, the corresponding key address is object offset `+0x4c`.

An opaque global supplies the address of the list-head cell. A different global supplies the address of a dispatch cell containing a pointer. The dispatch pointer’s first word is inspected only when that pointer is nonnull.

## Behavior

### Zero mode: prerequisite gates

The first prerequisite sequence calls F-6ac70711bbb5, discards its result, and then calls F-17d69da16882. A nonzero result from F-17d69da16882 is returned immediately.

If that result is zero, the function calls F-6ac70711bbb5 again, discards its result, and calls F-b7a2a0b657d4. A nonzero result from this second prerequisite is returned immediately.

Only two zero prerequisite results permit object processing. These early exits perform none of the function’s explicit object writes described below, although dependencies may have side effects.

### Zero mode: processing and comparison

After both gates permit continuation:

1. F-59222578e3f0 receives the address at object offset `+0x38`.
2. F-0f44a6c02122 receives the object address and an address into the current invocation’s stack frame.
3. The words at `+0x34` and `+0x30` are compared.

The processing-call results are not used to choose the next branch. Their effects can nevertheless determine the compared values.

If the words compare equal, the function returns zero. It performs no later notification construction or indirect receiver call.

If they differ, the function separately reads `+0x34` and writes that value to `+0x30`. The relative order of the two reads used for the equality comparison is not established. The subsequent assignment is a distinct access, not an explicitly retained comparison operand. Changes between comparison and assignment can therefore affect the value written.

The stack-frame address has no established public layout, initialization contract, or permitted retention period. No lifetime beyond this invocation is supplied by the visible function.

### Zero mode: notification construction and indirect call

After the recorded-value write, the function performs **two separate, ordered reads of the same notification global**:

- The first read is retained for the second argument to F-1958950ee503.
- A separate, later read is used in an addition of one to form the third argument.

The third argument is therefore not necessarily the successor of the second argument. That relationship requires the global to have the same value at both reads. The function does not visibly increment or otherwise write this global.

After both reads and the addition, F-ef0964bd9b8a is called. Its word-sized result becomes the first argument to F-1958950ee503. The four arguments to that dependency are, in order:

1. The result of F-ef0964bd9b8a.
2. The retained first notification-global read.
3. One added to the separate second notification-global read.
4. The literal value `6`.

The function then calls F-76b1c50b12f7 to obtain a receiver address. It reads the receiver’s first word as a table address, obtains the callable entry at table byte offset `+4`, and invokes it with the receiver and the result of F-1958950ee503. The indirect call’s result is returned.

The recorded-value write precedes all of these notification operations. No rollback is visible if a later dependency fails or does not return normally.

The notification operands and result are represented as 32-bit integers. Overflow behavior for the addition is unresolved; no wraparound, saturation, or overflow rejection is specified.

### Nonzero mode: common work and already-active return

The function forms the embedded-node address at `+0x44` and calls F-936a8728139d with no explicit arguments shown. After that call returns, it reads an opaque global word and writes it to `+0x64`.

It then reads `+0x60` for a zero/nonzero branch decision.

If this branch-test read is nonzero, the function **reads `+0x60` again** and returns the second read’s value. It does not explicitly retain the branch-test value for the return. Consequently:

- An unchanged field yields the value observed by the branch.
- A changed field yields the later value, including zero if it becomes zero.
- A later zero does not cause entry into the inactive-record path; that branch decision has already been made.

The initial dependency and `+0x64` write occur even on this already-active path. F-936a8728139d may affect the field subsequently tested; preservation is not assumed.

### Nonzero mode: inactive-record preparation

If the branch-test read of `+0x60` is zero, the function:

1. Reads the dispatch-related global value and passes it to F-338d16f9ce72.
2. Calls F-add44d4e9aaa with the embedded-node address.
3. Reads the interval-like word at `+0x48`.

If the interval is zero, it skips the value-producing call, deadline write, tag write, list traversal, insertion, and explicit activation write. It still proceeds to the release-and-dispatch sequence.

If the interval is nonzero, it calls an unmapped value-producing dependency with no explicit arguments shown. It retains that result and obtains the address of the global list-head cell.

It then writes the deadline-like word at `+0x4c`, combining the dependency result with the previously sampled interval multiplied by decimal `1000`. A later mutation of `+0x48` does not replace that sampled operand.

Next, it reads another opaque global word and writes it to `+0x5c`. Only after these writes does it read the initial node address from the retained list-head cell.

### Nonzero mode: traversal and insertion

Traversal begins with no predecessor. For each nonnull current node, F-0d70d6f9010c receives:

- The current node address plus eight bytes.
- The candidate key address at object offset `+0x4c`.

A zero comparator result means continue. The next-node word is read from the current node after the comparator returns, and that current node becomes the predecessor.

Every nonzero comparator result stops traversal at the current node. Positive and negative results are not distinguished. Traversal also stops on a null current node, without calling the comparator for null.

On termination, insertion makes these direct writes in order:

1. Publish the embedded-node address through the retained list-head cell if there is no predecessor, or through the predecessor’s first word otherwise.
2. Write `1` to object offset `+0x60`.
3. Write the stopped-at node address, possibly null, to the embedded node’s first word at `+0x44`.

The new node is thus published before its own next-link is written. The function alone does not establish exclusive access or conceal this intermediate state.

The list-head-cell address is retained across traversal. Comparator calls are not assumed to preserve globals, object fields, or list storage. The comparator receives key addresses, so later calls can inspect changed key contents.

No traversal bound, cycle check, duplicate-membership check, or validation of the head-cell address is visible. Termination is not guaranteed for a cycle whose comparisons keep returning zero.

### Nonzero mode: release and final dispatch

After inactive-record processing, with or without insertion, the function reads an opaque global value and passes it to F-5bcae2eff9a7.

After that dependency returns, it freshly reads the dispatch-related global, accesses the designated cell, and obtains the pointer stored there. It does not explicitly reuse the value supplied earlier to F-338d16f9ce72.

- A null pointer causes a zero return.
- A nonnull pointer causes its first 32-bit word to be read.
- A zero first word causes a zero return.
- A nonzero first word causes a separate global callable target to be invoked with no explicit arguments shown. Its result is returned.

The inspected first word is a condition, not the identified callable target. Insertion does not bypass this check or independently determine the function’s return value.

### Arithmetic boundaries

The interval, value-producing result, and deadline destination are represented as 32-bit quantities. Every nonzero interval enters the calculation, including negative values under a signed interpretation.

Neither the multiplication by `1000` nor the subsequent addition has a visible overflow check. The supplied evidence does not resolve intermediate overflow or establish modular arithmetic. The same limitation applies to the notification addition. Predictions involving overflowing arithmetic remain documented_only rather than being assigned assumed wraparound results.

## Side effects

Direct writes, depending on the path, affect `+0x30`, `+0x44`, `+0x4c`, `+0x5c`, `+0x60`, `+0x64`, and either the list-head cell or a predecessor’s next-link. No direct allocation or deallocation is visible.

Insertion publishes an address inside the supplied object. It does not visibly allocate or copy a node. Ownership transfer, removal protocol, and the required duration of the object’s validity are unknown. Any later traversal requires the published storage to remain valid for that access.

Dependencies may modify globals, object fields, list nodes, or other state. Their retention, allocation, and release behavior is not established. In particular, the embedded-node call must not be assumed to unlink, initialize, or free that node.

The calls surrounding inactive-record work suggest synchronization or release intent, but no lock, interrupt, atomicity, or memory-order contract is established.

There are no visible null checks for the object, receiver, method table, list-head-cell address, dispatch-cell address, or selected callable targets. Invalid storage and invalid callable addresses have no specified recovery path. No exception handler, rollback, or cleanup-on-abnormal-exit mechanism is visible.

## Dependencies

Mapped dependencies are identified only by their opaque IDs.

| Dependency | Established use |
|---|---|
| F-6ac70711bbb5 | Called before each zero-mode prerequisite; result unused. |
| F-17d69da16882 | First prerequisite; nonzero result returned. |
| F-b7a2a0b657d4 | Second prerequisite; nonzero result returned. |
| F-59222578e3f0 | Receives the address at `+0x38`; result unused. |
| F-0f44a6c02122 | Receives object and stack-frame addresses; result unused. |
| F-ef0964bd9b8a | Supplies the first notification-construction argument. |
| F-1958950ee503 | Receives four notification arguments; result passed to the receiver method. |
| F-76b1c50b12f7 | Supplies the indirect-call receiver. |
| F-936a8728139d | Called before the `+0x64` write and activation test. |
| F-338d16f9ce72 | Receives a global value on entry to inactive-record work. |
| F-add44d4e9aaa | Receives the embedded-node address before the interval read. |
| F-0d70d6f9010c | Receives two key addresses; zero continues traversal, nonzero stops it. |
| F-5bcae2eff9a7 | Receives a global value before final dispatch inspection. |

Opaque identities are unavailable for three additional callable dependencies: the value source used in deadline calculation, the receiver-table entry at byte offset `+4`, and the final global indirect target. Their identities and broader contracts remain documented_only.

No dependency is assumed to preserve globals, preserve object state, own or release memory, or return normally.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Unless specified otherwise, storage is valid, dependencies return normally, arithmetic is representable, and there is no unmentioned mutation or interference.

1. **First prerequisite rejects.** In zero mode, F-17d69da16882 returning `9` produces result `9`. The second prerequisite and object processing are not reached.

2. **Second prerequisite rejects.** If the first prerequisite returns zero and F-b7a2a0b657d4 returns `−3`, the result is `−3`. F-6ac70711bbb5 has been called twice; F-59222578e3f0 is not reached.

3. **Equal processed fields.** If both gates permit continuation and the compared fields are equal after processing, the result is zero and no notification is constructed.

4. **Commit before notification.** If the compared fields differ and the assignment reads `42` from `+0x34`, `+0x30` receives `42` before either notification-global read. An eventual receiver-method result of `27` becomes the function result.

5. **Distinct notification reads.** If the first notification-global read yields `10` and the later read yields `20`, F-1958950ee503 receives `10` as its second argument and `21` as its third—not `11`.

6. **Stable already-active field.** If `+0x60` yields `7` both when tested and when read for return, the result is `7`. The initial dependency and `+0x64` write have already occurred; inactive-record work is skipped.

7. **Changing already-active field.** If the branch read yields `7` and the separate return read yields zero, the result is zero. No insertion or release-and-dispatch sequence follows that return.

8. **Zero interval.** If the activation test yields zero and the later interval read yields zero, insertion-specific work is skipped. F-5bcae2eff9a7 and dispatch inspection still occur. A null dispatch pointer yields zero.

9. **Empty-list insertion.** With a nonzero interval and an empty list, no comparator call occurs. The head receives the embedded-node address, activation receives one, and the embedded next-link receives null, in that order.

10. **Insertion between nodes.** If the comparator returns zero for the first node and any nonzero value for the second, insertion occurs after the first and before the second.

11. **Representable deadline.** A sampled interval of `3` and value-source result of `200` produce a stored deadline of `3200`. An overflowing calculation has no specified numeric outcome here.

12. **Dispatch after insertion.** If insertion completes, the dispatch pointer and its first word are nonzero, and the final target returns `11`, the function returns `11`, not the activation value.

## Uncertainties

The following remain documented_only:

- Application meanings, units, result codes, and global-value roles.
- Overflow behavior for notification addition, interval multiplication, and deadline addition.
- Comparator ordering, equal-key treatment, key extent, and wrap-aware behavior.
- Stack-frame storage layout, initialization, and permitted retention.
- Dependency ownership, allocation, release, preservation, and abnormal-exit behavior.
- Synchronization guarantees and visibility of intermediate writes.
- Defined behavior under concurrent mutation, invalid addresses, duplicate membership, or cyclic lists.
- Identities and complete interfaces of the three unmapped callables.
- Completeness of the supplied representation, which reports omitted unreachable regions and an incompletely recovered indirect transfer.

## Evidence

Established behavior is derived solely from the supplied static function representation and dependency mapping. The prior draft and reviewer feedback are not independent evidence.

The representation specifically supports two ordered notification-global reads, distinct activation branch and return reads, and address-only treatment of `+0x38`. It also supports the mode split, direct word widths, call sequence, traversal condition, insertion write order, and path-specific return sources described above.

Scheduling and notification terminology remains inferred intent. No runtime tests have been performed; acceptance examples are conditional static predictions, not observed results.
