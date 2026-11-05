<!-- Trusted provenance: F-f2f640f1bcce; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f2f640f1bcce

## Purpose

F-f2f640f1bcce conditionally runs an initialization-like sequence on a caller-supplied object. A 32-bit value at object byte offset 4 controls entry: a nonzero value causes an immediate return of 0; zero permits the sequence.

On the attempted path, the function calls an entry dependency, stores two caller-supplied values in the object, prepares local state, and invokes several dependencies. The 32-bit result from F-195f5d43c91b is written to object offset 4. A nonzero result triggers one additional dependency and ultimately returns 1. A zero result omits that dependency and ultimately returns 0. Both attempted paths perform the same two final calls, assuming normal dependency returns.

“Initialization,” “entry,” and “cleanup” describe inferred roles. The supplied representation establishes the operations and their order, not the dependencies’ semantic contracts.

## Interface

The function receives five arguments:

| Position | Representation and established use |
|---|---|
| 1 | Object address, used for direct 32-bit accesses and for computing addresses passed to dependencies. |
| 2 | A 32-bit value stored at object offset 0 on the attempted path. |
| 3 | A 32-bit value passed unchanged to the second invocation of F-44362605ab79. Its interpretation is unknown. |
| 4 | A 32-bit value stored at object offset 8 on the attempted path. |
| 5 | A 32-bit value passed unchanged as the fourth argument to F-195f5d43c91b. Its interpretation is unknown. |

The function returns a 32-bit Boolean-like result:

- **0:** The initial guard was nonzero, or F-195f5d43c91b returned zero and the remaining calls returned normally.
- **1:** F-195f5d43c91b returned nonzero and all subsequent calls returned normally.

The nonzero result from F-195f5d43c91b is not returned directly. It is stored in the object.

### Object layout used here

All offsets below are byte offsets from the object address.

| Offset | Established access or use |
|---|---|
| 0 | A 32-bit write of the second argument. |
| 4 | A 32-bit guard read; on the attempted path, a later 32-bit write of the result from F-195f5d43c91b. |
| 8 | A 32-bit write of the fourth argument. |
| 12 | Address supplied first to F-338d16f9ce72 and later to F-1333331b2da1. |
| 20 | Address supplied to F-162a41bac6a7 only after a nonzero result. |

The direct field accesses are four bytes wide. Passing the addresses at offsets 12 and 20 does not establish the size or layout of the regions those dependencies access.

### Local storage

The supplied representation declares two separate 32-bit local storage objects relevant to preparation:

- One receives zero.
- The other receives a value read from a global seed location. Its address is passed to dependencies.

Their adjacency is not established. Neither their membership in a shared descriptor nor a two-word descriptor layout is established. In particular, the zero-initialized local cannot be specified as the second word of the address-exposed local.

The address-exposed local is explicitly four bytes wide. The representation does not establish the complete storage extent that dependencies expect to access through its address. Any interpretation as a larger temporary descriptor remains inferred.

### Preconditions

The caller must provide memory suitable for the operations reached on the selected path. Even the immediate-return path requires a valid 32-bit read at object offset 4.

The attempted path additionally requires writable storage for the direct object fields and valid inputs for every reached dependency. The dependencies’ requirements are not supplied.

There is no explicit null check, alignment check, object-size check, or address-overflow check. The representation does not define a recoverable error outcome for invalid memory or invalid dependency inputs.

## Behavior

### Guard and immediate return

The function first reads the 32-bit field at object offset 4.

If that value is nonzero:

- It returns 0.
- It makes no dependency calls.
- It performs no explicit object writes.
- It does not prepare the two local storage objects described above.

Every nonzero 32-bit guard pattern selects this path. The guard is not interpreted as a signed error code or compared against a particular nonzero constant.

If the guard is zero, the function proceeds. There is no later guard recheck.

### Entry and publication of caller values

On the attempted path, the function computes the object address at offset 12 and calls F-338d16f9ce72 with that address.

After the call returns, the function explicitly writes:

1. The second argument to object offset 0.
2. The fourth argument to object offset 8.

Both writes are 32 bits wide. The function does not explicitly write object offset 4 at this stage.

No supplied contract requires F-338d16f9ce72 to preserve the object or global state. If that dependency changes offset 4, this invocation nevertheless continues: entry was already selected by the earlier zero observation.

### Local preparation

After the object writes, the function performs two distinct local initializations in this order:

1. A separate 32-bit local receives zero.
2. Another 32-bit local receives the value read from the global seed location.

Only the second local’s address is explicitly passed onward. The zero-initialized local has no further explicit use in the supplied representation.

The function then calls F-1f62160dfd52 with:

