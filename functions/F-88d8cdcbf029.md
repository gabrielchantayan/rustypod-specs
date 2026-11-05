<!-- Trusted provenance: F-88d8cdcbf029; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-88d8cdcbf029

## Purpose

F-88d8cdcbf029 conditionally performs setup, conditionally prepares shared object state, and returns an integer representation obtained from the shared object-base reference at function exit.

Setup and preparation have independent gates:

- Setup is attempted only when the tested guard’s least-significant bit is clear and F-b255b26284e6 subsequently returns nonzero.
- Preparation occurs only when F-194cdec98e23 returns zero.

Preparation includes a population call, conditional creation of an auxiliary value, and a final two-dependency interaction. Skipping setup does not skip the preparation decision.

“Setup,” “population,” “auxiliary creation,” “temporary cleanup,” and “notification” describe apparent roles. These are inferred intent, not established guarantees about dependency behavior, synchronization, ownership, or successful initialization.

## Interface

### Inputs and external state

There are no explicit formal parameters.

The function uses:

- A shared reference supplying a guard address.
- A shared reference supplying an object-base representation.
- Two external registration values.
- An incoming 32-bit machine value, although it is not a declared argument.
- State accessed or changed by the dependencies.

The incoming machine value is copied into a four-byte local temporary before the guard decision. Its calling-convention significance is unresolved. Its contents can become observable to dependencies that later receive the temporary’s address.

The shared references are sampled at individual use sites. They must not be treated as immutable addresses or as one entry-time snapshot. In particular, a dependency is not established to preserve the guard reference, object-base reference, registration values, or pointed-to storage.

### Result

On normal completion, the function returns the then-current shared object-base representation through its declared integer result.

The return is a fresh use of the shared reference, not the base retained for preparation. On a preparation path, this use occurs after F-8f9bd59e3ec5 returns. On a path that skips preparation, it occurs after F-194cdec98e23 returns.

There is no separate explicit success or error result. Returning an object-base representation does not establish that setup or preparation occurred, or that any dependency succeeded semantically.

The source explicitly identifies four-byte opaque quantities and a four-byte auxiliary-field store. It declares the return as an integer, but does not supply a complete target ABI. Pointer width, integer-to-address interpretation, and portability of the returned representation remain unresolved.

### Object-relative layout

For preparation, let **B** denote the object base retained after guard processing.

| Offset from B | Established use |
|---|---|
| `0x38` — 56 bytes | Address supplied as the second argument to F-793b94643d2e |
| `0x44` — 68 bytes | Address supplied as the fourth argument to F-793b94643d2e and second argument to F-e778256efd99 |
| `0x48` — 72 bytes | Field compared with zero after population; receives a four-byte construction result on the creation path |

The predicate also receives an address at offset `0x44`, but that address is formed from a separate shared-base read rather than from B.

The region beginning at `0x44` is not established to end before `0x48`. Dependencies receiving its address may therefore affect the auxiliary-field decision. No complete object type, region extent, alignment requirement, or non-overlap guarantee is supplied.

### Preconditions

Ordinary defined completion requires:

- The address selected for the initial guard test to support the guard read.
- Each reached dependency to accept the particular arguments supplied at its call site.
- Each object-relative address actually used to designate suitable storage.
- The auxiliary field to support its reached read and conditional four-byte write.
- Every dependency on the selected path to return normally for later operations and the final return to occur.

The body provides no explicit null-base check, bounds check, address-overflow check, allocation-failure check, or validation of returned opaque values. Requirements imposed by dependency bodies remain additional, unresolved preconditions.

## Behavior

### Initial temporary and guard test

The incoming 32-bit machine value is first stored in the local temporary.

The function then obtains the guard address from the shared guard reference and tests the least-significant bit of the value read through that address. The guard access width is not established by the supplied representation.

If the low bit is set, setup is skipped. F-b255b26284e6 is not called.

If the low bit is clear, the shared guard reference is read again for the argument to F-b255b26284e6. This is not specified as reuse of the address used for the bit test.

- A zero acquisition result skips setup.
- Any nonzero acquisition result enters setup; an exact result of one is not required.

Neither outcome bypasses the later preparation predicate.

### Setup sequence and sampling

After a nonzero acquisition result, setup proceeds in this order:

1. The shared object-base reference is read for F-988068049f71. That dependency receives the selected base followed by two zero-valued arguments.
2. After F-988068049f71 returns, its four-byte opaque result becomes the first argument to F-804adc29bbfb. The two external registration values are read for that call and supplied as its second and third arguments, respectively.
3. After F-804adc29bbfb returns, the shared guard reference is read again and supplied to F-d08d3ca17b81.

No relative evaluation order is established between the two external registration-value reads within the registration call. Their argument positions, and their occurrence after setup construction returns, are established.

