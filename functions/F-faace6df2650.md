<!-- Trusted provenance: F-faace6df2650; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-faace6df2650

## Purpose

F-faace6df2650 conditionally obtains a record reference using a context and a descriptor. It first compares one 32-bit word from each object. A mismatch returns null without calling any dependency or directly modifying storage.

For matching words, the function may invoke F-3931e02c2857. A nonnull result from that dependency is returned immediately. Otherwise, the function invokes F-704d91f94e04 to acquire a record reference. A nonnull acquisition result is linked into the descriptor and passed to two further dependencies.

Every fallback path also merges a descriptor mask into a context mask, including when acquisition returns null. A successful result from F-3931e02c2857 bypasses that merge.

“Context,” “descriptor,” “record,” and “link” describe roles inferred from the accesses. Neither allocation nor fresh storage is established: F-704d91f94e04 may return storage obtained by an unknown mechanism.

## Interface

The function accepts four arguments and returns a record reference or null.

| Argument | Established use |
|---|---|
| Context reference | Identifies storage containing a comparison word, a dependency argument, a control byte, and an accumulated mask. |
| Descriptor reference | Identifies storage containing a comparison word, a link word, and a contributed mask. |
| Auxiliary word A | Opaque 32-bit value passed to selected dependencies. |
| Auxiliary word B | Opaque 32-bit value supplied in two argument positions to F-3931e02c2857. |

The visible representation is a 32-bit model. Integer fields below occupy four bytes, and references written into fields occupy 32-bit words. The evidence does not establish a portable calling convention or pointer representation for wider address spaces.

All offsets below are byte offsets from the corresponding reference.

### Context layout

| Offset | Width | Access |
|---|---:|---|
| `0x0C` | 32 bits | Read for the initial comparison. |
| `0x40` | 32 bits | Read as the first argument to F-3931e02c2857, if called. |
| `0x18D` | 8 bits | Conditionally read; bit `0x10` controls eligibility for F-3931e02c2857. |
| `0x1A8` | 32 bits | Read and written by the fallback mask merge. |

### Descriptor layout

| Offset | Width | Access |
|---|---:|---|
| `0x00` | 32 bits | Read for the initial comparison. |
| `0x24` | 32 bits | Tested for eligibility, subsequently read as a previous link, and replaced on nonnull fallback acquisition. |
| `0xBC` | 32 bits | Read as the contributed mask on fallback. |

### Record layout used after nonnull fallback acquisition

| Offset | Width | Access |
|---|---:|---|
| `0x00` | 32 bits | Read after F-493743038bd6 returns; its value is increased by `0x48` for the final dependency call. |
| `0x10` | 32 bits | Read and copied to record offset `0x24`. |
| `0x20` | 32 bits | Written with the descriptor reference. |
| `0x24` | 32 bits | Written with the value read from record offset `0x10`. |
| `0x28` | 32 bits | Written with the descriptor link value read during linking. |

The function also reads a separate data object to obtain the second argument to F-ebd0ee53699f. Its value, storage relationship to the supplied objects, declared type, and read width are not established by the supplied fragment. It is not established to be immutable.

### Preconditions and validity

Accessible storage is required only for the operations actually reached. There is no visible validation of null references, bounds, alignment, or writability.

Even the comparison-mismatch path requires both comparison words to be readable. A zero initial descriptor link avoids the context control-byte read, but does not avoid fallback accesses.

For nonnull fallback acquisition, the returned reference must support the specified record accesses for execution to complete normally. Required context, descriptor, record, and data-object storage must remain valid through each later access. The caller cannot infer that dependencies preserve this validity, retain references, or refrain from releasing storage.

Invalid accesses and abnormal dependency termination have no specified recovery behavior in this function.

## Behavior

### Comparison gate

The descriptor word at `0x00` is compared with the context word at `0x0C`. Equality is a comparison of the complete 32-bit values; their interpretation as signed or unsigned does not change this equality test.

If the values differ:

- The function returns null.
- No dependency is invoked.
- No direct write occurs.
- No later field access is reached.

Both operand reads precede the branch decision. Their relative evaluation order is not established.

### Eligibility and the first dependency

For matching comparison words, the eligibility control value initially represents zero.

The function tests the descriptor link at `0x24`. If it is nonzero, the context byte at `0x18D` is read and retained for the eligibility decision. If the initial link test is zero, that byte is not read.

A subsequent link test and the retained control value determine eligibility. F-3931e02c2857 is reached only if:

- The subsequent descriptor-link test is nonzero; and
- Bit `0x10` of the retained control value is set.

Other control-byte bits do not affect eligibility. The source expresses two link tests; this does not independently establish two physical memory transactions in a compiled implementation or define behavior under concurrent mutation.

