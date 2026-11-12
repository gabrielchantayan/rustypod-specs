<!-- Trusted provenance: F-ca493b2acd41; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ca493b2acd41

## Purpose

F-ca493b2acd41 coordinates an operation on a caller-supplied target using mutable configuration, a requested descriptor resource, and a requested work resource. Its success path includes preliminary operations, endpoint-derived submissions, two required region stages, an optional region stage, and two final descriptor submissions.

This contract describes observable accesses, argument values, ordering constraints, status handling, and cleanup requests. Terms such as “descriptor,” “region,” and “submission” identify roles in this function; they do not establish a file format, storage protocol, or dependency implementation.

The operation is not transactional. Failure prevents later stages but does not reverse earlier calls, submissions, or writes.

## Interface

The function accepts four 32-bit input words and returns a 32-bit status word.

| Input | Contract |
|---|---|
| Target | Used as an address and passed to dependencies. Four-byte fields at offsets `0x10` and `0x14` participate in position calculations. |
| Configuration | Address of mutable configuration. Supplies sizing, region parameters, and an optional-stage condition. |
| Placement | Passed to descriptor preparation and multiplied by a cached scale for a preliminary operation. |
| Additional input | Saved and subsequently copied into storage also used for an endpoint output. No dependency explicitly receives this input as an argument. Its possible indirect effect is unresolved. |

A zero return means every checked operation on the selected path succeeded. A nonzero checked status is returned unchanged after applicable cleanup requests. A zero first word in either resource-request record produces status `25`.

### Addressed storage

Every object-field access listed below is four bytes wide.

| Object | Offset | Observed role |
|---|---:|---|
| Target | `0x10` | Low position word; also used in unsigned borrow comparisons. |
| Target | `0x14` | Companion position word and independent dependency argument. |
| Configuration | `0x18` | Doubled sizing input; later supplies a truncated region parameter. |
| Configuration | `0x20` | Sizing input; later supplies a truncated region parameter. |
| Configuration | `0x138` | Explicitly overwritten with zero. |
| Configuration | `0x13c` | Late condition controlling the optional region stage. |
| Descriptor | `0x0c` | Optional-region position multiplier. |
| Descriptor | `0x28` | Scale source, sizing input, and exact endpoint-path comparison against `512`. |
| Descriptor | `0x2c` | Upper endpoint and threshold input. |
| Descriptor | `0x30` | Lower endpoint. |
| Descriptor | `0xd0` | First-region position multiplier. |
| Descriptor | `0x120` | Second-region position multiplier. |

Two separate local resource records each contain two 32-bit words. F-d9e0ae06e54d receives their addresses. The first word is subsequently tested and used as a resource address; the second word is not explicitly inspected. Initialization of either word is not established independently of that dependency.

Other relevant local storage comprises:

- A shared position slot, exposed to endpoint transformation and later assigned region bases.
- An alternate endpoint-output slot, later overwritten with the additional input.
- Two region-output slots. The first also holds a preliminary span before its address is exposed to region processing.

These are actual shared storage locations, not interchangeable abstract values. Their addresses can escape to dependencies. Resource-address words and object fields are read again at later uses; the function does not establish that dependencies preserve them.

### Arithmetic and validity

The visible operations use 32-bit words. This contract describes their bit-vector results: multiplication, doubling, addition, and subtraction retain the low 32 bits; right shifts used for scaling and position conversion are logical. Sizing and threshold comparisons are unsigned.

Region parameters retain only the low 16 bits of their configuration fields, supplied as nonnegative values in 32-bit argument words.

There is no explicit validation of addresses, object extent, alignment, resource capacity, endpoint consistency, or consumed output initialization. Defined execution requires usable storage and usable values for every subsequent read. Recovery from violations is not established.

## Behavior

### Resource lifetime and failure boundaries

The descriptor request is for `512` bytes. If its record’s first word is zero, only that record is submitted to F-cc6f52873d53, and the result is `25`.

Successful descriptor acquisition is followed by the explicit zero write to configuration offset `0x138`. Descriptor preparation then occurs before any preliminary operation. Its ordinary return value is not checked.