The setup base can therefore reflect changes made during acquisition. The registration values can reflect changes made during setup construction. The guard address used for completion can reflect changes made during either preceding setup call.

The result of F-804adc29bbfb is not checked. There is no visible failure branch, rollback, or explicit cleanup between setup construction, registration, and guard completion. The function itself does not explicitly write the guard; any guard mutation or publication behavior belongs to dependencies.

### Retained base and preparation predicate

After guard processing, the function reads the shared object-base reference and retains that value as B.

It then makes a separate shared-base read to form the predicate argument: that read’s value plus `0x44`. F-194cdec98e23 receives that address.

These two uses must not be collapsed into one contractual snapshot. The representation shows separate sampling points, although no dependency call lies between them. Whether they can differ through concurrent or other external mutation is unresolved.

The predicate controls preparation:

- Nonzero skips population, the auxiliary-field read, allocation, temporary processing, and the final notification sequence.
- Zero enters preparation.

The source does not establish whether zero means empty, uninitialized, unavailable, or anything else.

Changes made by the predicate to the shared-base reference do not redirect subsequent B-relative operations. Such changes can, however, affect later fresh shared-base reads.

### Population and auxiliary-field decision

On the zero-predicate path, F-8f6053ffecbf is called without explicit arguments.

Its returned four-byte opaque value is supplied to F-793b94643d2e with these argument positions:

1. The returned opaque value.
2. B plus `0x38`.
3. Zero.
4. B plus `0x44`.

Only after F-793b94643d2e returns does the function read the field at B plus `0x48` and compare it with zero.

The decision uses the post-population field value. A value observed before population does not determine whether creation will occur: either population dependency may change relevant state.

If the field is nonzero, auxiliary creation is skipped. The function performs no direct replacement, validation, retention, or release of that existing value.

If the field is zero, the creation sequence follows.

### Auxiliary creation and temporary processing

F-8157a180d901 receives the fixed size value `0x18`, or 24 bytes. There is no size calculation beyond this constant.

After that call returns, F-e778256efd99 receives:

- The address of the local temporary.
- B plus `0x44`.

The allocation result is retained across this call. The temporary still has the earlier initialization as its only explicit prior write in this function, but dependencies may have effects not visible here. Whether F-e778256efd99 reads that initial value, replaces it completely, or requires additional storage is unresolved.

F-1717624e236e then receives:

- The allocation result.
- The temporary’s address.

Its four-byte result is written directly to the field at B plus `0x48`. The stored value is the construction result, not necessarily the original allocation result.

There is no second field test before this write. If a dependency changes the field after the zero test, normal continuation still performs the unconditional construction-result store.

After the store, F-dc2261aac163 receives the temporary’s address. The field write therefore precedes this cleanup-like call. The call receives the temporary address, not the allocation result directly.

No explicit null-allocation test or zero-construction-result test is present. A normally returned zero construction result is stored as zero, after which temporary cleanup and final notification still proceed.

### Final preparation interaction and return

After population and any auxiliary creation and temporary cleanup:

1. F-f56d933c1225 is called without explicit arguments.
2. After it returns, the shared object-base reference is read again.
3. F-8f9bd59e3ec5 receives the returned opaque value as its first argument and that newly read base as its second argument.
4. After F-8f9bd59e3ec5 returns, the shared object-base reference is read again for the function’s return value.

F-1717624e236e has no role in this final interaction. Its only visible call is in auxiliary construction.

The final consumer’s base need not equal B. The returned base need not equal either B or the base passed to the final consumer. Dependency preservation of the shared reference is not established.

## Side effects

The body directly writes:

- The incoming 32-bit machine value into the local temporary before guard processing.
- A four-byte construction result into B plus `0x48`, only on the auxiliary-creation path.

It also performs the guard read, repeated shared-reference and registration-value reads, and the conditional auxiliary-field read described above. These are source-level access relationships, not independently established volatile, atomic, or machine-instruction guarantees.

Dependencies may change shared references, object storage, the guard, external state, or temporary storage. Their apparent roles do not constrain their full effects.

The normal-path ordering guarantees include:

- Temporary seeding before the guard decision.
- Setup construction before registration, then guard completion.
- Guard processing before base retention and the predicate.
- Population before the auxiliary-field read.
- Allocation before temporary construction.
- Auxiliary construction before the field store.
- The field store before temporary cleanup.
- Temporary cleanup, when reached, before the final supplier and consumer.
- The final consumer before the return-value read.

There is no explicit release of the allocation result or an existing auxiliary-field value. Neither the construction call nor temporary cleanup establishes ownership transfer or deallocation without dependency evidence.

No exceptional cleanup or rollback path is shown. If a dependency does not return normally, later calls, stores, and the final return are not established to occur.

## Dependencies

The table specifies visible argument relationships only.

