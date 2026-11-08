<!-- Trusted provenance: F-5a1d348cfcc9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5a1d348cfcc9

## Purpose

F-5a1d348cfcc9 conditionally writes an indexed byte in caller-supplied state and coordinates operations on an associated object.

Two request classes select different behavior:

- A zero request clears the indexed byte. Additional gates determine whether an object operation, an activation-marker clear, two state operations, and six conditional notifications follow.
- Any nonzero request sets the indexed byte. If the activation marker is zero after that write, an activation operation occurs, followed by four independently gated updates.

“Activation,” “notification,” and “pending update” are inferred descriptive roles. The established contract concerns field accesses, branch conditions, dependency calls, and direct writes. The dependencies’ domain-specific effects are unknown.

This specification is based on static evidence only. No runtime tests have been performed.

## Interface

### Inputs and result

| Input | Represented width | Interpretation |
|---|---:|---|
| State address | 32 bits | Base address for all state-relative accesses. |
| Index | 32 bits | Unscaled displacement added to the state address and `0x18` to select a writable byte. |
| Request | 32 bits | Zero selects clearing; every nonzero value selects setting. |

The request is compared with zero as an integer; it is not first reduced to a byte. Negative requests and nonzero requests with a zero low byte still select setting.

The supplied representation has no explicit return value. Early rejection, an indexed write without further action, and completion of the longer paths are not distinguished by a returned status.

Only the zero/nonzero result of F-5b4414e97173 affects subsequent behavior. Other dependency results are not used.

### State layout

All offsets are byte offsets from the state address. Roles are descriptive, not recovered original field names.

| Offset | Access width | Established role |
|---|---:|---|
| `0x00`, `0x04`, `0x08`, `0x0C`, `0x10`, `0x14` | Four bytes each | Six independently inspected notification entries. |
| `0x18 + index` | One byte | Indexed destination, written as zero or one. |
| `0x50` | Four bytes | Clearing-path blocker; zero permits the next gate. |
| `0x90` | One byte | Entry eligibility marker; must equal exactly one. |
| `0x94` | Four bytes | Associated-object field. |
| `0x98` | One byte | Activation marker; zero permits activation. |
| `0x99`, `0x9A`, `0x9B` | One byte each | Three activation payloads. |
| `0x9D` | One byte | Payload for F-7e415cb3218c. |
| `0x9E` | One byte | Pending marker governing that payload. |
| `0x9F` | One byte | Payload for F-b2f40e55a309. |
| `0xA0` | One byte | Pending marker governing that payload. |
| `0xA1` | One byte | Payload for the indirect dependency. |
| `0xA2` | One byte | Pending marker governing that payload. |
| `0xA3` | One byte | Payload for F-2d8b0b6097e7. |
| `0xA4` | One byte | Pending marker governing that payload. |

There is no dedicated fixed-offset access to `0x9C` or to unlisted gaps. This does not make those locations inaccessible: the unchecked indexed write can reach them. For example, index `0x84` selects offset `0x9C`.

### Preconditions and arithmetic

The caller must provide valid storage for every access actually reached. There is no initial null check for the state address and no index validation.

The indexed destination is selected by adding the state address, the index, and `0x18`, without scaling the index. The represented calculation uses 32-bit integer values. No explicit overflow check, bounds check, saturation, or recovery path is shown. Behavior for invalid or overflowing address calculations is documented_only; wrapping or fault behavior must not be assumed.

The layout does not establish a declared array boundary or total state-object size. Index values can select other fields or storage outside the listed layout. A destination overlapping a four-byte field changes one byte of that field; the resulting field value depends on its representation and prior contents.

Associated-object values supplied to dependencies must meet those dependencies’ unknown validity requirements. The indirect operation additionally requires valid object, dispatch-table, and method-entry accesses when evaluated.

The represented widths do not independently establish the original ABI, alignment requirements, byte order, or byte-argument extension rules.

## Behavior

### Entry gates

The first state access reads the four-byte associated-object field at `0x94`. A zero value causes immediate return.

Only when that value is nonzero is the eligibility byte at `0x90` read. Eligibility must equal exactly one. Zero and every other byte value cause immediate return.

Neither rejection performs the indexed write or invokes a dependency. The order is significant: rejection on the associated-object field prevents even the eligibility read.

These gates establish eligibility only at entry. They are not repeated before every operation. Later changes to either field do not trigger automatic rejection, and a nonzero associated-object value at entry does not guarantee that later reads remain nonzero.

### Clearing request

After both entry gates pass, a zero request writes zero to the indexed destination before any clearing-specific gate is evaluated.

The four-byte blocker at `0x50` is then read. A nonzero value ends processing, leaving the indexed clear in place. If the blocker is zero, F-5b4414e97173 receives the state address. Its zero result likewise ends processing without undoing the indexed clear.

A nonzero result permits the following ordered sequence:

1. F-f22506ed1cf0 receives a newly read associated-object value from `0x94`.
2. After normal return, this function clears the activation marker at `0x98`.
3. F-bdd02c544bdc receives the state address.
4. After that call returns, F-59798b8211ac receives the state address.
5. After that call returns, the six notification entries are inspected in increasing offset order.

