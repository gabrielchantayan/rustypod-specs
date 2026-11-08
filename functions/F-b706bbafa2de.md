<!-- Trusted provenance: F-b706bbafa2de; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b706bbafa2de

## Purpose

F-b706bbafa2de replaces an association in a first object, writes a backlink into a second object, and conditionally calls a dependency with the displaced association.

The represented control flow also contains a metadata-validation and accounting tail. Entry to that tail requires the first argument to be zero, although accesses through that argument occur before the zero check. Consequently:

- With a valid, nonzero first-object address, normal execution ends after the association and backlink writes.
- With a zero first argument in an environment where the corresponding low addresses are inaccessible, an earlier access prevents normal execution from reaching the tail.
- The tail is specified conditionally. Its feasibility in the intended environment is not established.

“Association,” “backlink,” “resource,” and “accounting” describe inferred roles. The established behavior consists of the accesses, comparisons, arithmetic, and dependency calls below. Ownership transfer and resource release are not established.

## Interface

### Inputs and result

The function accepts two values represented as 32-bit integers and used as addresses.

| Input | Observed role |
|---|---|
| First-object address | Base for association fields; value written into the second object's backlink. |
| Second-object address | Replacement association; base for the backlink write and a conditional byte read. |

No richer object type, calling convention, or address validation is established. There is no explicit return value or status result.

### Object fields

All offsets below are byte offsets.

| Base | Offset | Width | Access |
|---|---:|---:|---|
| First object | `0xAE4` | 32 bits | Read the previous association, then write the second argument. |
| First object | `0xAE0` | 32 bits | Read an exclusion value only when the previous association is nonzero. |
| Second object | `0x9C` | 32 bits | Write the first argument as a backlink. |
| Second object | `0xA0` | 8 bits | Read only after the first argument has been found to be zero. |

Object sizes, alignment requirements, byte order, and permissible overlap are not supplied.

### Preconditions

For the ordinary nonzero-first-argument path, the selected association fields must support their reads and writes, and the second object's backlink location must be writable. Any selected dependency must accept the supplied arguments and return normally for subsequent operations to occur.

Zero is not treated as a safe null-object sentinel. Both object-derived writes precede the relevant later zero checks.

The tail additionally requires accessible resource metadata, an accessible context global, and accessible selected counter locations. No bounds checks establish these requirements.

## Behavior

### 1. Select whether to notify a dependency

The function first reads the 32-bit previous association at first-object offset `0xAE4`.

If that value is nonzero, it reads the 32-bit exclusion value at first-object offset `0xAE0`. If the previous association is zero, this exclusion read does not occur.

F-ff03c5cac423 is called exactly when the previous association:

- Is nonzero.
- Differs from the exclusion value.
- Differs from the second argument.

Its arguments are the captured previous association and zero.

This call precedes both object writes. If it does not return normally, those writes are not reached. No assumption is made that it preserves object contents, globals, allocation state, or object lifetime.

The decision uses the association and exclusion values already read. Neither field is reread before the subsequent stores.

### 2. Replace the association and write the backlink

After the optional dependency returns, or immediately when it is skipped, the function performs two 32-bit stores in this order:

1. Write the second argument at first-object offset `0xAE4`.
2. Write the first argument at second-object offset `0x9C`.

These stores are not an atomic transaction. A failure during the second store does not have a represented rollback of the first.

No direct operation clears a backlink in the displaced object. Such an effect inside F-ff03c5cac423 is unknown.

Aliasing is not explicitly rejected. Where overlapping accesses are valid in the execution environment, the second store follows the first and may overwrite its effects. The optional dependency can also change the state encountered by these stores.

### 3. Return for a nonzero first argument

After both stores, a nonzero first argument causes an immediate return.

On this path the function does not:

- Read the byte at second-object offset `0xA0`.
- Obtain the tail resource.
- Read the context global.
- Perform the tail's metadata validation or counter updates.

This branch depends on the argument value, not on the contents of either object after the stores.

### 4. Enter the tail conditionally

A zero first argument proceeds to a read of the byte at second-object offset `0xA0`.

- A zero byte causes return.
- A nonzero byte leads to a test of the second argument; a zero second argument causes return.
- A nonzero byte and nonzero second argument lead to F-fb5a2b34c007, with no arguments shown.

The byte read precedes the second-argument zero check. The backlink store also precedes that check. The check therefore does not protect either access from a zero second argument.

F-fb5a2b34c007 supplies a 32-bit resource-address value. A zero result causes return without further tail calls or counter updates.

For a nonzero resource result:

1. Call F-2fd8826557a4, with no arguments shown.
2. Read a 32-bit context address from an external global location.
3. Continue using the captured resource and context values.

The representation tests the captured resource for nonzero again after the context read. No intervening reassignment of that value is shown, so the repeated test adds no distinct outcome under the represented value flow.

