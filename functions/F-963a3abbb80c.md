<!-- Trusted provenance: F-963a3abbb80c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-963a3abbb80c

## Purpose

F-963a3abbb80c coordinates an operation between source and destination entries identified by container-like identifiers and name-like values. It resolves endpoints, retrieves a source record, conditionally restricts destination ancestry, attempts destination installation, optionally updates an associated record, and removes the source.

**Inferred intent:** moving or renaming an entry within a structured namespace. “Container,” “installation,” “removal,” and “associated record” describe apparent roles, not established dependency semantics.

The operation is not established as atomic. Failures can occur after potentially state-changing dependency calls. Selected failures trigger a destination-removal attempt; source restoration and associated-record restoration are not established.

## Interface

The function accepts seven arguments and returns a signed 32-bit status.

| Argument | Established use |
|---|---|
| Context pointer | Direct field access and dependency argument. |
| Source container | Opaque 32-bit value supplied to source resolution and fallback lookup. |
| Source name | Opaque 32-bit value supplied to resolution, length, comparison, and fallback dependencies. |
| Source selector | Opaque 32-bit value copied into local writable storage; lookup may replace it. |
| Destination container | Unsigned 32-bit identifier, potentially replaced through normalization. |
| Destination name | Opaque 32-bit value supplied to length, comparison, and resolution dependencies. |
| Result pointer | Writable 32-bit storage supplied to installation and reconciliation dependencies; explicitly assigned on the unchanged-endpoint path. |

Zero denotes successful completion on explicit success paths. Other values are propagated or selected as described below. A nonzero result does not imply that no state changed.

There are no explicit null, alignment, extent, lifetime, or alias checks. Valid storage for direct accesses is a caller precondition. Dependency-specific requirements are **documented_only**.

### Directly accessed layouts

All offsets are in bytes.

| Object | Offset | Access |
|---|---:|---|
| Context | 0 | Read/write unsigned 16-bit state word. |
| Context | 2 | Read unsigned 16-bit format discriminator. |
| Context | 8 | Write 32-bit final value. |
| Context | 344 | Read 32-bit subordinate handle at relevant call sites. |
| Source/comparison record | 0 | Read signed 16-bit kind. |
| Source record | 2 | Read flag byte. |
| Source/comparison record | 8 | Read or write 32-bit value. |
| Source/comparison record | 20 | Read 32-bit value. |
| Source record | 80 | Write 32-bit resolution auxiliary value. |
| Associated record | 4 | Write 32-bit destination identifier in one format. |
| Associated record | 8 | Read signed 16-bit quantity in that format. |

Endpoint descriptor storage spans 520 bytes in the supplied representation. Dependencies receive addresses at descriptor offsets 2 and 6, source/comparison-record offset 6, and associated-record offsets 8, 10, and 14. Address arguments alone do not establish access width, encoding, or transfer extent.

### Format and arithmetic

Before the first dependency invocation, context offset 2 is compared with `0x482b`. Equality selects the **distinguished format**; every other value selects the **alternative format**. This decision is cached and is not recomputed after dependency calls.

Name-length results are narrowed to signed 16-bit values. Equality comparison uses those narrowed values, and the name-comparison dependency receives the sign-extended source length. Negative lengths are not rejected.

Identifiers used for ancestry traversal are unsigned 32-bit values. The traversal termination comparison against `3` is unsigned.

For distinguished-format associated-record reinstallation, the size-like value is twice a signed 16-bit quantity plus ten, narrowed to signed 16 bits. The intermediate mathematical range is −65526 through 65544, which fits signed 32 bits but not always signed 16 bits. Out-of-range narrowing behavior is **documented_only**. The resulting size-like argument is passed as a signed value; the reconciliation retry instead uses an unsigned 16-bit value.

## Behavior

### Endpoint resolution and name classification

F-a0c5d57d1d48 receives the context, source container, source name, zero, source-descriptor storage, and auxiliary-output storage. A nonzero status returns immediately.

F-1cb585febd84 is then invoked for the source name and destination name, in that order. Different signed 16-bit results classify the names as different without invoking comparison. Equal results cause F-7f14f1ab6851 to receive both names and the sign-extended source length. Only a zero comparison result classifies the names as equal.

For equal names, F-U01 receives destination-descriptor storage, source-descriptor storage, and `0x208`. F-946cac4b59bc then receives the requested destination identifier and destination-descriptor offset 2.

F-U01 is an opaque ID assigned here to the otherwise unmapped dependency. Its arguments suggest a 520-byte copy, but neither copying nor its extent is established.

