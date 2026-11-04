<!-- Trusted provenance: F-aba38163eef3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aba38163eef3

## Purpose

F-aba38163eef3 conditionally prepares a three-field storage descriptor for a requested logical extent, invokes an external operation on any newly exposed range, and records the requested extent.

The visible behavior resembles resizing a byte-oriented buffer with a capacity field. That interpretation is **inferred intent**, not a complete established abstraction. In particular, the supplied material does not establish the allocation dependencies’ ownership rules, the arithmetic helper’s meaning, or the external range operation’s effect on memory.

The central established behavior is:

- Requests no greater than the initially recorded extent bypass allocation and range processing.
- Larger requests use the recorded capacity to decide whether a storage dependency must be called.
- A zero result from that storage dependency triggers a diagnostic call and an immediate zero return, without the function performing its descriptor-update stores.
- All other normal paths write the requested extent to the first descriptor field and return that same requested value.

No runtime tests have been performed.

## Interface

### Inputs

The function takes four arguments:

1. **Descriptor address.** An address interpreted as a sequence of three integer-sized fields.
2. **Requested extent.** A signed integer used in comparisons, arithmetic, the final descriptor update, and the return value.
3. **Auxiliary word A.** An opaque 32-bit value forwarded only to the arithmetic dependency.
4. **Auxiliary word B.** An opaque 32-bit value forwarded twice in the same arithmetic-dependency invocation.

The visible representation uses 32-bit words and a pointer stored in an integer-valued field. This contract describes that recovered 32-bit interface; it does not establish portability to a different integer or pointer width.

### Descriptor layout

The descriptor has the following consecutive fields:

| Byte offset | Width | Contract name | Visible role |
|---|---:|---|---|
| 0 | 32 bits | Recorded extent | Signed comparison operand, range-offset source, and final requested-extent destination |
| 4 | 32 bits | Storage value | Zero/nonzero discriminator, storage-dependency argument, and arithmetic base address |
| 8 | 32 bits | Recorded capacity | Signed comparison operand and destination for a scaled helper result |

“Storage value” denotes the observed address-like word. It does not independently establish that the address is valid, owned by this function, or backed by the recorded capacity.

The external range operation receives the sum of the storage value and an extent. The visible addition has no element-size multiplier. A byte-offset interpretation is therefore reasonable, but the external dependency’s exact memory contract is unavailable.

### Output

The function returns a 32-bit integer:

- **Requested extent** on every normally completed path other than the explicitly handled storage-failure path.
- **Zero** when a storage dependency returns zero and the diagnostic dependency returns normally.

A zero return does not uniquely identify failure. A requested extent of zero also produces zero on a normal completion.

### Preconditions and unsupported inputs

There is no visible validation of:

- A null or inaccessible descriptor.
- Descriptor size or alignment.
- Negative extent or capacity values.
- Agreement between the storage value and the capacity.
- Validity of the destination range.
- Arithmetic overflow.
- Aliasing between the descriptor, storage, and memory accessed by dependencies.

For defined ordinary operation, the caller must provide accessible descriptor fields and satisfy whatever address, size, ownership, and arithmetic constraints the selected dependencies require. Those dependency constraints are **documented_only** where not supplied.

## Behavior

### 1. Initial decision

The function first reads the recorded extent.

It compares that value with the requested extent using a signed comparison. If the recorded extent is greater than or equal to the request, the function does not read the capacity or storage field on this path, does not invoke any visible dependency, and proceeds directly to the final extent store.

Consequently:

- A smaller request records a logical shrink without visible storage release.
- An equal request still performs the final extent store.
- Negative requests are not rejected. Their branch selection follows signed comparison.

### 2. Growth decision

If the initial recorded extent is less than the request, the function reads the recorded capacity and compares it with the requested extent using another signed comparison.

There are two growth paths:

- **Capacity sufficient:** recorded capacity is greater than or equal to the request.
- **Capacity insufficient:** recorded capacity is less than the request.

The capacity comparison uses the capacity value read before any dependency call on this path.

### 3. Growth within recorded capacity

For sufficient capacity, the function reads the storage value and invokes the external range dependency with:

1. The storage value plus the initially read recorded extent.
2. The requested extent minus that initially read extent.

There is no intervening storage allocation call.

The supplied material does not establish that this range operation clears, copies, validates, or otherwise initializes the range. The only established facts are its invocation, argument expressions, and position before the final extent store.

