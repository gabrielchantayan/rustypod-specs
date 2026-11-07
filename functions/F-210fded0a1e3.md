<!-- Trusted provenance: F-210fded0a1e3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-210fded0a1e3

## Purpose

F-210fded0a1e3 obtains a record from a source-associated collection, initializes selected record fields, and either submits the record to a destination-associated collection or returns it to the source collection. It optionally waits for a record to become available and optionally performs a second wait after submission.

The established result convention is a 32-bit value of **0** for the submission path and **1** for either immediate unavailability or the path that returns a record to the source. These meanings describe this function’s control flow; they do not establish whether a destination later processes the submitted record successfully.

The apparent use of synchronization handles, notifications, and a current execution context is inferred from call patterns. The dependency implementations are unavailable, so their synchronization guarantees, collection semantics, and memory-ownership effects remain unspecified.

## Interface

The function receives six arguments:

| Argument | Representation and use |
|---|---|
| Source object | Pointer to storage accessed through 32-bit fields and embedded objects. |
| Destination object | Address-valued 32-bit argument used as the base of byte-offset accesses. |
| Payload | Opaque 32-bit value written into the acquired record. |
| Availability-wait selector | 32-bit integer. Zero selects one acquisition attempt; any nonzero value selects repeated attempts separated by dependency calls. |
| Completion-wait selector | 32-bit integer. Its full-width zero/nonzero value controls optional context setup and post-submission waiting. Its low eight bits are also stored in the record. |
| Destination restriction selector | 32-bit integer. A nonzero value can cause the acquired record to be returned to the source instead of submitted, subject to the completion selector and destination state. |

The return value is a 32-bit quantity explicitly initialized to zero and changed to one on two paths. No other return values are established.

### Referenced layouts

Offsets below are byte offsets. They describe only storage this function accesses or passes to dependencies, not complete object definitions.

**Source object**

| Offset | Access |
|---|---|
| `+0x00` | Read as a 32-bit handle before calls to F-3fef4e222917 and F-599da3979baf. |
| `+0x08` | Address passed to F-162a41bac6a7 while waiting for an available record. |
| `+0x14` | Address passed to F-08e7aeef99f4 and, on the return-to-source path, F-37efbe955e3e. |
| `+0x20` | Written with a context-derived address; later read indirectly to obtain a 32-bit handle; eventually cleared. |
| `+0x24` | Written with another context-derived address; later read as an argument to F-162a41bac6a7; eventually cleared. |
| `+0x28` | Written with the acquired record address before the post-submission wait, then cleared. |

The direct accesses require source storage extending through `+0x2b`. Embedded objects passed to dependencies may require additional storage.

**Destination object**

| Offset | Access |
|---|---|
| `+0x2c` | Read as a 32-bit handle before calls to F-3fef4e222917 and F-599da3979baf. |
| `+0x34` | Address passed to F-acda426cbea1 after submission. |
| `+0x40` | Conditionally read as a 32-bit integer and passed by address to F-37efbe955e3e. |

**Acquired record**

| Offset | Access |
|---|---|
| `+0x08` | Address passed to F-c082dad756a2. |
| `+0x0c` | Written with the 32-bit payload. |
| `+0x10` | Written with the byte value 1. |
| `+0x11` | Written with the low eight bits of the completion-wait selector. |

**Context returned by F-3fdd14ecd250**

| Offset | Access |
|---|---|
| `+0x3c` | Address passed to F-b00260d1087a; also treated as containing a 32-bit handle. |
| `+0x44` | Address passed to F-79048eba87f4. |
| `+0x50` | Read as one byte; written with 1 after conditional initialization. |

### Preconditions

All executed memory accesses must be valid, and address-valued dependency results must be usable in the roles described below. The function performs no explicit null checks on the source, destination, or optional context.

An acquisition result of zero is handled as unavailability. Any nonzero result is treated as a record address without validation.

The supplied representation uses 32-bit words and integer-held addresses. It does not establish portability to a wider-pointer environment. No explicit address-overflow or alignment checks are present.

## Behavior

### Acquisition phase

The function first reads the source handle at `+0x00` and passes it to F-3fef4e222917.

With a zero availability-wait selector, it calls F-08e7aeef99f4 exactly once with the address of source `+0x14`.

With a nonzero selector, it repeatedly calls F-08e7aeef99f4 with that same address. Each zero result is followed by a call to F-162a41bac6a7 with the address of source `+0x08`, after which acquisition is attempted again. No retry bound, timeout, cancellation branch, or dependency-error branch is visible.