For different names, F-a0c5d57d1d48 resolves the destination using the context, destination container, destination name, zero, destination-descriptor storage, and the same auxiliary-output storage used by source resolution. A nonzero status returns immediately.

### Source lookup and unchanged-endpoint return

F-ae00f06fa2b4 receives the context, current selector, source descriptor, source-record storage, and writable selector storage.

If it returns `0x30`, the alternative format returns `0x30` immediately. The distinguished format instead invokes F-f6310772ddb8 with the context, original source container and name, zero, source descriptor, source record, and selector output. Any remaining nonzero lookup status returns directly.

F-4055be9c3f18 receives source-descriptor offset 2. Its 32-bit result is retained as the source identifier.

If this identifier equals the **requested**, not normalized, destination identifier and the names were classified equal, the current selector is written directly to the caller’s result storage and the function returns zero. Preparation, installation, associated-record processing, source removal, and final context updates are bypassed.

### Destination normalization and ancestry restriction

When the retained source identifier differs from the requested destination identifier, F-9a0ed9935fa9 receives the context, requested destination identifier, two zeros, and writable descriptor, record, and auxiliary outputs. Errors return directly.

Normalization initially reads the returned record’s 32-bit value at offset 8:

- Kind `1` retains that value as the destination identifier.
- Kind `0x100` replaces it with F-4055be9c3f18’s result for record offset 6.
- Any other kind returns `-122`.

The unchanged-endpoint fast path is not revisited, even if normalization makes the destination identifier equal to the retained source identifier.

F-4055be9c3f18 then receives the returned descriptor at offset 2. Its result is retained for later bookkeeping, together with an address at descriptor offset 6 and the normalization auxiliary output.

For source kind `1` or `0x100`, ancestry checking still occurs, including when normalization produced identifier equality:

- The candidate initially comes from source-record offset 8. In the alternative format, F-4055be9c3f18 applied to source-record offset 6 replaces it.
- F-4055be9c3f18 receives the destination-container descriptor at offset 2 to obtain the initial traversal identifier.
- Candidate `2`, or a candidate equal to the normalized destination identifier, causes `-122`.
- Otherwise, equality between candidate and traversal identifier causes `-122`.
- If unequal and the unsigned traversal identifier is below `3`, checking succeeds.
- Otherwise, F-9a0ed9935fa9 resolves the traversal identifier with two zeros and writable outputs. Errors propagate. F-4055be9c3f18 applied to the returned descriptor at offset 2 supplies the next traversal identifier.

Thus equality rejection precedes below-`3` termination. No cycle detection or iteration bound is visible.

Only after normalization and any applicable ancestry checking succeed does F-56433c4250f4 receive the normalized destination identifier. F-56433c4250f4 then receives the retained source identifier. Both calls include the context, and neither result is tested. Both occur even if normalization made their identifiers equal.

If normalization was unnecessary, only the source preparation call occurs.

### Destination installation and reconciliation

For different names in the distinguished format, the latest endpoint-resolution auxiliary value is written to source-record offset 80, followed by F-7103344f7722 with the context.

F-e320c3146c6e receives the source-record address. Its result supplies the fourth argument of F-6c1bc8c36c27, alongside the current context subordinate handle, destination descriptor, source record, and caller result pointer.

A status other than zero or `0x21` returns immediately.

Status `0x21` permits reconciliation only when retained source and current destination identifiers match and the names differ. Otherwise, the function returns `0x31`.

F-f26976f63516 receives the current subordinate handle, destination descriptor, current selector, descriptor and comparison-record outputs, unsigned 16-bit output storage, and caller result pointer. Its status `0x20` maps to `0x30`; other nonzero statuses propagate.

Reconciliation requires equal record kinds and matching identities:

| Kind | Identity comparison |
|---|---|
| `1` or `2` | Direct 32-bit values at offset 8. |
| `0x100` | F-4055be9c3f18 results for source-record offset 6, then comparison-record offset 6. |
| `0x200` | Direct 32-bit values at offset 20. |
| Other | Reject with `0x31`. |

A mismatch returns `0x31`.

For kinds outside `1`, `2`, and `0x100`, comparison-record offset 20 is assigned to comparison-record offset 8 before checking whether the kind is `0x200`. Unsupported kinds therefore undergo that local record write before rejection.

After a match, F-bd138ceb6421 receives the current subordinate handle and source descriptor. Failure returns immediately. Success records that source removal has already occurred.

