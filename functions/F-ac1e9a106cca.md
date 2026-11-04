<!-- Trusted provenance: F-ac1e9a106cca; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ac1e9a106cca

## Purpose

F-ac1e9a106cca conditionally performs a two-object setup sequence and writes a completion value into shared storage.

The supplied representation establishes two ordered evaluations of a global record reference:

1. The first value is retained for subsequent direct record accesses.
2. The second value supplies the record whose 32-bit field at byte offset 16 is tested as the entry guard.

Equality of these reference values requires a stability assumption. The representation does not establish volatile semantics or require two distinct physical memory loads.

If the guard is nonzero, the function returns without dependency calls or direct writes. Otherwise, it:

- Obtains a value from F-8157a180d901 and passes it to F-c5cc2791e2c4.
- Publishes the initializer result at offset 8 of the retained record.
- Makes six calls to F-0fdd13c3e50f, deriving arguments through that record field.
- Obtains and initializes a second object, publishing its reference at retained-record offset 12.
- Writes 1008 at byte offset 16 relative to the second initializer result.
- Copies the then-current retained-record offset-8 value into retained-record offset 16.

The apparent intent is guarded initialization. Successful construction, ownership transfer, and thread-safe one-time initialization are not established.

## Interface

### Inputs and result

The function has no explicit parameters and no returned result. Its effective inputs are the global record reference, reachable memory, two fixed data references, and dependency behavior.

For clarity:

- **Retained record** means the address obtained by the first evaluation of the global reference.
- **Guard record** means the address obtained by the subsequent evaluation used for the guard.

These names distinguish reference evaluations, not necessarily distinct objects. Under stable global state, both designate the same record.

After the guard, direct record accesses use the retained record. Later changes to the global reference do not redirect those accesses. Changes to the retained record’s contents can still affect behavior.

The supplied representation uses four-byte fields and 32-bit integer or reference-like values. This establishes the widths relevant here, not a portable interface for other address widths, byte orders, or alignment rules.

### Record fields

| Record | Byte offset | Width | Observed use |
|---|---:|---:|---|
| Guard record | 16 | 32 bits | Read once for the nonzero entry guard |
| Retained record | 8 | 32 bits | Receives the first initializer result; supplies object references during processing and the final copied value |
| Retained record | 12 | 32 bits | Receives the second initializer result |
| Retained record | 16 | 32 bits | Receives the final copy from retained-record offset 8 |

When the two reference evaluations agree, these fields belong to one record. Otherwise, the guard does not test the record later modified by this function.

No layout is established for other record bytes.

### Referenced storage

Each processing call obtains values through references read from retained-record offset 8:

| Referenced-object byte offset | Width | Use |
|---|---:|---|
| 0 | 32 bits | Integer value used as the base of the first argument |
| 24 | 32 bits | Integer value passed as the second argument |

The first argument is address-like in apparent use, but the supplied representation computes it by integer addition.

The second initializer result is subsequently used as the base of a four-byte write at byte offset 16. Neither this access nor the calls with argument 32 establishes a complete object type or allocation extent.

### Preconditions for ordinary completion

The executed path requires:

- Availability of the global reference for the two entry evaluations.
- A readable four-byte field at offset 16 of the guard record.
- On the initialization path, writable retained-record fields at offsets 8, 12, and 16, and readable offset-8 storage when it is reloaded.
- Valid object-field reads for every reference actually obtained through retained-record offset 8 during processing.
- A valid four-byte destination at offset 16 relative to the second initializer result.
- Dependency arguments satisfying any additional requirements imposed by those dependencies.
- Normal dependency returns and valid intervening accesses if execution is to reach completion.

On the nonzero-guard path, the function does not dereference the retained record. Its value therefore need not identify accessible record fields merely for that path to return.

These conditions must remain true at each relevant access; the function does not ensure that callees preserve memory validity. No explicit null, bounds, alignment, or failure checks enforce them.

## Behavior

### Entry reference evaluations and guard

First, the function evaluates the global record reference and retains its value.

Next, it evaluates the global reference again and reads the four-byte field at offset 16 through that second value. This second evaluation, not the retained value, determines the guard address.

These are ordered reference evaluations in the supplied representation. They are not evidence of volatile access, a synchronization operation, or two mandatory physical loads. In particular, the function supplies no mechanism guaranteeing that the evaluations observe different values—or guaranteeing equality in an execution model that permits relevant changes.

Any nonzero guard value causes immediate return. No specific sentinel or valid-reference interpretation is required. A bit pattern interpreted as a negative signed integer is also nonzero.

This return path makes no dependency calls and performs no direct writes.

A zero guard enters initialization. The guard is never checked again. If the guard record differs from the retained record, the retained record’s existing offset-16 value does not control entry.