The work resource is requested only after all three preliminary operations succeed. A zero first word in the work record causes cleanup requests for the work record and then the descriptor record, followed by return `25`.

Once both resources have been acquired, any checked failure ends further processing and requests cleanup in work-then-descriptor order. Success uses that same cleanup order. Cleanup results are ignored and do not replace the selected status.

### Preliminary operations and work sizing

After descriptor preparation, the function caches a scale equal to descriptor field `0x28` divided by `512`, using a logical nine-bit right shift. It also obtains a span by multiplying that scale by the wrapped difference between the upper and lower endpoints.

Three F-193bd9926bd2 operations are attempted in order, with each requiring a zero low 32-bit result before the next is attempted:

| Operation | Ordered arguments |
|---|---|
| First | Target; current lower endpoint; zero; zero; cached scale. |
| Second | Target; high 32 bits of the first operation’s result; placement multiplied by cached scale; zero; cached span minus cached scale. |
| Third | Target; current target field `0x14`; target field `0x10` minus cached scale; target field `0x14` decreased by the unsigned borrow condition; cached scale. |

For the third operation, the borrow condition is whether a read of target field `0x10` is less than the cached scale. With stable target fields, the third and fourth arguments form a two-word subtraction by the scale. The separate second argument remains the companion field itself. Independent field reads are not established as an atomic snapshot.

Work sizing uses later field reads, after the preliminary operations. With stable fields during this calculation, the candidate is the unsigned maximum of:

- Twice configuration field `0x18`, with 32-bit wraparound.
- Configuration field `0x20`.
- Descriptor field `0x28`.
- The wrapped endpoint difference logically shifted right by three.

The calculation can reread configuration field `0x20` and descriptor field `0x28` while selecting candidates. Consequently, a single-snapshot maximum is not guaranteed if those reads return changing values.

A candidate already divisible by `512` is unchanged. Otherwise, adding `512` with wraparound and clearing the low nine bits supplies the requested work size. This calculation can produce zero; no separate error or minimum-size correction is applied.

### Endpoint-derived submissions

After work acquisition, F-74cfcbeb5ec0 receives configuration, target, descriptor, and work-buffer addresses. Its status must be zero.

If the current unsigned upper endpoint exceeds `4096`, an unmapped dependency receives the work-buffer address and length `512`. Its effects are unknown. In particular, this contract does not identify it as a clearing operation.

F-b5df80907ff0 then receives the descriptor, the current upper endpoint minus one, the work buffer, and the shared position-slot address. Its ordinary return value is not checked.

The current descriptor field `0x28` selects the submission path:

- **Not `512`:** One endpoint submission uses the auxiliary ABI word following the transformation and the shared position.
- **Exactly `512`:** A second transformation uses the current upper endpoint minus two and the alternate output-slot address. The two position slots are compared after this transformation.

If those positions compare equal, there is one submission. Its second argument is the shared position value, rather than a transformation auxiliary ABI word. Its third argument is also the shared position. Both transformations have already had access to the work buffer, and the first output slot is not assumed unchanged by the second transformation.

If the positions differ, the function performs two fresh submission sequences. Each begins with the unmapped dependency receiving the work buffer and `512`, followed by transformation into the shared position slot. The first uses the then-current upper endpoint minus one; the second uses the then-current upper endpoint minus two. Each submission uses the auxiliary ABI word following its own transformation. Failure of the first submission prevents the second sequence.

Every endpoint submission through F-3d3ba69a50c8 has the following ordered arguments: target; the path-specific second word; shared position; zero; length `512`; work buffer. Every submission status is checked. Region processing begins only after the selected endpoint path succeeds.

### Region processing and output visibility

The first region base is the cached scale multiplied by current descriptor field `0xd0`. That product is assigned to the shared position slot. The low 16 bits of current configuration field `0x20` are captured before the region call.

F-4568711d0858 receives target, the shared base value, configuration, zero, work buffer, and the two region-output addresses. After the call, the second output is copied into a snapshot, followed by the first output. These reads precede the status decision, including on a failing path.

