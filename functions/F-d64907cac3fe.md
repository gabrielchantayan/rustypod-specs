<!-- Trusted provenance: F-d64907cac3fe; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d64907cac3fe

## Purpose

F-d64907cac3fe conditionally updates two fields of a destination obtained through a dependency. It always performs an initial destination-acquisition call and brackets the conditional work with two calls using the same eight-byte local storage region.

When either of two 32-bit control values is nonzero, the function obtains an intermediate value, uses it and a byte read from a separate input object to prepare a temporary object, and copies two fields from that temporary object into the destination. It then passes the temporary storage to a cleanup dependency.

The established behavior is a dependency-call sequence plus a conditional, ordered field transfer. Interpreting this as construction, assignment, locking, or resource management would require additional evidence.

## Interface

### Inputs

The function accepts four arguments:

| Argument | Established representation | Use |
|---|---|---|
| Destination token | Opaque 32-bit value | Passed to F-ddec67ee18a9 with two zero arguments. That dependency’s result supplies the destination address. |
| Context address | 32-bit address-bearing value | On the conditional path, supplies a byte at offset `0x44`. |
| First control value | 32-bit integer | Tested against zero and, on the conditional path, forwarded to F-6f3eae7c0190. |
| Second control value | 32-bit integer | Tested against zero and, on the conditional path, forwarded to F-6f3eae7c0190. |

The condition is true if either control value is nonzero. Negative values therefore select the conditional path just as positive values do. There is no range check or normalization.

The destination token is not established to be a direct pointer to the final destination. The initial dependency determines the address used for the explicit destination writes.

### Output

There is no declared return value. Observable results consist of:

- Calls to dependencies, including their arguments and ordering.
- Two explicit destination-field writes on the conditional path.
- Any additional effects produced by dependencies.

An incidental value left in a machine register is not an established return contract.

### Local storage and accessed layouts

Two distinct local storage regions are visible:

| Region | Visible extent | Use |
|---|---:|---|
| Bracketing storage | 8 bytes | Passed first to F-f3d840047ba4 and later to F-036034b4c0cc. |
| Temporary storage | 12 bytes | Passed to F-ddec67ee18a9 on the conditional path and subsequently to F-9beda7955a36. |

These extents describe the visible local allocations, not verified public type sizes or verified bounds on dependency accesses. Neither region is explicitly initialized before its first dependency call.

The relevant externally addressed fields are:

| Object or base | Offset | Access width | Access |
|---|---:|---:|---|
| Context address | `0x44` | 1 byte | Read on the conditional path |
| Object reached through a global root | `0x30` | 4 bytes | Read on the conditional path |
| Destination returned by the first F-ddec67ee18a9 call | `4` | 4 bytes | Written on the conditional path |
| Same destination | `8` | 1 byte | Written on the conditional path |
| Source returned by the second F-ddec67ee18a9 call | `4` | 4 bytes | Read on the conditional path |
| Same source | `8` | 1 byte | Read on the conditional path |

No complete object layout is established. In particular, the function does not explicitly copy the field at offset zero, any padding, or the remainder of either object.

## Behavior

### Unconditional preparation

The function first calls F-ddec67ee18a9 with the destination token and two zero arguments. It retains the returned address for possible later writes.

It then calls F-f3d840047ba4 with the address of the eight-byte bracketing storage.

Only after these calls does it select the conditional path. Thus, both calls occur even when both control values are zero. A zero-control invocation is not established to be side-effect-free.

### Both control values zero

If both control values are zero, the function skips:

- The global-root access and the field read at offset `0x30`.
- F-6f3eae7c0190.
- The context-byte read at offset `0x44`.
- The second F-ddec67ee18a9 call.
- The two explicit destination writes.
- F-9beda7955a36.

It proceeds directly to F-036034b4c0cc with the eight-byte bracketing storage and then returns normally, provided that dependency returns normally.