There is no precondition that the activation marker was nonzero. An already-zero activation marker does not suppress this sequence.

Each notification entry is read as a four-byte value. If its value is nonzero at inspection, F-65edd8002f52 is called without explicit arguments. A zero entry produces no call. The entry’s value serves only as a condition; it is not supplied as an explicit argument.

Notification inspection is sequential, not a snapshot. A notification call occurs before the next entry is read, so its effects can change later notification decisions. With normal returns, all six entries are inspected and between zero and six notification calls occur.

This function does not directly clear notification entries or pending markers on this path, except if the indexed write overlaps them. Dependency effects remain unknown.

### Setting request

After both entry gates pass, any nonzero request writes one to the indexed destination. The activation marker at `0x98` is read afterward.

Any nonzero activation marker ends processing immediately. The indexed write remains effective, but neither activation nor pending-update processing occurs.

If the activation marker is zero, F-231941f10f3a receives four arguments in this positional order:

- The current associated-object value from `0x94`.
- The byte payload at `0x99`.
- The byte payload at `0x9A`.
- The byte payload at `0x9B`.

The order in which those argument values are read is not established by the supplied representation.

After normal return, this function writes one to the activation marker and proceeds to pending-update processing. The marker is not set before the call, so it does not serve as a pre-call re-entry guard.

An indexed write that selects `0x98` occurs before the activation check. For a nonzero request, that overlap writes one and therefore suppresses activation.

### Pending updates

Pending updates are considered only after the activation call and the subsequent activation-marker write. They are processed in the following order, not in increasing offset order:

| Order | Marker checked | Operation if marker is nonzero | Write after normal return |
|---:|---|---|---|
| 1 | `0xA0` | F-b2f40e55a309 receives the current associated-object value and the byte at `0x9F`. | Clear `0xA0`. |
| 2 | `0x9E` | F-7e415cb3218c receives the current associated-object value and the byte at `0x9D`. | Clear `0x9E`. |
| 3 | `0xA2` | Invoke the indirect dependency using the dispatch mechanism below and the byte at `0xA1`. | Clear `0xA2`. |
| 4 | `0xA4` | F-2d8b0b6097e7 receives the current associated-object value and the byte at `0xA3`. | Clear `0xA4`. |

Every nonzero marker value is pending; equality to one is not required. A zero marker skips both its operation and its clearing write.

Each marker is checked when its stage is reached. Initial marker values do not determine the entire sequence: the activation dependency or an earlier pending-update dependency can change later markers or payloads.

Clearing occurs after the corresponding call returns. If a call sets its own marker, the subsequent direct clear overwrites that setting. There is no retry within this invocation.

Associated-object values and payloads are read for the individual operations, rather than captured together at entry. Dependencies are not assumed to preserve them.

### Indirect dispatch

The indirect-call expression contains **two separate reads** of the associated-object field at `0x94`:

- One supplies the object used to resolve the dispatch target. A four-byte table address is read from the beginning of that object, and the method entry is selected at byte offset `0x20` within that table.
- Another supplies the explicit first argument to the selected method.

The explicit second argument is the byte payload at `0xA1`.

These are not a single captured object value. Under intervening mutation, the object used to select the method and the explicit object argument need not be identical.

The supplied representation does not establish the evaluation order of dispatch-target resolution relative to argument evaluation, nor the order of the individual argument reads. It therefore does not support a guaranteed sequence between the two associated-object reads.

The marker at `0xA2` is cleared only after the indirect call returns normally.

## Side effects

### Direct writes

Excluding dependency effects, direct writes are limited to:

- Zero or one at the indexed destination, after successful entry checks.
- Zero at `0x98` after F-f22506ed1cf0 returns on the clearing path.
- One at `0x98` after F-231941f10f3a returns on the setting path.
- Zero at each pending marker whose corresponding operation returns normally.

There are no other dedicated field writes. The indexed write can nevertheless overlap any reachable byte, including payloads, notification entries, eligibility, the blocker, the associated-object field, layout gaps, or storage outside the listed layout.

### Ownership and lifetime

No direct allocation or release operation is shown. Existing state addresses and field values are passed to dependencies.

No ownership transfer, retention, destruction, or preservation guarantee can be derived from call placement. F-f22506ed1cf0 must not be assumed either to release the associated object or to preserve it. The same restriction applies to other dependencies.

Subsequent accesses require the relevant state and object storage to remain valid. Responsibility for maintaining those lifetimes is documented_only.

### Errors, re-entry, and concurrency

The function provides no explicit error report, rollback, exception handler, or recovery protocol.

Post-call behavior is conditional on normal return. A fault or nonlocal exit can leave the indexed destination changed, an activation marker not yet updated, or a pending marker uncleared.

No synchronization or atomic multi-field transition is established. Re-entry and concurrent mutation are not visibly prevented. Dependencies may modify state, globals, ownership, or other storage; none of those effects is assumed absent.

## Dependencies