When called, F-3931e02c2857 receives these five arguments, in positional order:

1. The word from context offset `0x40`.
2. The descriptor reference.
3. Auxiliary word A.
4. Auxiliary word B.
5. Auxiliary word B again.

Argument position is established; relative evaluation order among argument expressions is not.

A nonnull result is returned directly. This function performs no subsequent linking, post-link calls, or mask merge on that path.

A null result selects fallback. Ineligibility selects the same fallback without invoking F-3931e02c2857. There is no distinct visible status code for these two causes.

### Fallback acquisition

Fallback invokes F-704d91f94e04 once with the context reference.

If the result is null, record linking and both post-link calls are skipped. The fallback mask merge still occurs, after which the function returns null.

If the result is nonnull, it becomes the record reference used for linking, both post-link calls, and the eventual return. Nothing establishes that this storage is fresh, cleared, unshared, or newly allocated.

### Linking a nonnull acquisition result

The following direct operations occur in order:

1. The descriptor reference is written to record offset `0x20`.
2. The word at record offset `0x10` is read and written to record offset `0x24`.
3. The descriptor link at `0x24` is read and written to record offset `0x28`.
4. The acquired record reference is written to descriptor offset `0x24`.

The link copied into the record is the value read during this sequence. It is not necessarily the value used in an earlier eligibility test: a preceding dependency or an overlapping earlier write may have changed it.

These writes precede both post-link calls. No direct rollback is provided if later work fails to complete normally.

### Post-link calls and late argument reads

After linking, F-493743038bd6 is called with:

1. The acquired record reference.
2. Auxiliary word A.
3. The numeric value `1`.

After that call returns, the function prepares a call to F-ebd0ee53699f with:

1. The 32-bit word read from record offset `0x00`, increased by `0x48`.
2. The value read from the separate call-argument data object.
3. The acquired record reference.
4. Zero.
5. Zero.

Both the record-word read and the data-object read occur when preparing this final call, after F-493743038bd6 returns. Their relative evaluation order is not established.

The second argument is a read of storage, not an established fixed value. It can reflect changes caused by preceding dependencies or by preceding writes that alias the data object. No snapshot of this argument is established before acquisition, linking, or F-493743038bd6.

Likewise, the first argument uses the record word as it exists after F-493743038bd6 returns. It does not use a value cached before that call.

The addition applies to the 32-bit word in the record, not to the record reference itself. Its meaning as an address calculation, handle adjustment, or ordinary integer calculation is unknown. Because the visible expression uses a signed integer value and the fragment does not establish execution-level overflow semantics, overflowing inputs do not have a portable specified result. In particular, wrapping modulo \(2^{32}\) must not be assumed.

Neither post-link dependency’s return value is used by the visible control flow.

### Fallback mask merge and return

After fallback acquisition—and after both post-link calls when acquisition was nonnull—the function:

- Reads the context mask at `0x1A8`.
- Reads the descriptor mask at `0xBC`.
- Writes their 32-bit bitwise OR to context offset `0x1A8`.

The relative order of the two operand reads is not established. Both occur after the preceding dependencies have returned normally. Values modified by those dependencies or by earlier aliasing writes therefore participate in the merge.

This merge occurs on every normally completing fallback path, including null acquisition. It is not an atomic fetch-and-OR.

The return value is the acquisition result. The descriptor link is not reread to determine that result. A dependency changing the descriptor link does not itself change the reference selected for return.

## Side effects

The function’s direct writes are path-dependent:

| Path | Direct writes |
|---|---|
| Comparison mismatch | None. |
| F-3931e02c2857 returns nonnull | None. |
| Fallback acquisition returns null | Context mask only. |
| Fallback acquisition returns nonnull | Three record fields, the descriptor link, and the context mask. |

These are direct-write guarantees, not guarantees of overall absence of effects. Every invoked dependency may have additional effects not established here.

No visible operation explicitly frees storage, adjusts a reference count, declares ownership transfer, restores a previous link, or performs cleanup after abnormal termination. A returned reference does not establish caller ownership. Replacing the descriptor link does not establish that the previous link is retained or released.

Aliasing can change the values consumed by later reads. The descriptor-reference write to record offset `0x20`, for example, precedes the read from record offset `0x10` and the later descriptor-link read. If storage overlaps, the ordered access behavior governs; a nonoverlapping linked-record interpretation is only conditional.

The final-call data object also has no established non-aliasing or immutability guarantee. Its read must not be replaced in this contract by a presumed constant.

No synchronization, inter-thread visibility, reentrancy guarantee, or concurrency protocol is established.

## Dependencies