F-6c1bc8c36c27 is then retried for the destination and source record, using the unsigned 16-bit reconciliation output as its fourth argument. Any nonzero retry status returns directly. This failure does not reach destination cleanup or visibly restore the removed source.

### Associated-record selection and update

Following successful installation, the current source kind selects associated-record handling:

- Kinds `1` and `2` use source-record offset 8.
- Kind `0x100` invokes F-4055be9c3f18 for source-record offset 6 and writes its result to source-record offset 8 before checking for zero.
- Kind `0x200` copies source-record offset 20 to offset 8 before testing flag bit `0x02`. A clear bit skips associated processing despite the write.
- Other kinds skip associated processing.

For eligible kinds, zero at source-record offset 8 skips associated lookup. These writes affect record storage already exposed to dependencies.

For a nonzero selected identifier, F-eef02e922a52 receives the context, identifier, two zeros, and writable associated descriptor, record, and auxiliary outputs. Failure enters destination cleanup.

In the alternative format:

- Different source and current destination identifiers cause F-946cac4b59bc to receive the destination identifier and associated-record offset 10.
- Different names cause F-f6d6f6d14101 to receive destination-descriptor offset 6 and associated-record offset 14.
- The size-like value is `0x2e`.

In the distinguished format:

- Different identifiers cause a direct 32-bit destination-identifier write at associated-record offset 4.
- Different names cause F-f6d6f6d14101 to receive destination-descriptor offset 6 and associated-record offset 8.
- The signed 16-bit quantity at associated-record offset 8 is then read to calculate the size-like value described above.

That quantity read follows the name-helper call when present; helper-produced contents are not guaranteed.

F-bd138ceb6421 receives the current subordinate handle and associated descriptor. Only on success does F-6c1bc8c36c27 receive the current handle, associated descriptor, associated record, signed size-like value, and local output storage. Either failure enters destination cleanup.

### Source removal, bookkeeping, and finalization

Unless successful reconciliation already removed the source, F-bd138ceb6421 receives the current subordinate handle and source descriptor. Failure enters destination cleanup.

When retained source and current destination identifiers differ, F-5ff5fde1145e is invoked twice, in order:

1. Context, retained source identifier, zero, sign-extended current source kind, zero, `0xffffffff`.
2. Context, retained destination parent-like value, retained descriptor-offset-6 address, sign-extended current source kind, normalization auxiliary value, `1`.

The first result is ignored. The second becomes the return status. The source kind is read separately for each call; preservation across the first invocation is not assumed.

Without this bookkeeping branch, the selected return status is zero.

Finalization occurs even if the second bookkeeping status is nonzero:

1. Read context offset 0, OR its 16-bit value with `0xff00`, and write it back.
2. Invoke F-49743e74136c without arguments.
3. Write its 32-bit result to context offset 8.
4. Invoke F-92a758c7556d with the context.
5. Return the selected status.

The last helper’s effects on previously written fields remain unknown.

### Destination cleanup

Associated lookup, associated removal, associated reinstallation, and final source-removal failures cause one F-bd138ceb6421 invocation with the current subordinate handle and destination descriptor.

Its status is ignored; the triggering error is returned. No explicit associated-record restoration follows failed reinstallation. Other early failures do not automatically enter this cleanup path.

## Side effects

Established direct writes include writable selector storage, the fast-path caller result, source-record offsets 8 and 80 on specified branches, comparison-record offset 8, distinguished-format associated-record offset 4, and final context fields.

The caller result is not initialized at entry. Installation and reconciliation receive its address before overall success is known. An error does not establish unchanged, valid, or rolled-back output.

Descriptors and records use invocation-local storage. No explicit allocation or release is visible. Passing an address does not establish copying, retention permission, ownership transfer, or release responsibility.

The format choice and selected scalar identifiers are cached. The subordinate handle and source kind are read again at later call sites. No dependency is assumed to preserve context fields, globals, or previously exposed memory. Accordingly, descriptions of later processing are conditional on the values present when those accesses occur.

## Dependencies

These roles establish invocation behavior, not complete callee contracts.