The global is read after F-2fd8826557a4 returns. Its earlier value cannot be assumed to be the captured context.

### 5. Validate resource metadata

The resource address denotes a payload with preceding metadata:

| Relative offset | Width | Observed use |
|---|---:|---|
| `−8` | 32 bits | Header validation and masked size extraction. |
| `−4` | Unsigned 16 bits | First accounting index. |
| `−2` | Unsigned 16 bits | Second accounting index. |

Validation first reads the header and compares it with zero.

- If that read yields zero, call F-a817d8d58c10.
- Otherwise, read the header again and test its least significant bit. If that bit is set, call F-a817d8d58c10.

F-a817d8d58c10 is annotated as non-returning. Normal continuation after this call is not specified. The termination mechanism, diagnostics, and other effects are unknown.

The two tests are separate reads, not validation of a single captured header. Successful validation establishes only that the first read was nonzero and the second read had its low bit clear. It does not establish that both reads observed the same value. In particular, a second read of zero would pass the low-bit test if the first read had been nonzero.

No validation of metadata bounds, payload lifetime, or accounting indices is shown.

### 6. Update accounting state

After validation, call F-c0fc6d033caa with the captured context address. Metadata used afterward is read after that call returns; the dependency is not assumed to preserve it.

Subsequent operations occur in the following sequence:

1. Read the first unsigned 16-bit metadata index at resource offset `−4`.
2. Read the header at resource offset `−8` and mask it with `0x3FFFFFFC`. Capture this masked size for the next four subtractions.
3. Read and replace the 32-bit counter at context offset `0x224 + 4 × first index`, subtracting the captured size.
4. Read and replace the 32-bit counter at context offset `0x30C`, subtracting the same size.
5. Read the second unsigned 16-bit metadata index at resource offset `−2`.
6. Read and replace the 32-bit counter at context offset `0xE0 + 4 × second index`, subtracting the same size.
7. Read and replace the 32-bit counter at context offset `0x220`, subtracting the same size.
8. Call F-09dcb55e8e0a with no arguments shown and capture its 32-bit result as another index.
9. Read and replace the 32-bit counter at context offset `0x310 + 4 × returned index`, decrementing it by one.
10. Read and replace the 32-bit counter at context offset `0x390`, decrementing it by one.
11. Read the 32-bit counter at context offset `0x08` and read the resource header again. Subtract the newly masked header value, using `0x3FFFFFFC`, from the counter value and store the result at context offset `0x08`. **Both reads belong to this subtraction after the preceding decrement; their relative evaluation order is unspecified by the supplied representation.**
12. Call F-cc6c9b4a0f68 with the captured context address and the header address at resource offset `−8`.
13. If that call returns, call F-e586bff64fb7 with the captured context address.
14. Return if the final call returns.

The context address is captured once. Later changes to the global do not select another context for these operations.

The final size subtraction uses a fresh header read after F-09dcb55e8e0a, not the size captured for the first four subtractions. Those sizes can differ. The earlier validation is not repeated for either size read.

The seven counter updates need not address seven distinct locations. Indexed locations can coincide with fixed locations or with each other. In that case, the ordered updates accumulate on the shared location, subject to any other permitted mutations.

No direct metadata or payload access is shown after F-cc6c9b4a0f68 returns. That is compatible with resource consumption by the dependency, but does not establish it.

### 7. Widths and arithmetic

Both metadata indices are unsigned 16-bit values, widened before multiplication by four. Each scaled index ranges from zero through `0x3FFFC`, before addition of the context address and fixed offset.

Masking a 32-bit header with `0x3FFFFFFC` clears its lowest two and highest two bits. The result is a multiple of four from zero through `0x3FFFFFFC`. A nonzero, even header can pass stable validation while yielding size zero.

The size subtractions use unsigned 32-bit arithmetic in the supplied representation. Their stored results are modulo `2³²`; subtracting four from zero produces `0xFFFFFFFC`. There is no saturation or underflow check.

The two count decrements are represented using signed 32-bit arithmetic. Ordinary in-range decrements are established, including zero becoming negative one. Behavior at signed overflow is not established as modular merely because the stored location is 32 bits.

F-09dcb55e8e0a supplies a represented signed 32-bit index with no shown bounds check. Its multiplication by four and the address calculations have no demonstrated overflow protection. Negative indices, signed arithmetic overflow, and invalid computed addresses are not assigned a safe fallback behavior.

## Side effects

Directly represented effects include:

- The optional displaced-association dependency call.
- Two ordered object stores.
- Conditional resource-metadata and context-global reads.
- Up to seven ordered counter updates, potentially to overlapping locations.
- Conditional calls to validation, context, indexing, and resource-related dependencies.

No explicit allocation or directly identified deallocation operation appears. Ownership transfer, reference counting, lock acquisition, lock release, and reclamation remain unknown.

