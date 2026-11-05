<!-- Trusted provenance: F-ca1c4e2f14bb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ca1c4e2f14bb

## Purpose

F-ca1c4e2f14bb selects a slot-indexed record and associated object, inspects the record after an entry operation, and conditionally proceeds to a request sequence. The request sequence has one mandatory attempt and at most one retry. A nonzero original third input enables a final operation after the requests.

The established contract concerns input widths, address calculations, reads, local writes, dependency calls, and branch conditions. The resource meanings of the records, associated objects, request values, and final operation are not established.

**Inferred intent:** the function appears to prepare an inactive slot before issuing a request. This interpretation does not establish that the record operation clears memory, that the surrounding operations provide synchronization, or that the final operation releases a resource.

## Interface

### Inputs and result

| Input | Representation | Established use |
|---|---|---|
| First input | Opaque 32-bit value | Passed unchanged to every invocation of F-704f99fd5e22. |
| Slot index | Signed 32-bit integer | Compared with 2, used in address calculations, and passed to downstream operations. |
| Third input | Signed 32-bit integer | Initially copied into a local slot; its original value later controls the final operation. |
| Fourth input | Opaque 32-bit value | Initially copied into another local slot, then overwritten before that slot is passed to the request dependency. |

No return value is exposed. The visible ordinary path ends with a return. A nonreturning annotation also appears in the supplied material, so unconditional termination or unconditional nonreturn cannot be established from that annotation alone.

### Local storage

The function uses two distinct addressable 32-bit slots:

- An integer slot, initially containing the third input.
- An opaque-value slot, initially containing the fourth input.

F-704f99fd5e22 receives the opaque-value slot’s address before the integer slot’s address. The same respective addresses are reused on retry. No aggregate layout, adjacency requirement, or ownership transfer is established.

The initial values of both slots are overwritten before their addresses are first exposed to F-704f99fd5e22. In particular, the original fourth input is not the value supplied through the opaque-value slot on either request attempt.

### Global inputs and layout

The visible ordinary path reads:

1. A record-base global.
2. An associated-object-base global.
3. A 32-bit seed global into a retained value, after the exit operation.
4. The same seed global again into the opaque-value slot, after writing 20 into the integer slot.

The seed accesses are **two separate reads**. The first supplies the retained retry value; the second supplies the first request’s slot value. Their results are not guaranteed equal without an established stability assumption.

Records have a calculated stride of 36 bytes. Associated objects have a calculated stride of 28 bytes. Only the first 32-bit word of the selected record is directly inspected. These calculations do not establish actual allocation sizes, array bounds, or the internal layout of the associated objects.

### Preconditions and validity boundaries

Every address actually accessed or passed to a dependency must be valid for that operation. The selected record must support the initial 32-bit read.

The index comparison is only an upper-bound test:

- Indices 0, 1, and 2 skip the guard dependency.
- Negative indices also skip it.
- Indices greater than 2 invoke the guard dependency, but no explicit return follows that call.

The supplied representation uses 32-bit integer/address values. Defined overflow behavior, pointer wrapping, alignment requirements, and valid storage for arbitrary indices are not established. Neither negative indexing nor continuation after an excessive index is guaranteed safe.

## Behavior

### Initialization and guard

The function first stores the third input into the integer slot, then the fourth input into the opaque-value slot.

It compares the signed slot index with 2. If the index is greater than 2, it calls F-a817d8d58c10 without explicit arguments.

If that dependency returns, processing continues. The visible function does not independently reject the index, substitute another index, or return an error. Because the guard precedes the global-base reads, changes it makes to those globals can affect subsequent selection.

### Selection and entry

After the guard, if any, the function calculates the record address using the then-read record base and an index displacement of 36 bytes per slot. It next calculates the associated-object address using the then-read object base and a displacement of 28 bytes per slot.

The addresses are retained. Later changes to the base globals do not cause visible recalculation.

The function invokes the entry dependency, identified here as F-U1, with the associated-object address. Only after F-U1 returns does it read the first 32-bit word of the selected record.

The branch depends solely on whether that word is zero. No status enumeration, pointer meaning, or signed interpretation is needed for this comparison. The function does not read the word before F-U1 and does not visibly re-read it after deciding the branch.

### Nonzero record word

If the first word is nonzero, the function calls:

1. F-ae524095c0b0 without explicit arguments.
2. F-feee9156a257 without explicit arguments, if the preceding dependency returns.

