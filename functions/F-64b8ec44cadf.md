<!-- Trusted provenance: F-64b8ec44cadf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-64b8ec44cadf

## Purpose

F-64b8ec44cadf replaces two header words, performs dependency-driven polling, conditionally processes a linked structure, and invokes a final dependency.

Its established behavior is an ordered sequence of memory accesses and calls. The supplied static evidence does not establish the application-level meaning of the headers, descriptors, addresses, or payloads. Cleanup or destruction is a plausible interpretation, but not a proven ownership or deallocation contract.

Local-storage identity is part of the behavior: three storage areas passed to F-7fa3e55b7211 are subsequently passed to F-d0acc006efef. Two of those areas are overwritten with different values before their reuse. Dependencies retaining their addresses during this invocation can therefore observe those replacements.

## Interface

**Input:** one address interpreted as a pointer to four-byte words. The routine writes its first word and passes an address 180 bytes beyond it to a dependency.

**Output:** no explicit return value. Observable results are direct memory writes, dependency calls, and whatever effects those dependencies produce.

### Widths and address relationships

- Direct word accesses are four bytes wide.
- Polling status and the displayed integer address carriers are 32-bit quantities.
- Pointer-bearing structure fields occupy four-byte words in the supplied representation.
- The polling state occupies three contiguous four-byte words.
- The auxiliary local storage occupies four bytes.
- The local pointer array has two entries; only its first entry is visibly initialized.
- All offsets below are byte offsets. The initial 180-byte displacement comes from advancing 45 four-byte words.
- Polling ends only for status exactly `1`; structure tests compare a four-byte value with zero.

Two dependency results establish address bases:

| Descriptive address | Origin |
|---|---|
| Primary address | Return value of F-dae8ca024616 |
| Secondary address | Return value of F-1b9e6fff9bc9 |

These labels describe arithmetic relationships, not object types or boundaries. The evidence does not establish portable overflow, alignment, or invalid-address behavior. Normal completion requires meaningful derived addresses and valid storage at each actual access; address wrapping must not be treated as validation.

### External values

Three externally stored values have distinct roles:

- An entry header word, read for the write at the input address.
- A primary header word, read for the write 180 bytes before the primary address.
- A shared descriptor, read separately for F-f35532238f81 and F-7fa3e55b7211.

The descriptor reads are not interchangeable with one cached value. Intervening dependencies may modify its storage.

### Local-storage identities

The following descriptive names identify storage, not source identifiers:

| Storage | Established use |
|---|---|
| Original-result slot | Four-byte output address supplied to F-f35532238f81; later read to initialize polling state and the reusable value slot |
| Polling-output slot | Four-byte output address supplied on every polling call |
| Reusable value slot | First holds a copy of the original result; later holds the root’s offset-eight field |
| Reusable reference slot | First holds a copy of the polling output; later holds a freshly read root reference |
| Auxiliary area | Same four-byte area supplied to F-7fa3e55b7211 and F-d0acc006efef |
| Pointer array | First entry holds primary address minus 180 bytes when passed to F-f35532238f81 |

The reusable value and reference slots are distinct from the original-result and polling-output slots. Their identities remain unchanged when their contents are replaced.

There is no explicit validation of inputs, dependency results, local outputs, links, or payloads.

## Behavior

### Header replacement and initial preparation

The routine first reads the external entry header word and writes it into the input’s first four-byte word.

It then calls F-dae8ca024616 with input plus 180 bytes. After that call returns:

1. Its result establishes the primary address.
2. Primary address minus 180 bytes is stored in the first entry of the local pointer array.
3. The external primary header word is written at that derived address.
4. F-f35532238f81 is called with the original-result slot’s address, the then-current shared descriptor, and the pointer array’s address.

Only the pointer array’s first entry has an established initial value. Its second entry and the original-result slot are not visibly initialized before this call.

The third argument exposes pointer-slot storage rather than merely passing the derived address as a scalar. A dependency may therefore inspect or modify that storage according to its own contract. The later primary-address arithmetic uses the saved primary address, not a reload of the pointer array’s first entry.

### Polling-state preparation and progression

After F-f35532238f81 returns, the routine initializes its contiguous polling state in this order:

| Write order | Offset from state address | Written value |
|---|---|---|
| First | 8 | Zero |
| Second | 4 | Zero |
| Third | 0 | Current original-result word |

It then calls F-a1ecaae6738a with the polling-state address and the polling-output slot’s address.

A returned status exactly equal to `1` ends polling. Every other status causes one F-dcbd5e1e578f call with the same polling-state address, followed by another polling call.

Consequences:

- A first status of `1` produces no progression call.
- Zero, negative statuses, and positive statuses other than `1` all take the same progression path.
- State is not reinitialized between polling attempts.
- The same polling-output storage is supplied on each attempt.
- No timeout, attempt bound, or distinct error-status handling is present.
- Polling can continue indefinitely if no call returns `1`.