There is no explicit error result, exception handler, transaction boundary, or rollback. A failed access or dependency that does not return may leave earlier stores and accounting changes visible.

## Dependencies

| Opaque ID | Observed interface and placement | Unestablished behavior |
|---|---|---|
| F-ff03c5cac423 | Receives previous association and zero before object stores, when selected. | Detachment, release, ownership, and preservation of other state. |
| F-fb5a2b34c007 | No arguments shown; produces the tail resource address. | Allocation, borrowing, validity, and lifetime of the result. |
| F-2fd8826557a4 | No arguments shown; called before capturing the context global. | Global mutation, initialization, and synchronization. |
| F-a817d8d58c10 | No arguments shown; called on a failed header test; annotated non-returning. | Exact failure mechanism and diagnostics. |
| F-c0fc6d033caa | Receives context before metadata-index reads and accounting changes. | Locking, metadata preservation, and context validity. |
| F-09dcb55e8e0a | No arguments shown; supplies an index between size updates and count decrements. | Index range, sign constraints, and mutation of metadata or counters. |
| F-cc6c9b4a0f68 | Receives context and header address after accounting updates. | Whether it consumes, frees, retains, or modifies memory. |
| F-e586bff64fb7 | Receives context after the preceding dependency returns. | Unlocking, finalization, and preservation of globals or memory. |

## Acceptance examples

These are conditional expectations from static evidence, not executed tests. Unless stated otherwise, required accesses are valid, dependencies return normally, and relevant memory remains stable.

1. **No previous association.** With a nonzero first argument and zero previous association, the exclusion field is not read and F-ff03c5cac423 is skipped. The association store precedes the backlink store, followed by return.

2. **Excluded previous association.** A nonzero previous association equal to the exclusion value skips notification even if it differs from the replacement. Both stores still occur.

3. **Replacement already selected.** A previous association equal to the second argument skips notification. Equality does not remove either store.

4. **Distinct displaced association.** A nonzero previous association differing from both comparison values is passed to F-ff03c5cac423 with zero. Neither object store is reached unless that call returns.

5. **Unreadable flag on the ordinary path.** With a nonzero first argument and successful earlier operations, an inaccessible byte at second-object offset `0xA0` is irrelevant because it is not read.

6. **Zero-valued arguments.** No graceful null handling is promised. The first-object association access precedes the first-argument zero check; the second-object backlink store precedes its zero check.

7. **Tail resource absent.** If the environment permits the earlier zero-first-argument accesses, the second argument and flag are nonzero, and F-fb5a2b34c007 returns zero, execution returns without F-2fd8826557a4 or accounting updates.

8. **Invalid stable header.** Under reachable-tail assumptions, a stable zero header or stable odd header selects F-a817d8d58c10 before F-c0fc6d033caa and before accounting writes.

9. **Validation reads differ.** If the first validation read is nonzero and the second is zero, neither validation condition selects failure. This does not guarantee that later metadata accesses are valid.

10. **Zero accounted size.** A stable header of `0x40000000` passes validation and masks to zero. Size updates subtract zero, while the two count decrements still occur if reached.

11. **Header changes during indexing.** If the captured size is `16`, and F-09dcb55e8e0a changes the header so the final masked size is `32`, the first four size updates subtract `16` and the final size update subtracts `32`. This does not specify whether the final counter read precedes or follows the final header read.

12. **Counter aliasing.** If the index returned by F-09dcb55e8e0a is `32`, its selected offset is `0x390`. With a valid context and an initial counter value of five at that offset, the two consecutive count decrements leave three.

## Uncertainties

The following remain **documented_only**:

- Whether the zero-first-argument tail is reachable in the intended runtime or reflects an inaccurate recovered condition or type.
- Actual object types, extents, alignment, byte order, and address-space rules.
- The identity, lifetime, and synchronization discipline of the context global.
- Dependency effects on globals, metadata, counters, object lifetime, and ownership.
- Valid index ranges and handling of invalid computed addresses.
- Intended treatment of signed arithmetic overflow.
- Whether unsigned accounting wraparound is legitimate or indicates corrupted state.
- Failure behavior for inaccessible memory and the exact non-returning dependency behavior.
- Concurrent-access guarantees and whether external mutation is permitted.
- The relative order of the final counter and header reads. Exact machine-level ordering requires additional evidence.

The contract neither repairs the unusual branch structure nor supplies missing null guards.

## Evidence

The supplied static representation and opaque dependency mapping are the sole evidence.

Established facts include represented access widths and offsets, branch conditions, dependency arguments, distinct metadata reads, masks, and statement-level sequencing where supported. The final subtraction contains two operand reads whose relative evaluation order is not specified; no exact machine-level ordering is asserted for them.

Association, accounting, and resource-release interpretations are inferred intent, not independently established subsystem guarantees.

No dependency implementations were inspected. No runtime tests, executions, or dynamic observations have been performed.
