<!-- Trusted provenance: F-c890aeef9a26; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c890aeef9a26

## Purpose

F-c890aeef9a26 accepts a fixed quantity, conditionally invokes a dependency based on an existing object field, obtains a four-byte value, and publishes that value into the field. **After publication, it reads a global indirect-call target** and invokes the target with the produced value, the caller’s second argument, and the fixed quantity.

The accepted quantity is exactly 131,072 (`0x20000`). A different quantity selects a dependency annotated as non-returning.

The established contract includes the order of the field write and the global target read. It does not permit the target to be captured before the write. No non-aliasing constraint is supplied between the object field and the global target storage; consequently, a field write that overlaps that storage could affect the target subsequently obtained. This observation does not establish that such aliasing is valid or supported.

The sequence is consistent with fixed-size resource replacement or initialization. That is **inferred intent**, not an established allocation, deallocation, copying, or ownership contract.

## Interface

### Inputs

The recovered interface has three arguments:

| Input | Recovered representation | Established use |
|---|---|---|
| Object address | 32-bit integer used as an address | Identifies a four-byte field at byte offset 2,460 (`0x99C`). |
| Forwarded value | Uninterpreted four-byte value | Passed unchanged as the second explicit argument of the indirect invocation. |
| Required quantity | 32-bit integer | Compared with 131,072. No other use of the supplied quantity is visible. |

The object field occupies relative byte offsets 2,460 through 2,463 inclusive. If the object is represented by contiguous storage beginning at the supplied address, that representation must provide at least 2,464 accessible bytes for these accesses. This is not evidence of the complete object size.

The object address is not locally checked for nullness, accessibility, alignment, or object extent. Ordinary execution requires the field to be readable when the initial check occurs and writable when the later publication occurs. A dependency may change accessibility between those operations; initial validity alone is not a sufficient guarantee.

The forwarded value has no established address, buffer, encoding, or ownership semantics. Any additional requirements on it belong to the unresolved indirect dependency.

### Outputs and completion

No explicit result value is returned. Normal completion occurs after the indirect invocation returns.

The principal explicit output is the four-byte value written into the object field. Dependencies may produce additional effects. The field’s value at function return is not guaranteed to equal the published value because the indirect dependency may modify it.

### Widths and arithmetic

The initial field read, acquired value, field write, and forwarded second argument are four bytes wide.

The quantity comparison is exact equality against 131,072. That constant fits both signed and unsigned 32-bit representations, so signedness does not change which input bit pattern satisfies the comparison.

The field location is calculated from the object address plus 2,460 bytes. No portable overflow, wrapping, invalid-pointer, or alignment-fault behavior is established. Valid address calculation and access are preconditions for ordinary completion, rather than a promise of a particular failure mechanism.

The global target is used as a callable pointer. Its storage width, representation, alignment, and atomicity are not independently established by the supplied material. It must not be assigned a four-byte width merely because the nearby data operations are four bytes wide.

No complete calling convention or endianness specification is supplied.

## Behavior

### Quantity gate

The first visible decision compares the required quantity with 131,072.

If they differ, F-c890aeef9a26 invokes F-a817d8d58c10 without explicit arguments. The supplied static representation annotates this dependency as non-returning. Under that annotation, execution does not proceed to the field read, value acquisition, field write, global target read, or indirect invocation.

The annotation does not identify the failure mechanism. It does not establish whether the dependency terminates a process, transfers control elsewhere, throws, emits a diagnostic, or performs some other non-returning operation.

If the required quantity equals 131,072, execution proceeds to the object-field check without invoking F-a817d8d58c10.

### Existing-field check

The function reads the four-byte field at object-relative offset 2,460 and tests whether its value is zero.

- Zero skips F-58ba61416cc9.
- Any nonzero four-byte pattern causes one invocation of F-58ba61416cc9 without explicit arguments.

Neither the old field value nor the object address is visibly passed to that dependency. Its placement after the nonzero check does not establish that it frees, validates, resets, or otherwise operates on the old value.

If F-58ba61416cc9 returns, execution continues without a visible reread of the field. A dependency-induced change to the field does not trigger another check or conditional invocation.

Whether this dependency returns, and what state it changes, are otherwise unknown.