After the dependency returns normally, the function writes the requested extent to the descriptor’s first field and returns the request.

### 4. Growth requiring a storage call

For insufficient capacity, the function invokes F-870c4fcf7295 with five arguments, in this positional order:

1. Requested extent plus three.
2. The value three.
3. Auxiliary word A.
4. Auxiliary word B.
5. Auxiliary word B again.

It then left-shifts the returned integer value by two bits. The resulting word is used both as the size argument to the selected storage dependency and, after a nonzero storage result, as the new recorded capacity.

This is not sufficient evidence to replace the arithmetic dependency with a rounding formula. Its output and side effects remain dependency-defined.

The function reads the descriptor’s storage field **after** F-870c4fcf7295 returns:

- If that field is zero, it calls F-43da6b23bd44 with the scaled size and two zero arguments.
- If that field is nonzero, it calls F-8b998a07b5e1 with the storage value, the scaled size, and two zero arguments.

Thus, the storage choice depends on the field’s value after the arithmetic dependency, not necessarily its value at function entry. No assumption that the arithmetic dependency preserves the descriptor or other shared state is justified.

### 5. Explicit storage-failure path

If the selected storage dependency returns zero, the function calls F-4fc89ec41a6d with these five values:

1. 7.
2. 100.
3. 65, equivalently hexadecimal 41.
4. 0.
5. 0.

If this diagnostic dependency returns normally, the function returns zero immediately.

On this path, F-aba38163eef3 itself does not:

- Store a replacement storage value.
- Store a new recorded capacity.
- Invoke the external range operation.
- Perform the final requested-extent store.

This is not a guarantee that the descriptor or previous storage remains unchanged. Dependencies already called may have observable effects, and their failure and ownership semantics are unknown.

### 6. Successful storage result

When the selected storage dependency returns a nonzero word, the function performs these operations in order:

1. Stores the returned word in the storage field.
2. Stores the scaled arithmetic-helper result in the capacity field.
3. Rereads the recorded extent from the descriptor.
4. Invokes the external range dependency using the returned storage word plus that reread extent, and the requested extent minus that reread extent.
5. After normal return, stores the requested extent in the extent field.
6. Returns the requested extent.

The reread is significant. The range boundary on this path is not necessarily the extent initially observed at entry. A preceding dependency could have changed the descriptor.

The range operation receives the storage dependency’s returned word as its base, rather than a fresh read of the storage field after the descriptor stores.

### 7. Arithmetic boundaries

The visible arithmetic includes:

- Requested extent plus three.
- A two-bit left shift of the arithmetic helper’s result.
- Storage-address addition.
- Requested extent minus the selected prior extent.

These operations are represented with 32-bit integer values. Signed comparisons are explicit in the supplied representation.

Overflow and invalid-shift behavior are not established sufficiently to promise mathematical integers, saturation, checked failure, or a particular wraparound interpretation. Acceptance examples involving these expressions therefore require representable, valid arithmetic unless explicitly discussing an unknown.

In particular, the function contains no visible check that a scaled size is positive, large enough for the request, or consistent with the helper’s result.

## Side effects

### Direct descriptor writes

The normal no-allocation path writes only the recorded extent.

A successful storage-call path writes, in order:

1. Storage value.
2. Recorded capacity.
3. Recorded extent, but only after the external range dependency returns.

The explicit storage-failure path performs none of these direct stores.

A dependency that does not return normally can prevent later stores and the return from occurring. No rollback is visible.

### Storage and ownership

The visible function does not directly release storage.

F-43da6b23bd44 has an allocation-like call shape, and F-8b998a07b5e1 has a replacement-or-resizing-like call shape. These are inferred roles only. The supplied evidence does not establish:

- Whether either dependency allocates memory.
- Whether existing contents are preserved.
- Whether a successful replacement releases the previous storage.
- Whether a failed replacement preserves the previous storage.
- Whether storage aliases the descriptor.
- Whether the caller or a dependency owns the returned storage.

A consumer must not infer ownership transfer or release obligations from this function alone.

### Publication and observation

On the successful storage path, the storage and capacity fields are updated before the range operation, while the final requested extent is written afterward. This ordering is observable to a dependency that can access the descriptor.

No synchronization, atomicity, concurrency safety, or reentrancy guarantee is established.