The context address is not explicitly dereferenced on this path. That observation does not establish any dependency’s behavior for an invalid context address or an invalid destination token.

### At least one control value nonzero

After the initial two calls, the conditional path proceeds as follows.

1. **Obtain an intermediate value.**  
   The function obtains an object address through a global-root indirection and reads its four-byte field at offset `0x30`. That field value becomes the first argument to F-6f3eae7c0190.

   The second argument to F-6f3eae7c0190 is an unresolved secondary machine-register value exposed by the supplied representation. It is not a declared input to F-d64907cac3fe, and its semantic origin is not established.

   The third and fourth arguments are the first and second control values, respectively. They are forwarded without visible modification.

2. **Prepare the temporary object.**  
   After F-6f3eae7c0190 returns, the function reads one byte at context offset `0x44`.

   It calls F-ddec67ee18a9 with:
   - The address of the twelve-byte temporary storage.
   - The 32-bit result of F-6f3eae7c0190.
   - The byte obtained from the context.

   The returned address becomes the source of the field transfer. It is not established that this returned address equals the temporary-storage address.

3. **Transfer the four-byte field.**  
   The function reads four bytes at source offset `4` and writes that value to destination offset `4`.

4. **Transfer the one-byte field.**  
   After the four-byte write, the function reads one byte at source offset `8` and writes it to destination offset `8`.

5. **Invoke temporary cleanup.**  
   The function calls F-9beda7955a36 with the address of the twelve-byte temporary storage. It passes the storage address, not explicitly the source address returned by F-ddec67ee18a9.

6. **Invoke the final bracketing dependency.**  
   The function calls F-036034b4c0cc with the same eight-byte storage address previously passed to F-f3d840047ba4.

The function then returns normally if all preceding operations and dependencies complete normally.

### Ordering and aliasing

The two field transfers are sequential, not an established snapshot operation. The four-byte source read and destination write precede the one-byte source read.

This distinction matters if the source and destination overlap. If the first destination write changes the memory subsequently read as the source byte, the second transfer observes that changed memory. The function does not visibly reject overlap or prefetch both source fields before writing.

The context byte is read after F-6f3eae7c0190 returns. If that dependency changes the context byte through an alias, the later read observes the state left by the dependency.

The global field is accessed after F-f3d840047ba4 returns. No assumption that the earlier calls preserve the global root or its referenced object is justified.

## Side effects

### Direct effects

The only explicit externally addressed writes are:

- A four-byte write at destination offset `4`.
- A one-byte write at destination offset `8`.

Both occur only on the conditional path, and the four-byte write occurs first. They are not an atomic multi-field update.

The function has no explicit error result, validation branch, allocation operation, or global write. Dependencies may perform any of those actions; their bodies are not supplied.

### Ownership and lifetime

The visible local storage exists for this invocation. Passing its address to a dependency does not establish ownership transfer or permission to retain the address beyond the invocation.

The function does not directly free the destination or source addresses. F-9beda7955a36 is used in a cleanup position, but its actual ownership behavior is unknown. In particular, the evidence does not establish whether it releases resources, invalidates the returned source address, or changes external state.

Likewise, the paired use of the eight-byte storage suggests a bracketed operation, but does not prove locking, reference counting, transaction management, or any other particular mechanism.

### Preconditions and failures

For normal execution, each explicit read and write must address accessible storage with any alignment required by the target environment. On the conditional path this includes the global-root-derived field, the context byte, and the fields reached through both returned addresses.

The supplied representation uses 32-bit address-bearing values and fixed offsets. It does not establish behavior for invalid pointers, address overflow, misalignment, inaccessible memory, or concurrent conflicting access. No bounds checks are visible.

If a dependency fails, faults, or does not return, subsequent operations are not guaranteed to occur. The visible straight-line cleanup calls do not establish exception-safe or fault-safe cleanup. For example, a failure after the first destination write may leave a partially updated destination.

## Dependencies