| Opaque ID | Established interaction | Unresolved behavior |
|---|---|---|
| F-3931e02c2857 | Conditionally receives a context-sourced word, descriptor reference, auxiliary word A, and auxiliary word B twice. Nonnull returns immediately; null selects fallback. | Lookup semantics, allocation, ownership, validation, and other effects. |
| F-704d91f94e04 | Receives the context reference on fallback. Its result determines whether linking and post-link calls occur. | Allocation versus reuse, initialization, lifetime, failure causes, and other effects. |
| F-493743038bd6 | Receives the acquired reference, auxiliary word A, and `1`, after descriptor linking. | Meaning of arguments, storage mutation, ownership, release behavior, and other effects. |
| F-ebd0ee53699f | Receives a record-word-derived value, a late-read data-object value, the acquired reference, and two zeros. | Argument meanings, data-object value, registration or dispatch semantics, ownership, and other effects. |

Later operations require preceding calls to return normally. Exceptions, nonlocal transfers, process termination, or other abnormal completion do not have visible recovery handling.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Unless stated otherwise, assume valid storage throughout, no relevant overlap, stable memory except for specified dependency effects, normally returning dependencies, and representable arithmetic.

### Comparison mismatch

If the comparison words differ, the result is null. No dependency is called and no direct write occurs. Control-byte, link, and mask contents do not affect this outcome.

### Zero link

With matching comparison words and a stable zero descriptor link, F-3931e02c2857 is skipped and F-704d91f94e04 is invoked. Context offset `0x18D` need not be readable for this path; other reached accesses still require valid storage.

### Control bit clear

With matching comparison words, a nonzero link, and control byte `0xEF`, fallback is selected. Bit `0x10` is clear despite the other set bits.

### Successful first dependency

With matching comparison words, a nonzero link, control byte `0x10`, and a nonnull result from F-3931e02c2857, that reference is returned. No fallback acquisition, post-link call, or direct mask update follows.

### First dependency changes the link and returns null

If F-3931e02c2857 changes the descriptor link and returns null, fallback follows. A nonnull acquisition copies the link present at the later linking read into record offset `0x28`, not necessarily the original link.

### Null acquisition still merges masks

With fallback selected, a null acquisition result, context mask `0x80000001`, and descriptor mask `0x00000006`, the context mask becomes `0x80000007` and the result is null. Neither post-link dependency is invoked.

### Nonnull acquisition links the returned storage

If the acquired record contains `0x12345678` at offset `0x10`, and the descriptor link is `0x00002000` at the linking read:

- Record offset `0x20` receives the descriptor reference.
- Record offset `0x24` receives `0x12345678`.
- Record offset `0x28` receives `0x00002000`.
- Descriptor offset `0x24` receives the acquired reference.

F-493743038bd6 and F-ebd0ee53699f then run in that order, followed by the mask merge. This example does not require fresh allocation.

### Post-link mutation changes both final-call inputs

If F-493743038bd6 leaves the record’s first word as `0x1000` and changes the call-argument data object to another valid value, F-ebd0ee53699f receives `0x1048` as its first argument and the subsequently read data-object value as its second argument.

Neither input is required to match its value before F-493743038bd6. The order of the two reads remains unspecified.

### Final dependency changes the masks

If F-ebd0ee53699f leaves the context mask as `0x00000020` and the descriptor mask as `0x00000003`, the subsequent merge stores `0x00000023`, regardless of their earlier values.

## Uncertainties

The following remain **documented_only**:

- Application-level types and meanings of the supplied objects.
- Whether acquisition allocates, reuses, or shares storage.
- Ownership, lifetime, retention, and release obligations.
- Dependency implementations, global-state preservation, and failure behavior.
- The call-argument data object’s value, declared type, read width, purpose, mutability rules, and aliasing relationships.
- Exact ABI, alignment requirements, and address-space interpretation beyond the visible 32-bit model.
- Results of overflowing arithmetic in the final call’s first argument.
- Supported overlapping-storage arrangements and concurrent mutation behavior.
- Behavior following invalid memory access or abnormal dependency termination.

The record-linking and existing-record interpretations are inferred intent. They do not override the established access order, argument positions, or conditional paths.

## Evidence

Established static evidence consists of the supplied function body’s comparison, field access widths and offsets, conditional control-byte access, eligibility conditions, dependency argument positions, ordered linking writes, late final-call argument reads, fallback mask merge, and return paths.

The dependency mapping establishes opaque identities only. It supplies no implementations, ownership guarantees, or preservation guarantees.

The separate data-object expression establishes a read for the final call after F-493743038bd6 returns; it does not establish an immutable value. Likewise, a nonnull acquisition result establishes a reference used by this function, not fresh allocation.

No runtime tests have been performed. The acceptance examples are conditional static expectations, not observed executions.