After the acquisition loop or single attempt finishes, the function reads the source handle again and passes the newly read value to F-599da3979baf. It does not preserve the initial handle for this call. F-162a41bac6a7 may therefore be called between the two handle-oriented calls; whether that dependency releases or reacquires any resource is unknown.

If acquisition produced zero, the function calls F-3fdd14ecd250, ignores its return value, and returns 1. No record initialization, destination access, or optional completion setup follows on this path.

### Record preparation

For a nonzero acquisition result, the function performs these observable actions in order:

1. Calls F-c082dad756a2 with the address of record `+0x08`.
2. Writes the payload as 32 bits at record `+0x0c`.
3. Writes the low eight bits of the completion-wait selector at record `+0x11`.
4. Writes byte value 1 at record `+0x10`.

The byte-store order is significant: the selector byte is written before the byte at `+0x10`.

The full completion-wait selector continues to govern later branches independently of its stored byte. A nonzero selector whose low eight bits are zero still enables optional completion behavior.

### Optional context preparation

When the completion-wait selector is nonzero, the function calls F-3fdd14ecd250 and uses its result as a context address.

It reads the byte at context `+0x50`. If that byte is zero, it:

- Calls F-b00260d1087a with context `+0x3c`.
- Calls F-79048eba87f4 with context `+0x44` as its first argument and context `+0x3c` as its second.
- Writes byte value 1 at context `+0x50`.

Any nonzero initial byte skips both initialization calls and the flag write.

The function then writes context `+0x3c` into source `+0x20`, followed by context `+0x44` into source `+0x24`. It reads the 32-bit value at context `+0x3c` and passes that value to F-3fef4e222917.

The source fields are not saved and restored. Previous values at `+0x20` and `+0x24` are overwritten.

### Destination decision

After any optional context preparation, the function reads the destination handle at `+0x2c` and passes it to F-3fef4e222917.

Submission occurs if at least one of these conditions holds:

- The destination restriction selector is zero.
- The completion-wait selector is nonzero.
- The 32-bit value at destination `+0x40` is zero.

The destination-state read is conditional. It is reached only when the restriction selector is nonzero and the completion-wait selector is zero. The source therefore does not establish an unconditional state read at `+0x40`.

On submission, the function calls F-37efbe955e3e with destination `+0x40` and the acquired record address. It then calls F-acda426cbea1 with destination `+0x34`. The return value remains zero.

The alternative path requires all three conditions:

- A nonzero destination restriction selector.
- A zero completion-wait selector.
- A nonzero 32-bit value at destination `+0x40`.

On that path, the function reads the source handle and passes it to F-3fef4e222917, calls F-37efbe955e3e with source `+0x14` and the acquired record address, then rereads the source handle and passes it to F-599da3979baf. It sets the result to 1. F-acda426cbea1 is not called.

The record preparation writes occur before this decision and are not undone when the record is returned to the source.

After either path, the function rereads the destination handle at `+0x2c` and passes it to F-599da3979baf.

### Optional post-submission wait

A nonzero completion-wait selector enables this final sequence. Under the established branch conditions, that selector also forces the submission path, so this sequence does not follow the return-to-source path.

After the destination handle’s F-599da3979baf call, the function:

1. Writes the acquired record address into source `+0x28`.
2. Reads source `+0x24` and passes that stored value to F-162a41bac6a7.
3. Writes zero to source `+0x28`.
4. Reads source `+0x20`, reads a 32-bit handle through that address, and passes the handle to F-599da3979baf.
5. Writes zero to source `+0x20`, then zero to source `+0x24`.

The function does not reread record status, check a completion predicate, or inspect the wait dependency’s return value. One such call is visible. Whether it can return without the intended completion condition is unknown.

## Side effects

Direct writes are limited to the record fields, the source bookkeeping fields, and the conditional context initialization byte described above. Dependencies may cause additional effects that are not bounded by the supplied evidence.

The function visibly orders calls in pairs around source, destination, and optional context operations. Calling these pairs “lock/unlock” is inferred intent, not an established dependency contract.

No explicit allocation, deallocation, payload copy beyond the single 32-bit store, or payload dereference is shown. This does not prove that dependencies allocate or release memory.

The acquired record is passed either to the destination-associated operation or to the source-associated operation. This establishes record routing, not ownership transfer or lifetime rules. In particular, the optional path retains the record address and publishes it in source `+0x28` after destination submission. Any guarantee that the record remains usable at that point must come from the surrounding system.

There is no visible rollback if a dependency fails to return normally. Clearing the optional bookkeeping fields occurs only after the post-submission wait returns and the subsequent operations execute.