F-feee9156a257 is annotated as nonreturning. If that annotation holds, no record operation, exit operation, request, or optional final operation follows on this branch.

If F-ae524095c0b0 does not return, F-feee9156a257 is not reached. If F-feee9156a257 unexpectedly returns, the visible body falls through to the ordinary record-handling sequence; no separate recovery path is shown.

No matching exit operation is visible before these calls. Whether either dependency terminates execution, unwinds state, releases anything, or changes the selected record is unknown.

### Record operation and exit

When the inspected word is zero—or if the preceding branch unexpectedly falls through—the function calls F-U2 with:

1. The selected record address.
2. The value 36.

It then calls F-U3 with the retained associated-object address.

The value 36 matches the record stride. This supports an inference that it describes an extent associated with the record, but does not establish that F-U2 clears, copies, initializes, or writes exactly 36 bytes.

F-U1 and F-U3 resemble paired entry and exit operations. Synchronization semantics are not supplied. No result from F-U2 or F-U3 is inspected.

### Two seed reads and the first request

After F-U3 returns, the following order is established:

1. Read the 32-bit seed global and retain that value separately.
2. Write decimal 20 into the integer slot.
3. Read the seed global again and write this second result into the opaque-value slot.
4. Invoke F-704f99fd5e22.

The request arguments, in order, are:

1. The original first input.
2. The slot index.
3. The opaque-value slot’s address.
4. The integer slot’s address.
5. Zero.

Thus, immediately before the first request begins, the integer slot contains 20 and the opaque-value slot contains the **second** seed-read result. The separately retained **first** seed-read result is reserved for possible retry.

There is no intervening dependency call between these two seed reads. Nevertheless, the supplied body establishes two reads, not a guaranteed stable snapshot. It does not establish the conditions under which the seed may change or the reads must agree.

The first request returns a 32-bit integer, tested only for zero versus nonzero. The meanings of 20, the seed, and individual return codes are unknown. A zero result selects the no-retry path; it is not independently established as semantic success.

### Conditional retry

If the first request returns zero, there is no retry.

If it returns nonzero, the function:

1. Writes decimal 23 into the integer slot.
2. Writes the retained result of the **first** seed read into the opaque-value slot.
3. Calls F-704f99fd5e22 again with the same argument arrangement and the same respective local-slot addresses.

The retry does not read the seed global again. It discards any changes the first request made to either local slot.

Consequently:

- At most two request calls occur.
- The first request and retry may receive different seed values.
- Equality follows only if the two earlier seed reads produced equal values.
- Changes to the seed global during the first request do not select the retry value.
- The retry’s return value is not tested.
- No third request occurs.

After the request sequence, the function does not otherwise interpret the contents of either local slot.

### Optional final operation

The final branch tests the **original third input**, not the integer slot’s current contents.

If the original third input is zero, the provider and final operation are skipped.

If it is nonzero, including a negative value, the function calls F-U4 without explicit arguments. It then calls F-2250ce450b3d with:

1. F-U4’s returned 32-bit value.
2. The slot index.
3. Zero.

This branch does not depend on either request result. It can follow a zero first result or a retry with any result, provided all intervening dependencies return.

No result from F-2250ce450b3d is inspected. The ordinary path then returns without exposing a value.

## Side effects

### Directly established effects

The function visibly performs:

- Initial writes and later overwrites of two local 32-bit slots.
- Separate reads of the record and associated-object base globals.
- One 32-bit read from the selected record.
- Two ordered reads of the seed global on the ordinary request path.
- Dependency calls in the order described above.

No direct global write, allocation, deallocation, or returned resource is visible.

### Dependency-mediated effects and ownership

Dependency implementations are not supplied. Their memory effects, global changes, external effects, and termination behavior must not be inferred merely from call position.

In particular:

- F-U2’s effect on the record is unknown.
- F-704f99fd5e22 receives addresses of mutable local storage, but its actual reads, writes, or retention behavior are unknown.
- No dependency is assumed to preserve globals.
- No dependency is assumed to acquire, transfer, or release ownership.
- Pairing F-U1 with F-U3 does not establish a lock contract.
- No rollback, local error translation, or compensating action is visible.

The local-slot addresses refer to invocation-local storage. The source establishes their availability during the calls, but supplies no safe post-return retention protocol.

## Dependencies