| Opaque ID | Invocation and established role | Unresolved behavior |
|---|---|---|
| F-ddec67ee18a9 | Called first with the destination token and two zero values; conditionally called again with temporary storage, an intermediate value, and a context byte. Each result is used as an address. | Object construction, address identity, validation, ownership, allocation, and side effects. |
| F-f3d840047ba4 | Called once with the eight-byte storage before conditional work. | Storage initialization, register effects, global effects, and any bracketing semantics. |
| F-6f3eae7c0190 | Called only when either control is nonzero, with a global-derived field, an unresolved register value, and both controls. Its 32-bit result feeds temporary preparation. | Meaning of the result, interpretation of controls, and effects on reachable state. |
| F-9beda7955a36 | Called with the twelve-byte storage after both explicit field writes. | Resource release, external effects, and resulting object validity. |
| F-036034b4c0cc | Called with the eight-byte storage after the conditional work, or immediately after the branch is skipped. | Finalization semantics and effects on the destination or other state. |

No dependency is assumed to preserve globals, preserve addressed objects, retain no pointers, or own or release memory.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not reports of executed tests.

### Zero-control path

Given both controls equal zero and the unconditional dependencies return normally:

- The first F-ddec67ee18a9 call precedes F-f3d840047ba4.
- F-036034b4c0cc follows F-f3d840047ba4.
- There is no second F-ddec67ee18a9 call and no F-6f3eae7c0190 or F-9beda7955a36 call.
- The function performs neither explicit destination-field write.
- The function does not explicitly read the context byte.

Dependency-produced destination changes remain possible.

### Either control selects the update

Given one control is zero and the other is any nonzero 32-bit value, the conditional path is selected. This includes a negative control value. F-6f3eae7c0190 receives both original controls in their original positions.

### Ordinary nonoverlapping transfer

Suppose the returned source and destination are valid, nonoverlapping objects. At the respective reads, the source word at offset `4` is `0x12345678` and the source byte at offset `8` is `0xA5`.

Immediately after the two explicit stores, the corresponding destination fields contain those values. Persistence through function return additionally requires that the subsequent dependencies do not alter them.

### Context mutation by a dependency

Suppose F-6f3eae7c0190 changes the context byte from `0x02` to `0x07` before returning. The byte subsequently supplied to the second F-ddec67ee18a9 call is `0x07`, provided no intervening external mutation occurs.

### Overlapping field transfer

Suppose the first destination write overlaps the location used for the later source-byte read. The acceptance criterion is the actual sequential result: the byte read after that write is transferred. A requirement to preserve the source’s earlier byte value would contradict the visible ordering.

### Failure between stores

If the four-byte destination store succeeds but the later byte access faults, the evidence provides no rollback guarantee and no guarantee that either cleanup-position dependency runs.

## Uncertainties

The following are **documented_only**:

- The semantic meaning of the destination token and both controls.
- The identity and full layout of the globally reached object.
- The origin and validity of F-6f3eae7c0190’s unresolved second argument.
- The calling-convention extension of the one-byte argument beyond its established byte value.
- Whether either F-ddec67ee18a9 result aliases its first argument.
- Complete object sizes, alignment requirements, byte order, and permitted alias relationships.
- Dependency effects, ownership rules, failure conventions, and retained-pointer behavior.
- Whether the bracketing calls implement synchronization or another lifecycle.
- Concurrency guarantees and behavior under invalid memory or address overflow.

**Inferred intent:** the function appears to construct or acquire a temporary representation and transfer selected state into a destination under a bracketed context. This interpretation is not an additional behavioral guarantee.

## Evidence

The established contract comes solely from the supplied static function representation: argument widths, local-storage extents, fixed-offset accesses, conditional branching, dependency arguments, and the ordering of calls and field transfers.

No dependency bodies, external interface declarations, runtime observations, or execution traces were supplied. No runtime tests have been performed. Semantic names such as “destination,” “context,” “temporary,” and “cleanup” describe visible roles without asserting undocumented types or ownership rules.