The routine does not establish that either dependency initializes every word it later reads.

### Preparing reusable local storage

After polling ends, the routine performs two copies in this order:

1. The current polling-output word is copied into the reusable reference slot.
2. The current original-result word is copied into the reusable value slot.

It then calls F-7fa3e55b7211 with:

1. The auxiliary area’s address.
2. A fresh read of the shared descriptor.
3. The reusable value slot’s address.
4. The reusable reference slot’s address.

The auxiliary area has no visible initialization before this call. The dependency receives addresses of the copied words, not the words as scalar arguments. Its contract is unknown: it may read or modify those words, retain their addresses, or affect other accessible state.

These three local addresses are reused later; substituting fresh storage with merely equal contents would not preserve the established identity behavior.

### Secondary-address acquisition

After F-7fa3e55b7211 returns, F-1b9e6fff9bc9 is called with primary address minus 32 bytes. Its result establishes the secondary address.

The remaining external locations are:

| Location | Role |
|---|---|
| Secondary address − 28 bytes | Head slot |
| Secondary address − 12 bytes | Root/reference slot |
| Secondary address − 148 bytes | Address supplied to the final dependency |

These offsets do not establish allocation boundaries, containment, or ownership.

### Root-controlled processing and local-slot reuse

The routine reads the root/reference slot and tests it against zero.

If zero, it skips F-d0acc006efef, F-88d9235286f8, and the entire node-draining loop. A nonzero head alone does not enable draining.

If nonzero, the routine proceeds in this order:

1. Reads the root/reference slot again.
2. Reads the four-byte field eight bytes beyond that referenced address.
3. Overwrites the reusable value slot with that field.
4. Reads the root/reference slot again.
5. Overwrites the reusable reference slot with that reference.
6. Calls F-d0acc006efef with the auxiliary area’s address, the head-slot address, the reusable value slot’s address, and the reusable reference slot’s address.
7. After that call returns, reads the root/reference slot again.
8. Calls F-88d9235286f8 with the head-slot address, that newly read root/reference value, and literal zero.

The address identities across the two calls are exact:

| F-7fa3e55b7211 argument | Later F-d0acc006efef argument | Intervening change |
|---|---|---|
| First: auxiliary-area address | First: same address | No direct overwrite by this routine |
| Third: reusable value-slot address | Third: same address | Copied original result replaced by root offset-eight field |
| Fourth: reusable reference-slot address | Fourth: same address | Copied polling output replaced by freshly read root reference |

Dependency modifications to the auxiliary area are not reset by this routine before its reuse. Dependency modifications to the other two slots may occur, but the routine subsequently replaces their contents in the nonzero-root branch as described.

The root test does not cache a reference for all subsequent operations. In particular, F-88d9235286f8 receives a value read after F-d0acc006efef returns, even if that value is now zero.

There is no intervening dependency between the initial root test and the two slot replacements. Nevertheless, no synchronization contract establishes that the separate root reads must agree under concurrent modification.

### Node draining

After F-88d9235286f8 returns, the routine repeatedly reloads the head slot. A zero head ends draining.

For each nonzero head, that value is saved as the current-node address. The directly accessed node layout is:

| Node-relative offset | Use |
|---|---|
| 0 | Link copied into the head slot |
| 4 | Second argument of the payload-form call |
| 8 | First argument of the payload-form call |

The routine first reads the node’s offset-zero word and writes it into the head slot. It then calls F-ff1bdc376dcd twice:

1. With the node’s offset-eight word, its offset-four word, and literal zero.
2. With the saved current-node address, literal `1`, and literal zero.

The head update precedes both calls. The two payload reads occur after that update; their relative evaluation order is not established by the supplied representation.

The second call uses the saved node address without directly rereading node contents. This does not establish that the first call preserves the node’s lifetime: the address is passed onward regardless.

The next head read occurs only after both calls return. Dependencies may therefore alter which node is processed next. No immutable-list traversal, cycle detection, payload validation, null-payload branch, or iteration bound is established.

### Final processing

After either skipping root-controlled processing or finishing node draining, the routine calls F-57d76f0801f8 with secondary address minus 148 bytes.

If that call returns normally, this routine returns without a value. No success flag or final consistency check is produced.

## Side effects

Direct external writes are limited to the observed operations:

- Replacing the input’s first word.
- Replacing the word at primary address minus 180 bytes.
- Replacing the head slot with each processed node’s offset-zero word.

The routine also initializes, copies, and overwrites local storage exposed to dependencies. Those writes can matter outside the immediate call that first received an address. In particular, F-7fa3e55b7211 may retain the reusable slot addresses, and later activity during this invocation may observe the root-related replacement values or compare the addresses with those supplied to F-d0acc006efef.

