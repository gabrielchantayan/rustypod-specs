<!-- Trusted provenance: F-4f6c2faba913; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4f6c2faba913

## Purpose

F-4f6c2faba913 acquires an item, obtains an associated state object from its input, and selects processing according to the item’s payload reference.

For a nonzero payload, it conditionally updates state fields, places the acquired-item reference in an eight-byte auxiliary object, submits that object, and performs a subsequent notification-like sequence. For a zero payload, it instead invokes an operation selected through the acquired item’s operation table. Both paths converge on final processing gated by state at offset `0x10c`.

**Established static behavior:** the accesses, comparisons, direct writes, and dependency ordering specified below.

**Inferred intent:** the item may represent a command or event; submission may enqueue work; operations on embedded state may provide synchronization. Those interpretations are not established dependency contracts.

## Interface

### Input and output

The function accepts one address-bearing argument. After acquiring an item, it reads a 32-bit state-object reference from input offset `+0x04`.

The supplied interface has no return value. No success code, recoverable error result, or item-disposition result is explicitly returned.

### Representation

The supplied representation uses four-byte address-bearing values and four-byte integer fields. Offsets are byte offsets.

| Object or storage | Offset | Established use |
|---|---:|---|
| Input object | `+0x04` | State-object reference |
| Acquired item | `+0x00` | Operation-table reference |
| Acquired item | `+0x08` | Payload reference |
| Payload | `+0x04` | Signed 32-bit dispatch value |
| Payload | `+0x0c` | Reference to a four-byte value used by the copy case |
| State object | `+0x10c` | Address passed to an entry operation; later read as a reference |
| State object | `+0x124` | Address passed to auxiliary submission |
| State object | `+0x12c` | Address passed to operations surrounding submission and signaling |
| State object | `+0x134` | Address passed to the signal-like operation |
| State object | `+0x140` | First signed 32-bit control field |
| State object | `+0x144` | Second signed 32-bit control field |
| State object | `+0x148` | Four-byte copied-value field |
| Auxiliary object | `+0x04` | Acquired-item reference |
| Shared comparison storage | — | Four-byte value read separately for equality and, conditionally, signed ordering |

The comparison storage is one shared location, not an established immutable constant. Its two reads may observe different values.

The control fields use signed `-1` as a sentinel in specified branches. This does not establish their complete domains.

### Access preconditions

Normal execution requires valid storage and callable targets for every reached access:

- The input must support the state-reference read.
- The state object must support the reached fields and embedded-object operations.
- The acquired item must support the payload-reference read at `+0x08`. The later item-null check does not protect this earlier dereference.
- A nonzero payload must support the dispatch read at `+0x04`.
- The copy case additionally requires a readable reference at payload offset `+0x0c` and a readable four-byte value at that reference.
- Query results used as objects must support their operation-table reads.
- The auxiliary-object result must support the write at `+0x04`.
- Every reached indirect target must be callable.

These are requirements for the described path, not checks performed by this function. Allocation failure, invalid references, inaccessible storage, and invalid indirect targets have no explicit recovery path.

## Behavior

### Acquisition and entry

The function first calls F-0a77364c41cc with `12`. It passes that result, a descriptor value read from separate global storage, and zero to F-afb630a7107c. The latter result is the acquired-item reference.

Only after these calls does it read the state-object reference from input offset `+0x04`. It then calls F-338d16f9ce72 with the address of state offset `0x10c`.

After that call returns, it reads the acquired item’s payload reference at offset `+0x08`.

This order matters. Acquisition precedes the input-field read, and the entry operation precedes the payload-field read. The supplied source does not establish that these dependencies preserve the input, acquired item, state, or global storage.

The function retains the state reference it read; it does not subsequently reload that reference from the input.

### Zero-payload path

If the payload reference is zero, the function checks whether the acquired-item reference is nonzero. If so, it reads the operation-table reference at item offset zero, obtains the target at table byte offset `+0x04`, and invokes that target with the acquired item as an explicit argument.

The item-null check occurs after the payload dereference. It therefore does not establish safe handling of a null acquisition result.

This path skips:

- Dispatch-value and comparison-storage reads.
- Direct dispatch-dependent state writes.
- Auxiliary allocation and submission.
- The signal-like operation and its surrounding operations.
- The later notification-like construction and delivery.

It proceeds to final processing after the item operation returns. Whether the item operation destroys, releases, retains, or otherwise changes the item is unknown.

### Nonzero-payload dispatch: two comparison reads

For a nonzero payload, the function reads its signed 32-bit dispatch value once.

Selection then uses two distinct reads of the same shared comparison location:

1. **Equality read:** read the comparison storage and compare that observed value with the dispatch value.
2. If equal, write `1` to the first control field at state offset `0x140`. Do not perform the second comparison-storage read.
3. **Ordering read:** only if equality failed, read the comparison storage again and perform a signed less-than comparison between the dispatch value and this second observed value.
4. If less, enter the lower group. Otherwise, enter the non-lower group.