1. The address of the seed-initialized local.
2. The address of a global configuration location.
3. The original object address.

The second argument is an address, not an explicitly loaded copy of the configuration contents. The configuration’s type and extent are unknown.

F-1f62160dfd52 may modify the address-exposed local, the object, or other reachable state; no preservation guarantee is supplied. Subsequent calls receive the same local address, not a guaranteed unchanged seed value.

A shared descriptor spanning both locals is a possible interpretation, not an established layout. No behavior in this contract depends on that interpretation.

### Ordered extraction calls

F-44362605ab79 is called twice, in a fixed order:

1. With the address of the seed-initialized local, after F-1f62160dfd52 has returned.
2. With this function’s third argument.

Each result is represented as a 32-bit value. The first result is retained across the second call and used later.

“Extraction” or “conversion” is only an inferred role. The supplied evidence does not establish that either call is pure, reads only its argument, preserves globals, or returns an address rather than another kind of value.

### Result-producing call

The function next invokes F-195f5d43c91b with seven arguments:

| Position | Supplied value |
|---|---|
| 1 | Result of the second F-44362605ab79 call, which received this function’s third argument. |
| 2 | Value read from a global selector location. |
| 3 | Result of the first F-44362605ab79 call, which received the local address. |
| 4 | This function’s fifth argument. |
| 5 | Value read from a distinct global option location. |
| 6 | Constant 1. |
| 7 | Constant 60. |

The selector and option are distinct global inputs. Their meanings and values are not established. Their reads belong to this call’s argument preparation, after the two completed extraction calls. Their relative read order is not established by the supplied representation.

The return from F-195f5d43c91b is a 32-bit result. The function writes the entire result to object offset 4 before executing either result-dependent path.

Only zero versus nonzero matters. A high-bit-set result takes the nonzero path; there is no signed-negative failure branch and no narrowing to a smaller stored field.

### Nonzero-result path

After writing a nonzero result to offset 4, the function calls, in order:

1. F-162a41bac6a7 with the object address at offset 20.
2. F-289ec4311c64 with the address of the seed-initialized local.
3. F-1333331b2da1 with the same offset-12 address previously supplied to F-338d16f9ce72.

It then returns 1.

The branch decision uses the result returned by F-195f5d43c91b. The function does not reread object offset 4 to decide its return value. Later dependency changes to that field therefore do not change the selected return value.

### Zero-result path

After writing zero to object offset 4, the function calls:

1. F-289ec4311c64 with the address of the seed-initialized local.
2. F-1333331b2da1 with the previously computed offset-12 address.

It then returns 0. F-162a41bac6a7 is not called.

There is no explicit rollback of the values written at offsets 0 and 8, no restoration of a previous offset-4 value, and no retry.

## Side effects

### Explicit memory operations

On every normally progressing attempted path, the direct object writes occur in this order:

1. Offset 0 receives the second argument.
2. Offset 8 receives the fourth argument.
3. Offset 4 receives the result from F-195f5d43c91b.

The local zero initialization occurs before the global seed read and the initialization of the address-exposed local. These are separate operations on separate declared storage objects; they do not establish contiguous descriptor storage.

The function directly reads three global values: the seed, selector, and option. It also passes a fourth global location’s address to F-1f62160dfd52. No direct assignment to those global locations is shown. Dependencies may nevertheless change them.

Dependencies occur between the direct memory operations. Consequently, final object contents cannot be guaranteed solely from the explicit writes unless the relevant dependency behavior is constrained separately.

### Ownership and lifetime

The object is caller-supplied. No explicit allocation or deallocation of the object appears.

The two local storage objects belong to the invocation. The seed-initialized local’s address is exposed to F-1f62160dfd52, F-44362605ab79, and F-289ec4311c64. F-289ec4311c64 is called once on either normally returning attempted path.

That last call resembles cleanup, but its name and placement do not establish that it releases memory or any other resource. No ownership transfer, retention rule, or release obligation is established for the local’s contents or the dependency results.

The representation does not establish whether any dependency retains the local address. Safe retention beyond its invocation-local lifetime would require additional dependency evidence; this function does not provide such a guarantee.

### Concurrency and abnormal exits

The guard read precedes F-338d16f9ce72, and no guard recheck follows that call. Even if the entry and exit dependencies provide mutual exclusion, the function alone does not establish that concurrent callers initialize the object only once.

Atomicity and memory-order guarantees are unknown. The direct accesses must not be assumed to provide synchronization merely because they are 32 bits wide.

The stated final-call sequences assume normal returns. No local recovery path is visible for a dependency that faults, throws, terminates execution, or otherwise fails to return. In particular, later cleanup calls are not guaranteed after an abnormal exit.