| Opaque ID or unresolved target | Established use |
|---|---|
| F-5b4414e97173 | Receives the state address; zero stops the clearing sequence. Its represented result is a 32-bit integer. |
| F-f22506ed1cf0 | Receives the associated-object value after clearing gates pass. |
| F-bdd02c544bdc | Receives the state address after the activation-marker clear. |
| F-59798b8211ac | Receives the state address after F-bdd02c544bdc returns. |
| F-65edd8002f52 | Called without explicit arguments for each notification entry found nonzero. |
| F-231941f10f3a | Receives the associated-object value and three one-byte activation payloads. |
| F-b2f40e55a309 | Receives the associated-object value and the payload governed by `0xA0`. |
| F-7e415cb3218c | Receives the associated-object value and the payload governed by `0x9E`. |
| F-2d8b0b6097e7 | Receives the associated-object value and the payload governed by `0xA4`. |
| Unresolved indirect dependency | Selected through table offset `0x20`; receives a separately read associated-object value and the payload governed by `0xA2`. No opaque target ID is supplied. |

The table specifies argument roles, not unknown callee contracts. Apart from F-5b4414e97173’s gating result, no return-value meaning is established.

## Acceptance examples

These are conditional criteria derived from static evidence, not executed tests.

Unless an example states otherwise, assume valid reached storage, normal dependency returns, no concurrent or re-entrant mutation, and no indexed overlap with listed fields. **Every post-entry example additionally requires a nonzero associated-object field and eligibility exactly one at their respective entry checks.** Dependency preservation is assumed only where expressly stated.

1. **Associated object absent.** If the initial four-byte read at `0x94` is zero, either request returns without reading eligibility, writing the indexed destination, or invoking a dependency.

2. **Eligibility is exact.** If the initial associated-object value is nonzero but eligibility is two, either request returns before the indexed write. Merely nonzero eligibility is insufficient.

3. **Blocked clearing still writes.** With both entry gates passing, a zero request and a nonzero blocker clear the indexed byte and invoke no dependency.

4. **Clearing gate rejects.** With both entry gates passing, a zero blocker, and a zero result from F-5b4414e97173, the indexed byte is cleared. This function performs no subsequent activation-marker write or notification inspection. Effects inside the gating dependency are unconstrained.

5. **Clearing call order.** With both entry gates passing, a zero request, a zero blocker, and a nonzero gating result, the calls proceed through F-5b4414e97173, F-f22506ed1cf0, F-bdd02c544bdc, and F-59798b8211ac. This function clears `0x98` between the second and third calls. Notifications follow conditionally.

6. **Sequential notifications.** On reaching notification processing, suppose only entries `0x00` and `0x14` are nonzero. If notification dependencies preserve all six entries, exactly two notification calls occur. If the first notification instead clears `0x14`, and no other entries change, only one occurs.

7. **Already active.** With both entry gates passing, a nonzero request and activation marker two cause the indexed byte to become one. No activation or pending-update dependency is called.

8. **All four pending operations occur conditionally.** With both entry gates passing, a nonzero request, and a zero activation marker at its check, activation occurs. If each pending marker remains nonzero until its own check—including through activation and all earlier dependencies—the four pending operations occur in the order `0xA0`, `0x9E`, `0xA2`, `0xA4`. Each marker is cleared after its call returns. Initially nonzero markers alone do not guarantee this outcome.

9. **Later dependency changes are observed.** If the first pending operation is reached and F-b2f40e55a309 clears `0x9E`, the second operation is skipped, provided that marker remains zero until checked. If it changes the associated-object field, later operations read the changed field rather than a guaranteed entry-time value.

10. **Activation-marker overlap.** With both entry gates passing, index `0x80` selects `0x98`. A nonzero request writes one there before the activation check, suppressing activation even if the marker was initially zero.

11. **Gap access through indexing.** With both entry gates passing, index `0x84` selects `0x9C`. The request writes zero or one there despite the absence of a dedicated fixed-offset access to that location.

12. **Distinct indirect reads.** If indirect dispatch is reached and the associated-object field remains unchanged throughout expression evaluation, both reads supply the same value. If mutation intervenes between those reads, equality is not guaranteed; the static evidence does not determine which read occurs first.

## Uncertainties

The following remain documented_only:

- Domain-specific meanings of activation, notifications, payloads, and clearing gates.
- Valid index ranges, object boundaries, and caller guarantees against overlap.
- Original ABI, alignment, byte order, and byte-argument extension.
- Address-overflow and invalid-memory behavior.
- Dependency implementations, error behavior, global mutations, and ownership effects.
- The indirect target’s identity and full contract.
- Evaluation order within call expressions, including target resolution relative to arguments.
- Lifetime guarantees, re-entry rules, and concurrency permissions.
- Fidelity of the supplied representation to the original executable.

These unknowns do not alter the established branch conditions or direct-write order, but they limit claims about final state and external effects.

## Evidence

Established static evidence consists of the supplied function representation and opaque dependency mapping. It supports the represented widths, field offsets, entry-gate order, indexed writes, conditional call sequences, post-call marker writes, sequential notification reads, and separate associated-object reads in the indirect expression.

Semantic role names are inferred intent, not established external API definitions. No implementation access, independent executable validation, runtime execution, or runtime tests have been performed.