| Opaque ID | Observed role |
|---|---|
| F-a0c5d57d1d48 | Endpoint resolution; descriptor and auxiliary outputs; status checked. |
| F-1cb585febd84 | Name-length-like result narrowed to signed 16 bits. |
| F-7f14f1ab6851 | Name comparison; zero selects equality. |
| F-U01 | Two descriptor addresses and scalar `0x208`; effects unresolved. |
| F-946cac4b59bc | Identifier and writable address at specified offset. |
| F-ae00f06fa2b4 | Primary source lookup with writable selector. |
| F-f6310772ddb8 | Distinguished-format fallback lookup. |
| F-4055be9c3f18 | Address-based extraction of a 32-bit scalar. |
| F-9a0ed9935fa9 | Identifier resolution for normalization and ancestry. |
| F-56433c4250f4 | Context and identifier preparation; result ignored. |
| F-7103344f7722 | Context invocation after source-record offset-80 write. |
| F-e320c3146c6e | Source-record-derived installation argument. |
| F-6c1bc8c36c27 | Installation-like operation with writable output. |
| F-f26976f63516 | Reconciliation with record, descriptor, scalar, and caller outputs. |
| F-bd138ceb6421 | Removal-like operation; status handling depends on path. |
| F-eef02e922a52 | Associated-record lookup. |
| F-f6d6f6d14101 | Name-related source and destination addresses. |
| F-5ff5fde1145e | Ordered cross-identifier bookkeeping. |
| F-49743e74136c | No-argument provider of final 32-bit context value. |
| F-92a758c7556d | Final context invocation. |

Encoding conversion, descriptor copying, name transfer, and storage installation/removal are inferred roles. Their concrete implementations remain **documented_only**.

## Acceptance examples

These are conditional static expectations, not executed tests.

1. **Initial failure:** source resolution returning `9` produces return `9` before either length invocation.

2. **Equal names:** equal narrowed lengths and zero comparison cause F-U01 with `0x208`, followed by F-946cac4b59bc at destination-descriptor offset 2. Actual descriptor copying is not guaranteed.

3. **Unchanged endpoint:** successful lookup, equal names, and a retained source identifier equal to the requested destination cause a direct current-selector result write and return zero without finalization.

4. **Fallback distinction:** primary lookup `0x30` returns immediately in the alternative format but invokes fallback in the distinguished format.

5. **Normalization to equality:** the unchanged-endpoint fast path is not revisited. If ancestry checking is inapplicable or completes successfully, destination preparation and then source preparation both occur, even with equal identifiers. They are not unconditional.

6. **Normalization rejected before preparation:** suppose identifiers initially differ, normalization makes them equal, the source kind at ancestry selection is `1` or `0x100`, and the derived candidate equals the normalized destination. The function returns `-122` before either F-56433c4250f4 invocation.

7. **Ancestry ordering:** candidate and initial traversal identifier both equal to `1`, with normalized destination different from `1`, cause `-122`, not below-`3` success.

8. **Reconciliation mapping:** initial installation `0x21`, equal current identifiers, different names, and reconciliation `0x20` produce return `0x30`.

9. **Kind `0x100` zero selection:** a zero extraction result is written to source-record offset 8 before associated lookup is skipped.

10. **Kind `0x200`, clear flag:** offset 20 is copied to offset 8 even with flag bit `0x02` clear; associated lookup is skipped.

11. **Retry failure after removal:** successful reconciliation identity checks and source removal followed by installation retry status `7` return `7` without destination cleanup or visible source restoration.

12. **Cleanup preserves error:** associated reinstallation status `7` triggers destination cleanup; the cleanup status does not replace `7`.

13. **Late failure:** a nonzero second bookkeeping status is returned only after the state-word update, final-value invocation and store, and final context invocation.

## Uncertainties

The following remain **documented_only**:

- Concrete namespace semantics, kind meanings, name representation, and valid name bounds.
- Complete object layouts and helper-accessed extents.
- F-U01’s effects and whether it copies exactly 520 bytes.
- Pointer-based helper encoding, byte order, and access widths.
- Name-helper transfer or transformation behavior.
- Dependency mutation, retention, allocation, ownership, locking, durability, and reentrancy.
- Status meanings beyond observed comparisons, mappings, and propagation.
- Caller-result validity outside the explicit fast-path assignment.
- Out-of-range signed-16-bit narrowing.
- Recovery after partially completed installation or removal.
- Termination of ancestry cycles avoiding rejection and unsigned below-`3` termination.

## Evidence

Established behavior derives from the supplied static argument widths, direct field accesses, assignments, branch conditions, invocation ordering, and return handling. The normalization-to-equality path remains subject to ancestry rejection before either preparation invocation; it does not re-enter the unchanged-endpoint fast path.

Move/rename semantics, parent-like relationships, encoding conversion, copying, and name transfer are identified as inferred intent rather than dependency guarantees. Dependency implementations and complete contracts were not supplied.

No runtime tests have been performed.