On success, a zero second output skips the helper and continuation. A nonzero second output permits F-d7be9cfb4000, followed by F-fa4d0dd2b364.

The second region uses current descriptor field `0x120` for its scaled base. Before its region call, the additional input is copied into the alternate endpoint-output slot, and the low 16 bits of current configuration field `0x18` are captured.

F-6839106467da receives target, the shared base value, configuration, descriptor, work buffer, and the same two region-output addresses. Its first output is snapshotted before the status decision. Its second output is tested only after a successful status.

The two continuation contracts differ:

| Property | First region | Second region |
|---|---|---|
| Helper inputs | First output and captured parameter. | First output and captured parameter. |
| Continuation eligibility | Second output is nonzero after the region call. | Second output is nonzero at the post-success test. |
| Continuation’s first-output contribution | Snapshot logically shifted right by nine. | Snapshot logically shifted right by nine. |
| Continuation’s second-output argument | Snapshot taken before the helper. | Output slot reread after the helper. |

For either region, continuation arguments are, in order: target; shared position reloaded after the helper plus the shifted first-output snapshot; helper result; the region-specific second-output value; captured parameter; work buffer. Position addition wraps to 32 bits.

Reloading the shared position matters because its address previously escaped. A dependency that retained that address could change the continuation position. Likewise, the second region’s helper could change its second-output slot after the nonzero test; continuation would still occur and would receive the changed value. The first region instead retains its second-output snapshot.

Any nonzero region or continuation status ends processing through cleanup.

After the second region succeeds, including any required continuation, current configuration field `0x13c` controls the optional stage. If nonzero, the shared position is assigned cached scale multiplied by current descriptor field `0x0c`. F-22a34c527492 receives target, that base, configuration, descriptor, and work buffer. Its status must be zero.

### Final descriptor submissions

After all selected regions succeed, F-25ac48bbe608 receives the descriptor. Its ordinary return value is not checked.

The first final submission receives target, the auxiliary ABI word following that dependency, position `2`, zero, length `512`, and descriptor.

Only a zero status permits the second final submission. Its ordered arguments are target; current target field `0x14`; target field `0x10` minus two; target field `0x14` minus one plus the unsigned condition that target field `0x10` exceeds one; length `512`; descriptor.

With stable target fields, the position pair subtracts two with borrow: the companion decreases when the low word is zero or one and is unchanged otherwise. Separate reads need not describe one consistent pair if the fields change.

The final checked status is returned after both cleanup requests.

## Side effects

The only explicit write to caller-supplied object storage is zeroing configuration field `0x138`. It occurs after successful descriptor acquisition and before descriptor preparation. Later failure does not undo it.

Dependencies receive mutable addresses for resource records, configuration, descriptor, work buffer, and local output storage. Their writes, global effects, pointer retention, ownership transfers, and memory-preservation guarantees are not supplied.

The alternate endpoint-output slot has a later explicit write of the additional input. Where its address has previously escaped, that write may be observable through a retained pointer. No such retention or observation is established.

Cleanup means calling F-cc6f52873d53 with a record address. It does not independently prove deallocation, invalidation, or ownership release. Even a failed acquisition record is submitted for cleanup. There is no explicit release of the caller’s target or configuration.

No additional status is synthesized for zero scale, arithmetic wraparound, endpoint underflow, truncated parameters, inconsistent fields, or a zero work-size request.

## Dependencies

Only caller-visible interactions are established.

