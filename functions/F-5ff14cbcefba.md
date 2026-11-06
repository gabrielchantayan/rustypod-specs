<!-- Trusted provenance: F-5ff14cbcefba; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5ff14cbcefba

## Purpose

F-5ff14cbcefba conditionally updates an object, compares two indirect query results, and performs additional notifications when those results differ. Every active invocation that completes normally also performs a final externally resolved indirect dispatch.

The established contract concerns observable accesses, calls, arguments, comparisons, and writes. Application-level meanings such as selection, resource release, or user-interface notification are not established.

Two paths are distinguished:

- **Inactive:** the initial dependency returns zero. No subsequent operation shown in this function executes.
- **Active:** the initial dependency returns nonzero. The function obtains a query result, operates on embedded components, stores a result field, invokes another dependency, queries again, conditionally notifies, and performs the final dispatch.

The notification sequence uses two distinct reads of external notification-value storage on the changed-result path. The first is retained for the embedded-receiver notification; the second supplies F-069a1d444a2d. Their values are not guaranteed to match.

## Interface

### Input and result

The function accepts one object address and produces no explicit return value.

The supplied representation uses 32-bit integer words and 32-bit address-bearing fields. All offsets below are byte offsets. This describes the supplied representation, not a portable native-pointer interface.

No object-address validation, extent check, dispatch-table validation, or general null check is visible.

### Object storage accessed

| Offset | Width or interpretation | Use |
|---|---|---|
| `0x00` | 32-bit dispatch-table address | Resolves each object query through table offset `0x158`. |
| `0x14` | Embedded receiver whose first word is a dispatch-table address | Receives the changed-result indirect notification through table offset `0x14`. |
| `0x54` | 32-bit optional associated-object address | Controls and addresses the conditional propagated write. |
| `0x5C` | 32-bit result field | Receives the selected result and may later supply the propagated value. |
| `0x60` | Embedded component address | Passed to F-b7abcf60c990 and F-e41650e0b8d9. |
| `0x64` | 32-bit value | Read after F-unresolved-01 and passed to F-869599a633dd. |
| `0x88` | Embedded component address | Passed to F-unresolved-01. |
| `0x948` | 32-bit value | Supplies an argument to the final indirect dispatch. |

The embedded components’ extents are unknown. In particular, the field at `0x64` may belong to the component beginning at `0x60`; the table does not establish separate ownership or nonoverlapping objects.

When the associated-object address is nonzero, this function writes a 32-bit value at offset `0x1C` from that address.

### External storage

Two external storage locations are referenced:

- **Notification-value storage:** a 32-bit value source. Every active path reads it once after the second query and retains that result. The changed-result path reads it again for F-069a1d444a2d. The embedded-receiver notification instead receives the retained result of the first read.
- **Dispatch-anchor storage:** an address source used by the final dispatch. A field at offset `0x04` from the anchor identifies a receiver. That receiver’s dispatch table supplies the target at offset `0x5C`.

No dependency call intervenes between the two notification-value reads. Nevertheless, the evidence does not establish that the storage cannot change independently or that the values must be equal.

### Preconditions for normal completion

Every executed read and write must designate valid memory with the required permissions. Executed indirect targets must be callable with the observed argument layout.

Dependencies must return normally for later operations to occur. Object storage, embedded receivers, external storage, and dispatch structures must remain usable whenever accessed. The function itself does not enforce those conditions or visibly establish synchronization.

Pointer arithmetic is used to form interior addresses and field addresses. No bounds or address-overflow handling is shown. The function performs no substantive integer arithmetic on query results; their later comparison is equality-based.

## Behavior

### Gate and inactive path

The function calls F-b7abcf60c990 with the address at object offset `0x60`.

A zero 32-bit return ends the invocation immediately. On this path, the function performs neither query, neither explicit store, neither changed-result notification, and no final dispatch.

This is not a side-effect-free guarantee. F-b7abcf60c990 may itself affect memory or external state.

Any nonzero return selects the active path. The particular nonzero value has no further visible use.

### First query and component operation

The active path resolves an indirect query through the object’s dispatch-table address at offset `0x00`, selecting table offset `0x158`. The original object address is the receiver.

The returned 32-bit value is retained across subsequent calls. It serves both as an argument to F-e41650e0b8d9 and as the earlier result in the eventual query comparison.

F-e41650e0b8d9 receives the component address at object offset `0x60` and that first query result. Its 32-bit return is also retained.

Next, F-unresolved-01 receives the component address at object offset `0x88`. After it returns, the function reads the 32-bit value at object offset `0x64` and passes that value to F-869599a633dd.

The `0x64` read therefore observes storage after the component operation and F-unresolved-01, not a value captured before those calls. No return from F-unresolved-01 or F-869599a633dd is visibly consumed.

### Result-field selection and write