Retention alone does not imply a lifetime violation. Subsequent use while this invocation remains active can use the existing local-storage lifetime. No valid arrangement for dereferencing those local addresses after this invocation returns is established.

Additional dependency effects are unknown, including global mutation, allocation, deallocation, synchronization, and ownership transfer. The two calls per node resemble payload and node release operations, but that interpretation remains inferred.

The routine contains no direct input deallocation and no explicit rollback path. If a dependency fails, does not return, or transfers control elsewhere, earlier writes have already occurred. Whether the input or any related storage remains valid afterward depends on contracts not supplied here.

## Dependencies

| Opaque function ID | Established interface use |
|---|---|
| F-dae8ca024616 | Receives input plus 180 bytes; returns the primary address |
| F-f35532238f81 | Receives original-result storage, current descriptor, and pointer-array storage |
| F-a1ecaae6738a | Receives polling state and output storage; status `1` ends polling |
| F-dcbd5e1e578f | Receives polling state after each non-`1` status |
| F-7fa3e55b7211 | Receives auxiliary storage, freshly read descriptor, and reusable value/reference slot addresses |
| F-1b9e6fff9bc9 | Receives primary address minus 32 bytes; returns the secondary address |
| F-d0acc006efef | Receives the same auxiliary and reusable slot addresses, plus the head-slot address |
| F-88d9235286f8 | Receives head-slot address, freshly read root/reference value, and zero |
| F-ff1bdc376dcd | Receives payload-form arguments, then node-address-form arguments, for each processed node |
| F-57d76f0801f8 | Receives secondary address minus 148 bytes as the final call |

No dependency is assumed to preserve globals, leave exposed storage unchanged, initialize outputs, retain or release ownership, or return normally.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Unless stated otherwise, assume valid storage and normally returning dependencies.

### Immediate polling completion and zero root

If the first polling status is `1`, no F-dcbd5e1e578f call occurs. F-7fa3e55b7211 and F-1b9e6fff9bc9 still occur.

If the subsequent root test reads zero, both root-processing calls and all node-processing calls are skipped, even if the head is nonzero. Final processing still occurs.

### Non-success polling statuses

For statuses `0`, `−1`, and `1`, the sequence contains three F-a1ecaae6738a calls and two intervening F-dcbd5e1e578f calls. The negative status does not invoke a separate error path.

### Retained local addresses observe replacement contents

Suppose F-7fa3e55b7211 retains its third and fourth argument addresses. Later, the root branch is taken, the root’s offset-eight field is `37`, and the reference read for the fourth-argument slot is a valid address R.

Before F-d0acc006efef begins, those same local addresses contain `37` and R, respectively. F-d0acc006efef receives those addresses in its third and fourth argument positions. Any valid observation during that call, before further mutation, sees the replacement contents rather than the earlier copied result and polling output.

This scenario requires no extension of local lifetime beyond the active invocation.

### Root changed by a dependency

If F-d0acc006efef changes the root/reference slot to zero, F-88d9235286f8 receives zero as its second argument. The routine nevertheless proceeds to inspect the head afterward; it does not repeat the original branch test.

### One stable node

If the head after F-88d9235286f8 identifies one valid node with a zero next link, and neither per-node dependency changes the head, the head becomes zero before the payload-form call. The node-address-form call follows. The next head read ends draining, then final processing occurs.

### Head changed during node processing

If a per-node dependency replaces the head with another valid node address, the next iteration uses that replacement. Acceptance cannot require traversal of the original link chain independently of dependency effects.

## Uncertainties

The following remain **documented_only**:

- Semantic types and object boundaries for all addresses, headers, descriptors, roots, and payloads.
- Concrete external values and permitted mutations.
- Dependency output initialization, ownership, lifetime, error, and synchronization contracts.
- Whether retained local addresses are actually used, and by which dependencies.
- Any valid post-return lifetime arrangement for exposed local storage.
- Meaning of polling statuses other than their branch effect.
- Relative evaluation order of independent argument reads within a call.
- Address-overflow behavior, alignment requirements, and invalid-memory behavior.
- Termination under persistent non-`1` statuses, cycles, or dependency-driven replenishment.
- Nonlocal control flow or failures originating in dependencies.

Cleanup/destruction remains inferred intent. Address identity, overwrites, branch conditions, and call ordering are established behavior.

## Evidence

This contract uses only the supplied static representation and opaque dependency mapping. Established evidence includes four-byte accesses, address displacements, local-storage reuse, initialization and overwrite order, repeated root and head reads, literal arguments, and dependency-call order.

The two reusable argument slots are visibly assigned both before F-7fa3e55b7211 and before F-d0acc006efef; both calls receive the same respective addresses. The auxiliary area is likewise shared across those calls.

No runtime tests have been performed. Acceptance examples are conditional static scenarios, not observed execution results.