| Opaque function ID | Visible relationship |
|---|---|
| F-b255b26284e6 | Receives a newly sampled guard address after a clear low-bit test; nonzero selects setup |
| F-988068049f71 | Receives a base sampled after successful acquisition, then zero and zero; returns a four-byte opaque value |
| F-804adc29bbfb | Receives the setup result, then the two registration values sampled after setup construction |
| F-d08d3ca17b81 | Receives a newly sampled guard address after registration |
| F-194cdec98e23 | Receives a separately sampled shared base plus `0x44`; zero selects preparation |
| F-8f6053ffecbf | No explicit arguments; supplies the first population argument |
| F-793b94643d2e | Receives that value, B plus `0x38`, zero, and B plus `0x44` |
| F-8157a180d901 | Receives the fixed size 24 |
| F-e778256efd99 | Receives the temporary address and B plus `0x44` |
| F-1717624e236e | Receives the allocation result and temporary address; its result is stored at B plus `0x48` |
| F-dc2261aac163 | Receives the temporary address after the field store |
| F-f56d933c1225 | No explicit arguments; supplies the final consumer’s first argument |
| F-8f9bd59e3ec5 | Receives that value and a shared base sampled after the supplier returns |

No dependency is assumed to preserve globals, avoid reentrancy, retain or release memory, or provide synchronization.

## Acceptance examples

These are conditional static acceptance cases, not runtime observations. They assume valid reached accesses and normal dependency returns.

### Guard marked and preparation skipped

Given a set guard low bit and a nonzero predicate result:

- Acquisition and all setup calls are skipped.
- B is still retained, and the predicate uses its separate base read.
- Population, auxiliary-field inspection, and final notification are skipped.
- The function returns a fresh shared-base read after the predicate.

### Acquisition declines setup

Given a clear tested guard low bit and a zero acquisition result:

- Acquisition receives the guard address sampled for its call, not contractually the earlier tested address.
- Setup construction, registration, and guard completion are skipped.
- Preparation remains independently controlled by the predicate.

### Dependencies change setup references

Suppose acquisition changes the shared base, setup construction changes the registration values, and registration changes the shared guard reference.

Then:

- Setup construction receives the base sampled after acquisition.
- Registration receives the values sampled after setup construction, in their specified argument positions.
- Guard completion receives the guard address sampled after registration.

No entry-time snapshot substitutes for these samples.

### Predicate redirects the shared base

Suppose B is retained, then the predicate changes the shared-base reference and returns zero.

Population, the auxiliary-field test, temporary construction’s object argument, and any field store still use B. Later final-consumer and return reads use the then-current shared reference rather than B.

### Population leaves a nonzero auxiliary field

Given a zero predicate result and a nonzero field after population:

- No allocation or temporary-processing calls occur.
- F-f56d933c1225 runs.
- F-8f9bd59e3ec5 receives its result and the newly sampled shared base.

### Auxiliary construction returns zero

Given a zero field after population and a normally returned zero from F-1717624e236e:

- The allocation request is exactly 24 bytes.
- Zero is stored in the four-byte field.
- Temporary cleanup still runs.
- The final supplier and consumer still run.
- No explicit retry or distinct error result occurs.

### Final dependencies change the shared base

Suppose F-f56d933c1225 changes the shared-base reference to one value and F-8f9bd59e3ec5 later changes it to another.

The final consumer receives the value sampled after the supplier. The function returns the value sampled after the consumer. Neither is required to equal B.

## Uncertainties

The following remain **documented_only**:

- Guard access width, atomicity, synchronization, and thread-safety.
- Whether external references are mutable, and whether concurrent mutation is valid.
- Complete object and temporary layouts, alignment, and dependency access extents.
- Complete target ABI, pointer width, integer width conventions, and address-representation rules.
- Meanings of predicate results and opaque returned values.
- Allocation-failure behavior and whether construction accepts a null allocation result.
- Ownership, lifetime, and release responsibilities for all passed and returned values.
- Reentrancy, exceptions, termination, nonlocal transfer, and dependency-provided rollback.
- The purpose of the incoming 32-bit machine value.
- Any argument-evaluation order not established by the visible sequencing.

Offsets are fixed byte offsets; the allocation size is a fixed constant. No bounds or overflow handling is visible. Validity of reached address calculations is a precondition, not an enforced property.

## Evidence

This contract derives from the supplied static function representation and opaque dependency mapping. The prior draft and reviewer feedback are not independent behavioral evidence.

Established static evidence includes the branch conditions, individual shared-reference sampling sites, argument positions, fixed offsets, four-byte local initialization and field storage, 24-byte request, and normal-return ordering.

Inferred intent includes guarded initialization, population, auxiliary construction, temporary cleanup, and notification. These labels do not establish dependency semantics.

No dependency bodies or implementation environment were available. No runtime tests have been performed.