F-U1 through F-U4 are document-local opaque identifiers for dependencies whose published opaque IDs were not supplied. They do not assert additional identities or implementations.

| Dependency | Established interaction |
|---|---|
| F-a817d8d58c10 | No-argument call when the signed index exceeds 2; continuation remains visible if it returns. |
| F-ae524095c0b0 | No-argument call when the inspected record word is nonzero. |
| F-feee9156a257 | Follows a returning F-ae524095c0b0; annotated as nonreturning. |
| F-U1 | Receives the selected associated-object address before the record read. |
| F-U2 | Receives the selected record address and 36 before F-U3. |
| F-U3 | Receives the retained associated-object address before both seed reads. |
| F-704f99fd5e22 | Receives the first input, index, two local-slot addresses, and zero; called once with one conditional retry. |
| F-U4 | No-argument provider called only when the original third input is nonzero; supplies a 32-bit value. |
| F-2250ce450b3d | Receives F-U4’s result, the index, and zero; its result is ignored. |

## Acceptance examples

These are conditional criteria derived from static evidence. They are not executed tests.

### Ordinary path without retry

Given valid addresses, returning dependencies, a zero inspected record word, a zero first request result, and an original third input of zero:

- F-U1 precedes the record read.
- F-U2 receives the record address and 36, then F-U3 receives the associated-object address.
- After F-U3, the first seed read precedes the integer-slot write of 20.
- The second seed read follows that write and supplies the first request’s opaque-value slot.
- Exactly one request occurs.
- Neither F-U4 nor F-2250ce450b3d is called.

### Unequal seed reads followed by retry

Suppose the first seed read yields 100 and the second yields 200. Suppose the first request returns nonzero.

- The first request begins with opaque-value slot 200 and integer slot 20.
- The retry begins with opaque-value slot 100 and integer slot 23.
- The same respective slot addresses are passed both times.
- No additional seed read occurs for the retry.

This example is conditional on the two reads producing different values; it does not establish a mechanism by which the global changes.

### Request mutation does not choose the retry seed

Suppose the two seed reads yield 100 and 200. During the first request, the dependency changes both local slots and changes the seed global to 300, then returns nonzero.

The retry still receives 100 through the opaque-value slot and 23 through the integer slot. Neither the first request’s slot changes, the second seed-read value of 200, nor the later global value of 300 selects the retry seed.

### Stable seed values

If both seed reads yield 100, both requests receive 100 through the opaque-value slot, if a retry occurs. This equality is conditional on the read results, not an unconditional property of the function.

### Final operation uses the original third input

With an original third input of −1 and returning dependencies, F-U4 and F-2250ce450b3d are called after the request sequence, even if the retry returns nonzero.

With an original third input of zero, changes to the local integer slot do not enable that final operation.

### Record branch and index boundaries

If F-U1 changes the selected first word before returning, the branch uses the subsequently read value.

If that value is nonzero, F-ae524095c0b0 is called, followed by F-feee9156a257 if the first dependency returns. If the nonreturning annotation holds, no exit operation or request follows.

Index 2 skips F-a817d8d58c10. Index 3 calls it before either base read; further processing is conditional on return and valid subsequent operations. Index −1 also skips the guard, without establishing valid negative indexing.

## Uncertainties

The following remain **documented_only**:

- Semantic domains of the opaque inputs, seed, records, and associated objects.
- Allocation bounds, alignment, valid index range, and overflow or address-wrap behavior.
- Meaning and mutability of record storage beyond the inspected first word.
- Actual synchronization behavior of F-U1 and F-U3.
- Memory effects of F-U2.
- Meaning of request values 20 and 23 and return codes beyond their branch effects.
- Stability, concurrency, and access semantics of the seed global; equality of its two reads is not established.
- Pointer retention, ownership, and resource effects of dependencies.
- Meaning of F-U4’s result and F-2250ce450b3d.
- Exact termination behavior where annotations and visible continuation differ.

## Evidence

Established claims derive from the supplied static body: represented widths, constants, address strides, ordered reads and writes, argument order, branches, and local-slot reuse.

In particular, the body distinguishes a retained first seed read from a second seed read into the first request’s opaque-value slot. The retry restores the retained first value rather than reloading the global.

Nonreturning claims derive from annotations, not observed execution. Synchronization, clearing, preparation, and resource-management interpretations remain inferred intent or unknown as identified above.

No dependency implementations were supplied. No runtime tests have been performed.