### Value acquisition

After the existing-field check and any returning conditional invocation, the function invokes F-fab3799b3190 with one explicit argument: 131,072.

The supplied third argument is not forwarded as a variable quantity. The dependency receives the fixed constant.

The dependency produces a four-byte result. No local check classifies that result as success or failure. In particular, zero and other possible sentinel patterns are not rejected by visible local logic.

### Publication

If F-fab3799b3190 returns and the field remains writable, the function writes its four-byte result to the object field at offset 2,460.

This write occurs on both accepted-quantity paths: whether the initial field was zero or nonzero.

There is no visible clearing operation before acquisition, preservation of the old value for recovery, or rollback after publication.

### Global target read and indirect invocation

**Only after the field write does the function read the global indirect-call target.** It then invokes the pointer obtained by that read.

The explicit indirect arguments, in order, are:

1. The four-byte result produced by F-fab3799b3190.
2. The original four-byte forwarded value.
3. The constant 131,072.

The first argument comes from the acquired result, not from a visible reread of the object field. Thus, a later change to the field does not by itself change that argument.

The target read, however, observes the global target storage after publication. Earlier dependencies may have modified that storage. Additionally, no supplied constraint excludes overlap between that storage and the caller-selected field address. If such overlap is possible in the execution environment, the publication may itself change all or part of the target representation before it is read.

This overlap possibility must remain visible in the contract. It neither authorizes overlapping storage nor guarantees that the resulting pointer is valid. A compatible implementation must not assume non-aliasing and cache the target before publication.

The target’s identity and semantics are unresolved. It is not established to be a memory-copy routine or any other standard operation.

If the indirect invocation returns, F-c890aeef9a26 returns without visibly interpreting its result or performing another field write.

### Required observable order

For an accepted quantity, and conditional on each preceding operation completing, the local order is:

1. Read the four-byte object field.
2. If that value is nonzero, invoke F-58ba61416cc9.
3. Invoke F-fab3799b3190 with 131,072 and obtain its four-byte result.
4. Write that result to the object field.
5. Read the global function pointer.
6. Invoke the pointer just read with the acquired result, the original forwarded value, and 131,072.
7. Return after the indirect invocation returns.

Dependency effects may intervene at their invocation points. No dependency is assumed to preserve the object, globals, target storage, or memory accessibility.

This is an ordering contract for the represented operations, not a guarantee of synchronization or cross-thread visibility.

## Side effects

### Explicit local effects

On a completed accepted-quantity path, the function performs:

- One four-byte object-field read.
- Zero or one invocation of F-58ba61416cc9, selected by that read.
- One invocation of F-fab3799b3190.
- One four-byte object-field write.
- One global function-pointer read, after the field write.
- One invocation through the pointer obtained by that read.

These are the visible local operations. Dependencies may perform additional reads, writes, calls, or control transfers.

The global target read is a distinct operation, not merely an unspecified part of invocation setup. Its position after publication matters even without dependency-induced global changes: the field write itself may affect overlapping target storage.

### Ownership and lifetime

No ownership transfer is established.

The supplied material does not prove that:

- The initial field value denotes owned storage.
- F-58ba61416cc9 releases any resource.
- F-fab3799b3190 allocates memory.
- The produced value denotes a valid region of any size.
- The forwarded value denotes readable storage.
- The indirect dependency retains, releases, or takes ownership of either value.

These properties require dependency contracts or additional evidence.

### Errors and partial effects

The only explicit local validation is the required-quantity equality check.

No local recovery is shown for an inaccessible field, a dependency that fails to return, an unreadable target location, an invalid function pointer, or failure during the indirect invocation.

Publication precedes both target resolution and invocation. Therefore, if the target read or indirect invocation fails after the field write has completed, the write has already occurred. No rollback is shown.

Likewise, the function does not provide transactional initialization in which successful indirect processing must precede publication.

No synchronization is visible. Thread safety, reentrancy, atomic pointer access, and memory-ordering guarantees are not established.

## Dependencies