## Dependencies

| Opaque identifier | Established interaction | Unestablished behavior |
|---|---|---|
| F-870c4fcf7295 | Receives the request-plus-three value, three, and the auxiliary words; its result is shifted left two bits | Arithmetic policy, valid input domain, side effects, preservation of shared state |
| F-43da6b23bd44 | Selected when the post-helper storage field is zero; receives scaled size and two zeros | Allocation policy, initialization, ownership, failure effects |
| F-8b998a07b5e1 | Selected when the post-helper storage field is nonzero; receives storage value, scaled size, and two zeros | Replacement policy, content preservation, release behavior, failure effects |
| F-4fc89ec41a6d | Called after a zero storage result with values 7, 100, 65, 0, 0 | Diagnostic meaning, state changes, termination or return behavior |

One additional external range dependency is present, but no opaque function ID for it was supplied. Its identity and semantics are **documented_only**. Its established interface here consists of the computed range-start value and range-length value. It is not assigned a fabricated function ID.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Unless stated otherwise, arithmetic is representable, addresses satisfy dependency requirements, and dependencies return normally without modifying the descriptor.

### Shrinking

Given recorded extent 20 and requested extent 12:

- No capacity or storage read is required by the visible path.
- No dependency is invoked.
- The recorded extent becomes 12.
- Storage and capacity receive no direct writes.
- The return value is 12.

### Equal extent

Given recorded extent 12 and requested extent 12:

- No dependency is invoked.
- The function stores 12 in the extent field.
- The return value is 12.

### Growth within capacity

Given extent 8, capacity 24, storage value \(P\), and request 14:

- No arithmetic or storage dependency is called.
- The external range dependency receives \(P + 8\) and 6.
- After it returns, the extent becomes 14.
- The return value is 14.

This does not establish what bytes, if any, that dependency writes.

### Successful growth from zero storage

Given extent 4, capacity 4, zero storage, and request 10:

- F-870c4fcf7295 receives 13, 3, auxiliary word A, auxiliary word B, and auxiliary word B.
- If it returns 4, the scaled size is 16.
- F-43da6b23bd44 receives 16, 0, 0.
- If it returns nonzero \(Q\), storage becomes \(Q\), then capacity becomes 16.
- The external range dependency receives \(Q + 4\) and 6.
- After its return, extent becomes 10 and the function returns 10.

The helper result in this example is a stipulated result, not a claimed rounding rule.

### Nonzero-storage failure

Given insufficient capacity and a storage field that remains nonzero after the arithmetic dependency:

- F-8b998a07b5e1 is selected.
- If it returns zero, F-4fc89ec41a6d receives 7, 100, 65, 0, 0.
- If the diagnostic returns, the function returns zero.
- The function performs no descriptor stores and no external range call on this failure path.

Preservation of the previous allocation is not an acceptance requirement established by this evidence.

### Extent changed by a dependency

Suppose entry extent is 4, request is 10, and storage growth is required. If a preceding dependency changes the extent to 7 before the successful storage result is processed:

- The function rereads 7 after storing storage and capacity.
- The external range arguments use returned storage plus 7, and length 3.
- The eventual final extent is 10.

### Zero request

Given entry extent 5 and request 0:

- The function takes the non-growth path.
- It stores zero and returns zero.
- No storage failure is implied.

## Uncertainties

The following remain **documented_only**:

- The arithmetic dependency’s intended computation.
- The external range dependency’s identity and memory effect.
- Allocation, replacement, preservation, and ownership guarantees.
- Dependency effects on the descriptor, globals, or aliased memory.
- Behavior for arithmetic overflow, invalid addresses, or inconsistent descriptors.
- Whether diagnostic invocation returns normally.
- Any domain restriction requiring nonnegative extents or capacities.
- Runtime behavior under concurrent or reentrant access.

The buffer-resizing interpretation is useful inferred intent, but none of these unknowns is resolved by that interpretation.

## Evidence

Established static evidence consists solely of the supplied function representation and the supplied opaque dependency mappings.

That evidence supports the signed branch structure, three-field descriptor accesses, dependency argument values, two-bit scaling operation, direct-store order, extent reread on successful storage growth, and explicit zero-result handling described above.

It does not include dependency implementations, ABI documentation, caller contracts, runtime traces, or observed memory contents. No runtime tests have been performed.