### First result and publication

The function calls F-8157a180d901 with the integer argument 32.

Its returned four-byte value is passed to F-c5cc2791e2c4 with three further arguments, in this order:

1. A fixed data reference, designated **descriptor A**.
2. The integer 10.
3. The integer 2.

The initializer’s four-byte result is stored at retained-record offset 8.

The result is neither checked nor required by this function to equal the preceding dependency result. Calling the first dependency with 32 does not establish allocation success, zero-filled storage, or ownership transfer.

Descriptor A is passed as a reference. Its contents and format are not established by this function’s own accesses.

### Six-call processing phase

After the first publication, the function makes six calls to F-0fdd13c3e50f, provided all calls and accesses complete normally.

For successive call positions zero through five:

- The first argument is the 32-bit integer read at referenced-object offset 0, plus respectively 0, 4, 8, 12, 16, or 20.
- The second argument is the 32-bit integer read at referenced-object offset 24.

For each argument, the object reference is obtained through retained-record offset 8. The representation contains separate accesses through that field for the two argument expressions. It does not establish an evaluation order between those expressions.

Consequently:

- The contract does not require the offset-0 object read to precede the offset-24 read.
- A single saved first-object reference must not replace all subsequent field reloads when doing so would change observable behavior.
- There is no unconditional guarantee that both arguments, or all six calls, derive from the same object.

Under stable relevant memory, every call uses the same base value and second argument, with first-argument increments of four. Dependencies are not assumed to preserve that stability.

The six-call count follows from a 32-bit unsigned selection value. Its tested values are 16, 32, 64, 128, 256, and 512, each of which intersects mask 1008. After the next left shift, 1024 does not intersect the mask, so processing ends. This sequence does not overflow the unsigned width.

The position count advances from zero to six. Multiplication by four produces at most 20 while constructing a call argument, so that multiplication does not overflow a 32-bit signed integer.

Adding the displacement to the loaded signed integer is a separate issue. The representation does not establish defined wraparound for arbitrary base values or prove that the resulting integer identifies valid storage. Ordinary examples below assume representable sums and arguments valid for F-0fdd13c3e50f.

Any result from F-0fdd13c3e50f is ignored.

### Second result and completion

After all six processing calls, the function calls F-8157a180d901 again with 32.

It passes that result to F-c5cc2791e2c4 with:

1. A second fixed data reference, designated **descriptor B**.
2. The integer 10.
3. The integer 1.

The initializer result is stored at retained-record offset 12.

Next, the function writes the four-byte value 1008, equivalently 0x3F0, at byte offset 16 relative to that returned result. This destination is derived directly from the result, not from a fresh read of retained-record offset 12.

Finally, the function reads the then-current four-byte value at retained-record offset 8 and writes it to retained-record offset 16.

The copied value need not equal the first initializer’s original result. Dependency effects or overlapping direct writes may have changed offset 8. No dependency calls follow this final write.

## Side effects

### Direct write order

On a normally completing initialization path, the function’s direct writes are ordered as follows:

1. First initializer result to retained-record offset 8.
2. Second initializer result to retained-record offset 12.
3. Value 1008 to second-result-relative offset 16.
4. Current retained-record offset-8 value to retained-record offset 16.

The six processing calls occur between the first and second writes.

These are logical destinations, not guaranteed distinct memory locations. The second-result-relative write may overlap a retained-record field if the returned value and memory model permit that arrangement.

The guard read uses the guard record. The writes above use the retained record or the second initializer result. There is no separate completion write redirected through the second entry reference evaluation.

Dependencies may perform additional reads, writes, allocation, release, or other effects. No preservation guarantee for globals, record contents, or object lifetimes is supplied.

### Partial state and failure

There is no explicit rollback or error result.

The first result is published before processing begins. The second result is published before its offset-16 write. In a non-overlapping ordinary execution, completion is written last.

If a dependency fails to return normally, subsequent steps are not reached through the normal path. If an access faults, earlier effects may remain. The function provides no explicit recovery branch.

A zero first initializer result is stored and then used through subsequent reference loads unless intervening effects replace it. A zero second initializer result is likewise used to form the offset-16 write destination. Neither result triggers a safe failure path; actual consequences depend on memory validity and the execution model.

### Ownership, repetition, and concurrency

No direct release operation is present. Ownership of either dependency result, consumption of dependency inputs, and responsibility for eventual release remain unknown.

Repetition is controlled by the guard record selected on each invocation. Under a stable global reference identifying the retained record, a nonzero final completion value suppresses a later invocation while that field remains unchanged. A zero final value does not.

Without that stability condition, completion of one retained record does not establish what a later invocation will test. Even within one invocation, differing entry reference values permit a zero guard in one record to authorize writes to another.