| Dependency | Visible call-site interface | Established role |
|---|---|---|
| F-a817d8d58c10 | No explicit arguments; no result used | Selected for a quantity other than 131,072; annotated non-returning. |
| F-58ba61416cc9 | No explicit arguments; no result used | Selected when the initial object-field read is nonzero. |
| F-fab3799b3190 | One explicit argument, 131,072; four-byte result used | Supplies the value published to the field and forwarded to the indirect invocation. |
| Unresolved indirect dependency; no opaque function ID supplied | Three explicit arguments: acquired result, original forwarded value, 131,072 | Target read from global storage after publication, then invoked. |

The global target location is a data dependency used to select the last invocation. Neither its concrete address nor a stable target identity is required to express this contract.

Only F-a817d8d58c10 carries an explicit non-returning annotation. No dependency is established as pure, memory-safe, ownership-preserving, or global-state-preserving.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not reports of executed tests.

### Accepted quantity with an initially zero field

Given quantity 131,072, an initially zero field, valid accesses, and returning dependencies:

- F-58ba61416cc9 is skipped.
- F-fab3799b3190 receives 131,072.
- Its four-byte result is written to the field.
- The global target is read after that write.
- The pointer obtained by that read receives the result, the original forwarded value, and 131,072.
- The function returns after that invocation returns.

### Accepted quantity with an initially nonzero field

Given the same conditions except for any nonzero initial field value:

- F-58ba61416cc9 is invoked before value acquisition.
- If it returns, the function does not recheck the field.
- Acquisition, publication, target read, and indirect invocation follow in that order.

Changes made by the conditional dependency do not justify using a target captured before publication.

### Rejected quantity

For any quantity other than 131,072:

- F-a817d8d58c10 is invoked before any local object-field access.
- Under its non-returning annotation, no subsequent local field access, global target read, or indirect invocation occurs.

The dependency’s own memory accesses remain unspecified.

### Zero acquired result

If F-fab3799b3190 returns zero:

- Zero is written to the field.
- The global target is then read.
- The target obtained by that read is invoked with zero as its first argument.

There is no intervening local failure branch. Validity and completion of that invocation remain dependency-specific.

### Target changed by an earlier dependency

If an earlier dependency changes the global target, the later target read must observe the storage at its specified post-publication position. A pointer captured before the dependency calls or before the field write does not satisfy this ordering requirement.

### Possible overlap with target storage

If the execution environment permits the object field to overlap global target storage, and the field write changes the stored target representation:

- Target selection uses the representation read after that write.
- A previously captured target is not substituted.
- A valid indirect invocation is required only if the resulting target and access satisfy the environment’s requirements.

This example does not establish overlap as a supported input arrangement.

### Indirect dependency modifies the field

If the indirect dependency changes the published field and then returns, the function does not visibly restore the acquired value. Normal return therefore does not guarantee that the field retains it.

## Uncertainties

The following remain **documented_only**:

- The concrete object type and meaning of its field.
- The purposes and full effects of the dependencies.
- Whether 131,072 denotes bytes, elements, capacity, or another unit.
- Whether either four-byte value denotes an address.
- Allocation, deallocation, ownership, aliasing validity, and lifetime rules.
- The indirect target’s identity, result type, and calling requirements.
- The width and representation of global target storage.
- Whether overlapping field and target storage is possible or supported.
- Dependency failure modes and any internal sentinel conventions.
- Concurrency, atomicity, reentrancy, and synchronization guarantees.
- Exact ABI, endianness, and invalid-address behavior.

The absence of an established non-aliasing guarantee is not evidence that overlap is valid. It is a reason to preserve the observed write-before-target-read order rather than assume overlap away.

If F-a817d8d58c10 unexpectedly returned, the supplied sequence contains no separate local return before the field check. This is a structural observation outside the annotated non-returning contract, not an established supported path.

Resource replacement and fixed-size initialization remain inferred intent only.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping.

Established static evidence includes the exact quantity gate, the annotated mismatch dependency, the four-byte field access at offset 2,460, the nonzero-dependent invocation, the fixed acquisition argument, the four-byte publication, and the subsequent global target read followed by indirect invocation.

The overlap consequence follows from that ordering together with the absence of a supplied non-aliasing constraint. It does not establish the runtime validity of overlapping storage.

No dependency implementation, independent ABI specification, or runtime trace was supplied. No runtime tests have been performed.