| Opaque ID | Interaction and result use |
|---|---|
| F-d9e0ae06e54d | Receives a two-word record address and requested byte count. Acquisition is judged from the record’s first word, not an ordinary return status. |
| F-cc6f52873d53 | Receives a resource-record address for cleanup. Result ignored. |
| F-77036d05aebd | Receives configuration; auxiliary ABI word following descriptor acquisition; target fields `0x10` and `0x14`; descriptor; placement. Ordinary result unchecked. |
| F-193bd9926bd2 | Preliminary operation. The first result contributes both a checked low word and a consumed high word; later checked statuses are 32 bits. |
| F-74cfcbeb5ec0 | Checked preparation using configuration, target, descriptor, and work buffer. |
| F-b5df80907ff0 | Receives descriptor, endpoint-derived word, work buffer, and output address. Certain following auxiliary ABI words are consumed. |
| F-3d3ba69a50c8 | Checked submissions with six arguments and explicit length `512`. |
| F-4568711d0858 | First checked region operation; receives two output addresses. |
| F-d7be9cfb4000 | Receives a region output and captured parameter; its 32-bit result becomes a continuation argument. |
| F-fa4d0dd2b364 | Checked continuation using the snapshot/reload distinctions above. |
| F-6839106467da | Second checked region operation; receives the same two output addresses. |
| F-22a34c527492 | Optional checked region operation. |
| F-25ac48bbe608 | Receives descriptor before final submissions. Ordinary result unchecked; a following auxiliary ABI word is consumed. |

One additional dependency receives work buffer and length `512`. No opaque ID is supplied for it. Its identity and effects remain **documented_only**; no replacement ID or semantics are invented.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Stable-field assumptions apply only where stated.

1. **Descriptor acquisition failure:** A zero first descriptor-record word causes that record’s cleanup request and return `25`. Configuration field `0x138` is not explicitly written.

2. **Preliminary failure:** If the first preliminary status is `7`, configuration field `0x138` has already been zeroed. Remaining preliminary operations and work acquisition are skipped. Descriptor cleanup precedes return `7`.

3. **Work acquisition failure:** A zero first work-record word causes work-record cleanup followed by descriptor-record cleanup and return `25`.

4. **Alignment overflow:** If sizing selects `0xffffff01`, the requested work size is zero after wrapped alignment. This does not itself produce a checked error.

5. **Equal endpoint positions:** When descriptor field `0x28` is `512` at its branch read and both output slots compare equal after the second transformation, one endpoint submission occurs. Its second and third arguments use the shared position, and its buffer reflects any effects of both transformations.

6. **Distinct endpoint positions:** Unequal slots cause two fresh endpoint sequences. Failure of the first submission prevents the second sequence and all region processing.

7. **First-region snapshots:** Suppose the region outputs are `1024` and `64`. If the helper changes the escaped second-output slot to `9`, continuation still receives `64`. If it also changes the shared position to `100`, the continuation position is `102`.

8. **Second-region reread:** Suppose the second output is `64` at its eligibility test, then becomes zero during the helper. Continuation still occurs and receives zero for that argument; eligibility is not retested.

9. **Skipped continuation:** A successful region whose second output is zero at its branch test invokes neither the helper nor continuation.

10. **Truncation:** Configuration field `0x20` equal to `0x12345678` when captured supplies first-region parameter `0x5678`, retained across the helper.

11. **Final borrow:** With stable target fields, low word one produces final low position `0xffffffff` and companion minus one. Low word two produces zero and an unchanged companion.

12. **Final submission failure:** Failure of the first final submission skips the second, requests both cleanups, and returns that status without rollback.

## Uncertainties

The following remain **documented_only**:

- Concrete resource formats, field meanings, and higher-level operation intent.
- Allocation, initialization, capacity, ownership, and cleanup guarantees.
- Complete ABI meaning of auxiliary words and the first preliminary operation’s two-word result.
- The unmapped dependency’s identity and memory effects.
- Whether dependencies retain escaped pointers, modify resource records, or preserve globals and other accessible storage.
- Whether the additional input has any downstream effect through its local copy.
- Exact ordering of independent argument-field reads and behavior under changing fields.
- Recovery from invalid storage or uninitialized consumed outputs.
- Higher-level language overflow semantics beyond the described visible 32-bit results.

## Evidence

Established static evidence consists of the supplied function representation and opaque dependency mapping: access widths and offsets, constants, arithmetic, explicit writes, escaped addresses, output snapshots and reloads, conditional calls, checked statuses, and cleanup ordering.

Resource and region terminology expresses observed roles. Allocation semantics, storage-format intent, dependency internals, and preservation guarantees are not established by those roles.

No runtime tests have been performed.