Only after F-869599a633dd returns does the function select the value for object offset `0x5C`:

- A zero return retained from F-e41650e0b8d9 selects the 32-bit all-one pattern, `0xFFFFFFFF`, representing signed `-1`.
- A nonzero retained return is passed unchanged to F-6fa4e01d8d29, together with decimal 51 (`0x33`) and zero. That dependency’s 32-bit return is selected.

The selected value is then written to object offset `0x5C`.

A zero component result does not terminate the active path or skip the intervening calls. A nonzero component result is not restricted to positive values. The transformation result is stored without a visible range check or special handling for zero, negative values, or `0xFFFFFFFF`.

### Subsequent call and second query

After the result-field write, F-b582c09a05ae receives the original object address.

The function then performs a second indirect query through the object’s current dispatch-table address, again selecting table offset `0x158` and passing the original object address as receiver.

The dispatch-table lookup is performed again; the target need not be the same as for the first query if intervening behavior changed the relevant storage.

After this query returns, the function reads notification-value storage and retains the resulting 32-bit value. This first notification-value read occurs on every active path, even when the two query results are equal.

### Changed-result notifications and propagation

The two retained query results are compared for 32-bit inequality. No subtraction, magnitude comparison, or signed ordering is involved.

If the results are equal, the function skips the changed-result notifications and associated-object propagation. It does not perform the second notification-value read.

If the results differ, the following ordering is established:

1. Notification-value storage is read again to supply F-069a1d444a2d’s second argument. That dependency receives the original object address as its first argument.
2. After F-069a1d444a2d returns, an indirect notification is resolved through the embedded receiver at object offset `0x14`, using its dispatch-table entry at offset `0x14`.
3. After that indirect notification returns, the associated-object field at object offset `0x54` is tested. A nonzero value enables the propagated write.

The embedded-receiver notification receives three arguments:

- The embedded receiver address, at object offset `0x14`.
- The retained value from the **first** notification-value read, performed after the second query.
- Zero.

Thus F-069a1d444a2d receives the changed-path reread, whereas the embedded receiver receives the earlier captured value. The source does not establish that those arguments are equal. A change to notification-value storage during F-069a1d444a2d also does not replace the retained argument.

When propagation is enabled, the function writes the object’s then-current 32-bit field at `0x5C` to offset `0x1C` within the associated object. The source and destination-address expressions are evaluated after both notifications, so this need not propagate the value originally written before F-b582c09a05ae.

The representation references the associated-object field both for the null test and for the destination address. It does not establish one retained address spanning both uses. Exact evaluation ordering within the assignment and behavior under independently changing storage remain unresolved.

### Final dispatch

Every active invocation reaching this stage performs a final indirect dispatch, regardless of whether the query results differed or the associated-object write occurred.

Its resolution uses dispatch-anchor storage, the receiver-address field at anchor offset `0x04`, and that receiver’s dispatch-table entry at offset `0x5C`.

The call receives the externally obtained receiver and a 32-bit value read from the original object at offset `0x948`.

This operation follows all applicable notifications and propagation. Neither the external receiver nor the `0x948` argument is established as an earlier snapshot. Exact ordering among reads within this indirect-call expression is not established, nor is consistency among repeated references if the underlying storage changes independently.

The supplied representation flags uncertainty about the original indirect control transfer and presents it as a call. On the represented normal path, the function returns afterward without an explicit result.

## Side effects

### Explicit writes

Two explicit write sites are established:

- Every active path that reaches result selection writes 32 bits to object offset `0x5C`.
- A changed-result path with an enabled associated-object write stores 32 bits at associated-object offset `0x1C`.

No explicit write is shown on the inactive path. Dependencies may introduce additional effects on either path.

### Read timing

Several reads intentionally occur after potentially mutating calls:

- The `0x64` field is read after F-unresolved-01.
- The second query resolves the object’s dispatch table again after F-b582c09a05ae.
- Notification-value storage is first read after the second query and reread only on the changed-result path.
- The embedded receiver’s dispatch target is resolved after F-069a1d444a2d.
- Associated-object propagation uses storage evaluated after the embedded notification.
- Final-dispatch state is accessed after the preceding work.

These timing properties do not imply synchronization, atomic snapshots, or protection against reentrancy.

### Ownership and dependency effects

No allocation, release, ownership transfer, reference-count adjustment, or lifetime guarantee is directly established.

Passing an interior address or stored value to a dependency does not prove that the dependency owns, preserves, or releases anything. In particular, F-unresolved-01 cannot be classified as a destructor, and F-869599a633dd cannot be classified as a release operation from this evidence.

Dependencies may modify object fields, external storage, dispatch tables, or lifetimes; may invoke callbacks; or may fail to return. Such behavior is not specified by the supplied function alone.

### Errors and abnormal completion

There is no visible error return, exception handler, rollback, or aggregate status.