The non-lower group therefore means **greater than or equal to the second observed comparison value**, after inequality with the first observed value. It is not unconditionally an “above the pivot” group.

If both reads observe the same value, a single-pivot interpretation is valid: equality takes precedence, the lower group requires a smaller dispatch value, and the non-lower group requires a larger one. If the observations differ, that interpretation is insufficient. In particular, a dispatch value equal to the second observation can enter the non-lower group.

There is no explicit dependency call between the two comparison reads. Nevertheless, the supplied source does not establish immutable comparison storage or guarantee identical observations.

### Lower group

This group is reached only when equality with the first observation failed and the dispatch value is signed-less-than the second observation.

| Dispatch value | Direct state behavior |
|---|---|
| `0x6600000d` | Write zero to the second control field, then zero to the first |
| `0x66000014` | Initialize the second control field if sentinel-valued, then increment it |
| `0x66000015` | Initialize the second control field if sentinel-valued, then conditionally decrement it |
| `0x66000017` | Write zero to the second control field, then zero to the first |

Other values in this group cause no direct dispatch-field write.

For either initialization case, the function first reads the second control field at `+0x144`. If it equals signed `-1`, it:

- Calls F-87be4aa684d1.
- Reads the returned object’s operation-table reference.
- Invokes the target at table byte offset `0x16c`.
- Writes that operation’s four-byte result to the second control field.

The indirect invocation has no explicit arguments in the supplied representation. An implicit receiver or other ABI inputs are not established.

For `0x66000014`, the function then reads the second control field and writes its value plus one. Thus a query result is stored first and subsequently incremented; it is not itself the final field value.

For `0x66000015`, the function reads the second control field for a signed-positive test. Only if that test succeeds does it read the field again, subtract one, and write the result. The positivity test and subtraction use separate field reads. With stable storage, zero and negative values receive no decrement write. A preceding initialization write remains observable even when no decrement follows.

### Non-lower group

This group is reached only when equality with the first comparison observation failed and the dispatch value is not signed-less-than the second observation.

| Dispatch value | Direct state behavior |
|---|---|
| `0x6600001a` | Write zero to the first control field |
| `0x6600001b` | Conditionally toggle the first control field between `1` and `2` |
| `0x66000024` | Copy a referenced four-byte value into state offset `0x148` |

Other values in this group cause no direct dispatch-field write.

For `0x6600001b`, the function reads the first control field at `+0x140`. If it is not `-1`, that read value becomes the decision value. If it is `-1`, the function calls F-87be4aa684d1, reads the returned object’s operation-table reference, and invokes its target at byte offset `0x18c`. The returned value becomes the decision value.

A decision value of `1` causes a write of `2`; a decision value of `2` causes a write of `1`. Any other value causes no direct write.

The query result is not independently stored into the first control field. Consequently, if the field was `-1` and the query returns neither `1` nor `2`, this function leaves the sentinel unchanged, subject to dependency effects.

For `0x66000024`, the function reads the reference at payload offset `+0x0c`, reads four bytes through that reference, and writes those four bytes to state offset `0x148`. It copies the referenced value, not the reference.

### Common nonzero-payload sequence

Every nonzero-payload case reaches this sequence after dispatch, including unrecognized values:

1. Call F-338d16f9ce72 with the address of state offset `0x12c`.
2. Call F-8157a180d901 with `8`.
3. Write the acquired-item reference at offset `+0x04` of the returned auxiliary object.
4. Call F-37efbe955e3e with the address of state offset `0x124` and the auxiliary-object reference.
5. Call F-acda426cbea1 with the address of state offset `0x134`.
6. Call F-1333331b2da1 with the address of state offset `0x12c`.
7. Call F-0a77364c41cc with `12`.
8. Call F-afb630a7107c with that result, a descriptor value read from another global location, zero, and zero.
9. Call F-9a5c97dc580d with the newly constructed result, the retained state reference, and `1`.

Only the auxiliary object’s second four-byte slot is directly initialized here. Its first slot is neither directly read nor directly written by this function.

The submission result is not tested. There is no direct acquired-item release on this path. Neither fact establishes ownership transfer.

### Final processing

After either payload path, the function reads a reference from state offset `0x10c`. If nonzero, it reads the first four-byte word of the referenced object.

If the reference is zero, or its first word is zero, the function returns without the final indirect operation.

Otherwise, it invokes a target obtained from separate global target storage. The referenced object’s first word is only a nonzero gate; it is not used to select this target.

No explicit arguments appear for this final operation. Its argument contract, ownership effects, and ordinary-call versus tail-transfer behavior remain unresolved.

## Side effects

Direct writes are limited to:

- Dispatch-dependent writes at state offsets `0x140`, `0x144`, and `0x148`.
- The acquired-item reference written at auxiliary-object offset `+0x04`.

Dependencies may perform additional writes, release memory, retain references, modify globals, or reenter processing. No preservation or ownership guarantees are supplied.

The final state-offset-`0x10c` reference is read after preceding dependency calls. It need not equal any earlier contents of that field.