## Dependencies

| Opaque function ID | Established interaction |
|---|---|
| F-338d16f9ce72 | First dependency on the attempted path; receives the object’s offset-12 address. Synchronization entry is inferred. |
| F-1f62160dfd52 | Receives the seed-initialized local’s address, a global configuration address, and the object address. |
| F-44362605ab79 | Called first with the local address and then with the third argument; supplies two 32-bit results for F-195f5d43c91b. |
| F-195f5d43c91b | Receives seven arguments; its 32-bit result is stored at offset 4 and determines the zero/nonzero branch. |
| F-162a41bac6a7 | Receives the offset-20 address only after a nonzero result has been stored. |
| F-289ec4311c64 | Receives the local address once during either normally progressing attempted-result path. Cleanup is inferred. |
| F-1333331b2da1 | Receives the same offset-12 address used by the entry call, after F-289ec4311c64 returns. Synchronization exit is inferred. |

No supplied dependency contract establishes preservation of object fields, globals, local contents, or referenced memory. Return values from dependencies other than the two extraction calls and F-195f5d43c91b are not used by this function.

## Acceptance examples

These are conditional predictions from static evidence, not executed tests. Unless stated otherwise, memory is valid and dependencies return normally.

### Initially nonzero guard

Given an offset-4 value of 7:

- Return 0.
- Call no dependencies.
- Perform no explicit writes to the object or the two preparation locals.

The same behavior applies to any nonzero 32-bit guard pattern.

### Zero result with no subsequent field mutation

Given an initially zero guard and F-195f5d43c91b returning zero:

- Call the entry dependency before writing the two caller values.
- Write the second and fourth arguments at offsets 0 and 8.
- Write zero at offset 4 after the result-producing call.
- Omit F-162a41bac6a7.
- Call F-289ec4311c64 before F-1333331b2da1.
- Return 0.

If dependencies do not overwrite those fields after their explicit writes, offsets 0 and 8 retain the caller values despite the zero result.

### High-bit-set result

Given an initially zero guard and a result of 0x80000000:

- Store the full 32-bit pattern at offset 4.
- Call F-162a41bac6a7 with the offset-20 address.
- Perform the two final calls in order.
- Return 1.

The high bit does not select a failure path.

### Guard mutation during entry

If the initial guard is zero but F-338d16f9ce72 changes it to nonzero:

- Continue the attempted sequence without rechecking.
- Later overwrite offset 4 with the result from F-195f5d43c91b.

### Mutation after the result is stored

If F-195f5d43c91b returns nonzero and F-162a41bac6a7 subsequently sets offset 4 to zero, the function still returns 1 after normal completion of the remaining calls. A nonzero final field value is not an unconditional postcondition.

### Local preparation without a layout assumption

Suppose the global seed contains 0x12345678 at the seed read:

- The separate zero-initialized local receives zero first.
- The address-exposed local then receives 0x12345678.
- F-1f62160dfd52 receives the latter local’s address.
- Later address-taking calls use that same local address.

This scenario requires neither adjacency of the locals nor an eight-byte descriptor. It does not establish that a read four bytes beyond the passed address would reach the zero-initialized local or would be valid.

## Uncertainties

The following remain **documented_only**:

- The object’s semantic identity and the meanings of its fields and embedded regions.
- Whether the dependencies implement synchronization, construction, conversion, notification, cleanup, or other operations.
- Whether the two preparation locals are related in the original implementation.
- Their relative addresses, padding, adjacency, or membership in any shared descriptor.
- The extent and layout expected by dependencies receiving the seed-initialized local’s address.
- Global values, semantic types, and widths beyond those established by their visible use.
- The representation and validity requirements of the third and fifth arguments beyond their 32-bit interface representation.
- The meanings of constants 1 and 60.
- Ownership, resource release, and address-retention behavior.
- Complete object size, alignment requirements, native pointer width, byte order, and ABI details.
- Atomicity, memory ordering, and concurrent-initialization safety.
- Behavior following invalid accesses or abnormal dependency exits.

No unresolved dependency behavior is replaced here with an assumption of purity, preservation, or resource ownership.

## Evidence

This contract uses only the supplied static function representation and opaque dependency mapping. They establish the direct access widths and offsets, guard behavior, ordered local initializations, address arguments, dependency sequence, result storage, zero/nonzero branching, and return constants.

The representation declares two separate 32-bit preparation locals. It does not establish a two-word descriptor, their adjacency, or the accessible extent beyond the local whose address is passed.

Lifecycle and synchronization terminology is inferred intent rather than an established dependency guarantee. No runtime tests have been performed; the acceptance examples are conditional static predictions.