The initial zero result is a gate outcome. A zero component result selects the all-one result-field value. Neither is established as an application-level error.

The associated-object null test protects only the conditional propagated write. Other pointer chains are unguarded. Invalid accesses, incompatible indirect targets, and abnormal dependency completion have no established recovery behavior here.

## Dependencies

The unresolved IDs are document-local opaque identifiers. An indirect-operation ID describes a call-site role, not a guarantee of one immutable runtime target.

| Function ID | Observed call-site contract |
|---|---|
| F-b7abcf60c990 | Receives the component address at `0x60`; returns the 32-bit zero/nonzero gate result. |
| F-e41650e0b8d9 | Receives that component address and the first query result; returns the 32-bit value controlling result selection. |
| F-unresolved-01 | Receives the component address at `0x88`; completes before the `0x64` field read. No return is visibly used. |
| F-869599a633dd | Receives the current 32-bit value at `0x64`. No return is visibly used. |
| F-6fa4e01d8d29 | Receives a nonzero retained component result, 51, and zero; returns the value stored at `0x5C`. |
| F-b582c09a05ae | Receives the original object address after the result-field write. |
| F-unresolved-02 | Object query through table offset `0x158`; called twice on the active path, with separately resolved targets and 32-bit results. |
| F-069a1d444a2d | Called only when query results differ; receives the original object address and the **second** read of notification-value storage. |
| F-unresolved-03 | Embedded-receiver notification through table offset `0x14`; receives the receiver address, the retained **first** notification-value read, and zero. |
| F-unresolved-04 | Final externally resolved dispatch through table offset `0x5C`; receives its receiver and the object’s current `0x948` value. Original transfer details remain unresolved. |

## Acceptance examples

These are conditional contract examples, not executed tests. Unless stated otherwise, all accesses are valid and dependencies return normally.

1. **Inactive gate.** If F-b7abcf60c990 returns zero, no later dependency, query, notification-value read, explicit write, or final dispatch occurs.

2. **Zero component result.** If the gate is nonzero and F-e41650e0b8d9 returns zero, F-unresolved-01 and F-869599a633dd still execute before `0xFFFFFFFF` is written to `0x5C`. F-6fa4e01d8d29 is skipped.

3. **Nonzero result transformation.** If F-e41650e0b8d9 returns `0xFFFFFFFF`, that nonzero bit pattern is supplied to F-6fa4e01d8d29 with 51 and zero. Its returned value is stored without filtering.

4. **Equal query results.** If both queries return 12, notification-value storage is read once. Neither changed-result notification nor associated-object propagation occurs. The final dispatch still follows.

5. **Different notification reads.** Suppose the query results differ, the first notification-value read yields 7, and the changed-path reread yields 9. F-069a1d444a2d receives 9; F-unresolved-03 receives 7. No dependency call is required between those reads for this distinction to matter; whether the environment permits such a change is unresolved.

6. **Mutation during the first notification.** Suppose both notification-value reads yield 7, then F-069a1d444a2d changes the storage to 11. F-unresolved-03 still receives the retained 7.

7. **No associated object after notification.** If the queries differ but the associated-object field is zero when tested after F-unresolved-03, both notifications have already occurred. Propagation is skipped, and the final dispatch follows.

8. **Current propagated result.** If a notification changes `0x5C` to 42 and a valid associated-object address remains consistently available through the test and destination evaluation, the propagated value is 42 rather than necessarily the earlier stored result.

9. **Equality boundary.** Query bit patterns `0xFFFFFFFF` and `0x00000000` select the changed-result path. Matching `0xFFFFFFFF` patterns do not. No overflow-sensitive subtraction is involved.

## Uncertainties

The following remain **documented_only**:

- Application meanings of the object, fields, notifications, and dependency results.
- Full layouts, component extents, alignment requirements, and byte order.
- Dependency allocation, release, ownership, lifetime, failure, and reentrancy behavior.
- Purity or stability of either query.
- Whether notification-value storage can change independently between its two reads.
- Synchronization, atomicity, and validity under concurrent mutation.
- Exact intra-expression read ordering and consistency of repeated pointer references.
- Original control-transfer details for the final dispatch.
- Portability beyond the supplied 32-bit representation.

An update followed by change-sensitive notification is a reasonable interpretation of intent, not an established application-level contract.

## Evidence

This contract is derived from the supplied static function representation and dependency identity map. Prior prose and reviewer feedback are not independent evidence.

The representation establishes the branch structure, call ordering, field offsets, argument constants, 32-bit result handling, explicit stores, and equality comparison. It separately references notification-value storage after the second query and again as an argument on the changed-result path; the embedded-receiver notification uses the value retained from the earlier reference.

Callee implementations and runtime effects are unavailable. No runtime tests, execution traces, or independent binary validation have been performed.