## Dependencies

The names below identify dependencies opaquely. Descriptions state their use here, not independently verified behavior.

| ID | Use by this function |
|---|---|
| F-3fef4e222917 | Receives 32-bit values read from source, destination, or context handle fields. |
| F-599da3979baf | Receives corresponding handle values at later control-flow points; those values are reread rather than necessarily reused. |
| F-08e7aeef99f4 | Receives source `+0x14`; its zero/nonzero result controls acquisition. |
| F-162a41bac6a7 | Receives source `+0x08` after unsuccessful acquisition, or the value stored at source `+0x24` during the optional final wait. |
| F-c082dad756a2 | Receives record `+0x08` before the record field writes. |
| F-3fdd14ecd250 | Called with no arguments on immediate unavailability, with its result ignored; also supplies the optional context address. |
| F-b00260d1087a | Receives context `+0x3c` during conditional context initialization. |
| F-79048eba87f4 | Receives context `+0x44` and context `+0x3c` during that initialization. |
| F-37efbe955e3e | Receives a destination- or source-associated collection address and the acquired record address. |
| F-acda426cbea1 | Receives destination `+0x34` after destination submission. |

Except for acquisition and context lookup, no dependency return value is used to select behavior. No dependency is assumed to preserve globals, aliased fields, handles, or memory ownership.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. They assume valid storage and dependencies that return normally. Specific observations of stored values additionally assume dependencies do not independently overwrite those values.

1. **Immediate unavailability.** With availability waiting disabled and acquisition returning zero, exactly one acquisition attempt occurs. The source handle’s F-599da3979baf call precedes F-3fdd14ecd250. The function returns 1 without accessing the destination.

2. **Availability retry.** With availability waiting enabled and acquisition results zero, zero, then a valid address, two calls to F-162a41bac6a7 using source `+0x08` occur between three acquisition attempts. Record preparation begins only after the successful result and the source handle’s F-599da3979baf call.

3. **Submission without restriction.** With a valid record, both wait selectors zero, and the restriction selector zero, destination `+0x40` is not directly read for the branch decision. F-37efbe955e3e receives destination `+0x40` and the record, followed by F-acda426cbea1. The result is 0.

4. **Restriction rejects the record.** With completion waiting disabled, the restriction selector nonzero, and destination `+0x40` nonzero when read, the prepared record is passed to F-37efbe955e3e using source `+0x14`. No destination notification call occurs. The result is 1, and the preparation stores are not rolled back.

5. **Restriction permits an empty destination.** Under the same selectors, but with destination `+0x40` equal to zero when read, the destination submission path is taken and the result remains 0.

6. **Full-width selector differs from stored byte.** A completion-wait selector of 256 stores zero at record `+0x11` but still enables context preparation and the final wait. It forces destination submission regardless of the restriction selector, without the conditional destination-state read.

7. **Previously initialized context.** With completion waiting enabled and context `+0x50` nonzero, both context initialization dependencies are skipped. Source bookkeeping publication and the context handle call still occur.

8. **Final wait returns.** With completion waiting enabled, a returning final wait is followed by clearing source `+0x28`, the indirect context-handle call, and clearing source `+0x20` and `+0x24`, in that order. No record completion field is checked by this function.

## Uncertainties

- **documented_only — Synchronization:** Mutual exclusion, blocking behavior, wakeup semantics, and memory-order guarantees of the dependencies are unknown.
- **documented_only — Collection semantics:** Removal order, insertion order, capacity, and the meaning of a nonzero destination state are unknown.
- **documented_only — Ownership:** Record, payload, source, destination, and context lifetime obligations are not established.
- **documented_only — Aliasing and reentrancy:** The evidence does not establish whether objects may overlap or whether concurrent calls may safely share a source object. Optional bookkeeping writes make unsupported sharing particularly consequential.
- **documented_only — Dependency mutation:** Calls may modify aliased storage or global state. Later field reads must not be replaced conceptually with earlier values without an external preservation guarantee.
- **documented_only — Failure behavior:** No explicit dependency-error propagation, timeout, abnormal-exit cleanup, or recovery contract is visible.
- **Inferred intent:** The paired handle calls resemble synchronization operations; collection calls resemble removal and insertion; the final wait resembles completion waiting. These interpretations do not strengthen the established behavior.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping. Established claims cover visible branches, argument use, access widths, offsets, store order, call order, and explicit return-value changes.

Dependency internals, platform ABI details beyond the represented widths, concurrency guarantees, ownership rules, and runtime outcomes are not available. No runtime tests have been performed.