The increment is a 32-bit integer addition without an explicit upper-bound check. The source does not independently establish overflow semantics; maximum-signed-value behavior is **documented_only**.

With stable field contents, the positive-only decrement avoids signed underflow. Without stability between its separate test and arithmetic reads, that conclusion is not established.

No explicit rollback, allocation-result check, submission-failure branch, or exception-handling behavior is shown.

## Dependencies

| Opaque function ID | Established use |
|---|---|
| F-0a77364c41cc | Called with `12` before each construction-like operation |
| F-afb630a7107c | Produces the acquired item and later another object, using separate descriptor storage and different explicit argument counts |
| F-338d16f9ce72 | Receives state addresses `+0x10c` and, on the nonzero-payload path, `+0x12c` |
| F-8157a180d901 | Called with `8`; its result receives the item reference |
| F-37efbe955e3e | Receives state address `+0x124` and the auxiliary object |
| F-acda426cbea1 | Receives state address `+0x134` after submission |
| F-1333331b2da1 | Receives state address `+0x12c` after the signal-like operation |
| F-87be4aa684d1 | Supplies objects for sentinel-dependent indirect queries |
| F-9a5c97dc580d | Receives the second constructed object, retained state reference, and `1` |

Additional indirect dependencies are identified by selection mechanism: item-table offset `+0x04`, query-table offsets `0x16c` and `0x18c`, and the final global target storage. No opaque function IDs for those targets are supplied.

Although the dependency mapping includes F-599da3979baf, the body does not establish an explicit call to it. A control-flow destination bearing a mapped name is not sufficient call evidence.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Unless otherwise stated, references are valid, dependencies return normally, and dependency or external effects do not alter the fields under discussion.

1. **Zero payload.** A valid item with a zero payload reference has its table-offset-`+0x04` operation invoked before final processing. No dispatch comparison read, auxiliary submission, or later notification-like sequence occurs.

2. **Equality short-circuits the second read.** If the dispatch value equals the first comparison observation, the function writes `1` to the first control field and does not perform the ordering read. This takes precedence even when the dispatch value matches a listed numeric case.

3. **Stable-comparison reset.** If both comparison reads observe the same value greater than `0x6600000d`, dispatch `0x6600000d` writes zero to the second control field before writing zero to the first. Common submission follows.

4. **Different observations, equality with the second.** For dispatch `0x6600001a`, suppose the first comparison observation is `0x66000019` and the second is `0x6600001a`. Equality first fails; signed less-than then fails because the values are equal. The non-lower group writes zero to the first control field.

5. **Different observations, lower-group selection.** For dispatch `0x66000014`, suppose the first observation is `0x66000013` and the second is `0x66000015`. The lower group is selected despite the dispatch exceeding the first observation. With a second control field of `-1` and a query result of `7`, the direct field stores are `7`, then `8`.

6. **Stable-comparison decrement boundary.** If both comparison observations exceed `0x66000015`, that dispatch value enters the lower group. A stable second control field of zero receives no decrement write; a stable value of `3` becomes `2`.

7. **Stable-comparison toggle.** If both comparison observations are less than `0x6600001b`, that dispatch value enters the non-lower group. A first control field of `1` becomes `2`. An initial sentinel with a query result of `2` becomes `1`; a query result of zero causes no direct field write.

8. **Referenced copy.** For dispatch `0x66000024`, provided the first observation differs from it and the second observation is no greater than it under signed comparison, the referenced four-byte value is stored at state offset `0x148` before submission.

9. **Unrecognized dispatch.** A dispatch unequal to the first observation and unrecognized in its selected group causes no direct dispatch-field write but still performs the full common sequence.

10. **Final gate.** A zero final reference at state offset `0x10c` skips the referenced-word read and final target invocation. A nonzero reference with a zero first word also skips invocation. With both gates nonzero, the separately selected final target is invoked.

## Uncertainties

The following remain **documented_only**:

- Values observed from comparison storage, whether they remain stable, and unconditional reachability of numeric cases.
- Descriptor contents, meaning, lifetime, and mutability.
- Whether size-taking dependencies allocate, initialize, or return null.
- Ownership and lifetime of acquired, auxiliary, query, and notification-like objects.
- Synchronization, memory-ordering, reentrancy, and concurrent-mutation guarantees.
- Implicit arguments and ABI requirements for indirect operations.
- Final target identity and transfer semantics.
- Unchecked arithmetic overflow behavior.
- Dependency failures and their effects on subsequent execution.

A single-pivot reachability model is conditional on both comparison reads observing the same value; it is not an unconditional property of this function.

## Evidence

This contract derives from static inspection of the supplied function body and opaque dependency mapping. Field widths, offsets, explicit arguments, separate comparison reads, branch conditions, direct stores, and call ordering are supported by that material.

Allocation, queueing, signaling, locking, notification, and destruction are interpretations only where qualified. Dependency implementations and runtime observations were not supplied. No runtime tests have been performed.