No lock, atomic protocol, or reentrancy protection is established. The entry reference evaluations and guard do not constitute a coherent snapshot or safe one-time initialization protocol.

## Dependencies

| Opaque function ID | Established invocation and use | Not established |
|---|---|---|
| F-8157a180d901 | Called twice with 32 on a normally completing initialization path; each result feeds the following initializer | Allocation semantics, failure convention, side effects, ownership |
| F-c5cc2791e2c4 | Called with the preceding result, descriptor A, 10, 2; later with the second preceding result, descriptor B, 10, 1; results are published | Descriptor formats, construction semantics, aliasing, ownership, failure handling |
| F-0fdd13c3e50f | Called six times with arguments derived through retained-record offset 8; result unused | Target accesses, mutation, release behavior, argument requirements |

The normal initialization path contains ten dependency calls in this order: F-8157a180d901, F-c5cc2791e2c4, six calls to F-0fdd13c3e50f, F-8157a180d901, then F-c5cc2791e2c4.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests.

### Nonzero guard

Given a readable nonzero field at offset 16 of the guard record, the function returns without dependency calls or direct writes.

This remains the selected branch even if the retained record’s offset-16 field is zero. The function does not dereference that retained field on this path.

### Different entry reference values

If the execution model permits the first global-reference evaluation to identify one record and the second to identify another, a zero offset-16 field in the second record permits initialization of the first record.

Assuming valid, non-overlapping storage and no relevant dependency mutations, the first record receives publications at offsets 8 and 12 and completion at offset 16. The second record is only read by this function.

This example does not require two physical global loads or assert that unsynchronized concurrent modification has defined behavior.

### Stable ordinary initialization

Given equal entry reference values, a zero guard, valid disjoint objects, representable integer sums, normally returning dependencies, and no relevant mutations:

- Retained-record offset 8 receives the first initializer result.
- Processing first arguments are the first object’s offset-0 integer plus 0, 4, 8, 12, 16, and 20, in order.
- All six second arguments equal that object’s offset-24 integer.
- Retained-record offset 12 receives the second initializer result.
- Second-result-relative offset 16 receives 1008.
- Retained-record offset 16 receives the first initializer result.

### Processing replaces the published reference

If the first processing call changes retained-record offset 8 to another valid object reference and leaves it there, subsequent calls derive their arguments through that replacement.

Unconditionally retaining the original first-object reference for all six calls would not preserve this behavior.

### Later global replacement

If a dependency replaces the global record reference, subsequent direct record accesses still use the first entry reference value. No guard re-evaluation occurs.

This does not prevent the dependency from also modifying or invalidating the retained record.

### Current-value completion and aliasing

If the second initializer changes retained-record offset 8, returns valid non-overlapping storage, and no later effect changes offset 8, completion receives that replacement value.

Alternatively, if the second-result-relative offset-16 write validly overlaps retained-record offset 8, that field becomes 1008 before the final copy. Completion consequently receives 1008.

### Repeated invocation

With a stable global reference, a completed nonzero offset-16 value suppresses the next invocation. If completion is zero, another initialization attempt is permitted.

If the global reference changes, the later invocation instead follows its own two entry evaluations and guard.

## Uncertainties

The following remain **documented_only**:

- Application-level meanings of the objects, descriptors, and constants.
- Whether 32 denotes allocation size or another quantity.
- Meanings of initializer arguments 10, 2, and 1.
- Complete layouts, alignment rules, byte order, and supported addressing model.
- Dependency failure conventions, nonlocal control flow, and memory effects.
- Ownership, retention, consumption, and release obligations.
- Permitted aliasing and concurrency.
- Physical realization of the two global-reference evaluations.
- Evaluation order of sibling processing-call argument expressions.
- Defined behavior for arbitrary signed base-plus-displacement arithmetic and integer-derived addresses.

“Initialization” and “completion” describe inferred intent. The calls, field accesses, widths, values, and stated ordering are established static behavior.

## Evidence

The supplied representation establishes:

- A parameterless function with no returned result.
- An initial retained global-reference value followed by a separate global-reference evaluation for the offset-16 guard.
- Subsequent direct record accesses based on the first value.
- Two calls with argument 32 and two initializer calls distinguished by fixed references and final arguments 2 and 1.
- Publications at retained-record offsets 8 and 12.
- Six processing calls with four-unit integer increments and object-field reads at offsets 0 and 24.
- A four-byte write of 1008 relative to the second initializer result.
- A final four-byte copy from retained-record offset 8 to offset 16.
- No explicit result checks, rollback, direct release calls, or synchronization.

This contract is based on static evidence only. No runtime tests were performed, and no dynamic outcome is claimed.
