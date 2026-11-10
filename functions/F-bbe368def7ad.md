<!-- Trusted provenance: F-bbe368def7ad; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bbe368def7ad

## Purpose

F-bbe368def7ad selects an object provider using a descriptor byte, initializes the resulting object, obtains metadata, conditionally establishes shared resources, and attempts to create an associated per-context object.

On the final success path, it stores one descriptor word in the context, writes an active/status byte, invokes an external dependency with a context-relative destination address, the descriptor address, and `0x168`, then returns zero if that dependency returns normally. The external dependency’s implementation is unavailable: neither a completed descriptor snapshot nor preservation of the status byte is established.

Failure is not transactional. Object references, shared fields, and per-context fields may already have changed when failure is reported. Ownership and cleanup effects remain dependency-defined.

These are static behavioral findings. Interpreting the objects and shared resources as media-processing infrastructure is inferred intent, not part of the established contract.

## Interface

### Arguments and representation

The function accepts two addresses:

- **Context:** storage used for object references, a status byte, a separate descriptor word, and an address passed to an external dependency.
- **Descriptor:** storage containing the initial selection byte and, on the final success path, a directly read 32-bit word.

The supplied representation uses 32-bit addresses and four-byte integer fields. The result is a signed 32-bit integer.

### Context layout

| Byte offset | Width | Established use |
|---|---:|---|
| 0 | 4 bytes | Provider result; subsequently read to dispatch metadata acquisition |
| 4 | 4 bytes | Existing associated-object check; destination for the constructor-like result |
| 12 | 1 byte | Written as zero before initialization and as one on final success |
| 24 | Not established | Address passed as the first argument to the final external dependency |
| 384 | 4 bytes | Destination for the descriptor word read at offset 36 |

There is no established direct write to the range 24–383. The final dependency receives the address at offset 24 and the value `0x168`, but its memory-access extent is unknown.

### Descriptor requirements

The visible body directly requires:

- A readable byte at offset 1 for provider selection, potentially read twice.
- A readable four-byte word at offset 36 if the final success path is reached.

It does **not** establish a requirement that all 360 descriptor bytes be readable merely because `0x168` is passed to the final dependency. That dependency, the initialization method, and other dependencies may impose additional requirements unavailable here.

On the final success path, direct context accesses extend through byte 387. A conventional contiguous context allocation therefore needs at least 388 bytes for those direct accesses alone. This bound does not cover dependency accesses.

### Preconditions and limitations

No visible null, bounds, alignment, or object-validity checks precede the corresponding accesses. Defined execution requires valid storage and callable dispatch entries for every reached operation.

In particular:

- The selected provider must supply a usable object address.
- Dispatch-table entries at byte offsets 16 and 68 must be callable when reached.
- Shared-state addresses used by the function must support the reached field accesses.
- Metadata outputs consumed by the function must have usable values.
- The capacity output used after construction must have a defined value.

Argument aliasing and aliasing with shared state are not checked. Writes or dependency effects may consequently change later reads. No general alias-safe behavior is established.

### Results

| Result | Established condition |
|---|---|
| 0 | Construction returned nonzero, the unsigned capacity comparison passed, and the final external dependency returned normally |
| −1 | An explicit rejection or failure path was reached |
| Earlier initialization result | Metadata format was neither 8 nor 9, and the per-context field at offset 4 was zero when checked |
| No normal return | The represented non-returning dependency was reached, or another dependency did not return |

The propagated initialization result is known only to differ from −1. It can itself be zero, so a zero result alone does not prove that construction or the final external invocation occurred.

## Behavior

### 1. Entry and provider selection

F-02b5cfbe6749 is called first, with no explicit argument shown. Descriptor inspection follows that call.

The function reads descriptor byte 1:

- If it equals 8, F-362dfede5dca supplies the object.
- Otherwise, the function reads the byte again. If this second value equals 9, F-10436c42da2e supplies the object.
- Otherwise, the function returns −1.

The two reads are distinct in the representation. An intervening mutation is not specified; stable-descriptor examples assume both reads observe the same byte.

For a supported selector, the function:

1. Stores the provider result at context offset 0.
2. Writes zero to context offset 12.
3. Calls F-D01 on the provider result, passing that object and the descriptor.

F-D01 is the dynamically dispatched initialization method at dispatch-table byte offset 16.

If its return value is exactly −1, the function writes zero to context offset 0 and returns −1. There is no visible release call on this path. Other initialization results continue, including nonzero values other than −1.

The status-byte write precedes initialization. Its value after initialization is not guaranteed if that dependency can mutate the context.

### 2. Metadata acquisition

The function reloads the object address from context offset 0 and invokes F-D02, the dynamically dispatched method at dispatch-table byte offset 68.

F-D02 receives the object and fourteen output addresses, each referring to four-byte local storage. Four outputs are subsequently consumed:

- An unsigned 32-bit first dimension, called **width** below.
- An unsigned 32-bit second dimension, called **height**.
- A 32-bit comparison value.
- A four-byte format output, of which only the first byte is inspected.

The other ten output locations are not subsequently read by the visible body. The return value of F-D02 is not checked.

“Width,” “height,” and “capacity” describe local usage. Their external units and broader interpretation are not established.

### 3. Shared-state references and reuse guard

Immediately after metadata acquisition, the representation makes two ordered references to the shared-state global:

1. It reads the global address and saves that address locally.
2. It separately references the global again to obtain the address used for the initial offset-12 guard.

No dependency call occurs between these references. The representation does not establish volatile access or concurrent-mutation semantics; it therefore does not establish whether they can observe different addresses in a particular execution environment.

If the four-byte field at offset 12 through the **second** global reference is nonzero, shared-resource initialization and validation are skipped. Execution proceeds directly to the per-context check.

All subsequent shared-field accesses use the **saved** address from the first reference. This includes the backing value later passed to construction. The initial guard therefore must not be described as a read solely through the saved address.

A nonzero reuse guard does not establish that shared offset 8 is valid, nor does the function check that field on this bypass path.

### 4. Shared-resource initialization

When the initial guard is zero, the function initializes a local unsigned 32-bit capacity:

- Metadata format byte 8: capacity 9.
- Any other format byte: capacity 5.

Unsigned dimension comparisons select a fixed allocation request:

| Dimensions | Requested size | Capacity effect |
|---|---:|---|
| Width ≤ 320 and height ≤ 240 | Selected by the small-dimension policy | See below |
| Width ≤ 640 and height ≤ 480, outside the smaller class | `0x500000` | No further change |
| Width > 640 or height > 480 | `0x680000` | Set to 8 after storing the allocation result |

The boundaries are inclusive. Zero dimensions enter the smallest class. High-bit-set dimension values are large unsigned values, not negative values.

For the smallest class, F-c0c0b8633623 is called, followed by F-bc4e8f603b51:

- A zero query result selects `0x180000`.
- A nonzero result selects `0x100000` and sets capacity to 6 after storing the allocation result.

Every backing request calls F-49d04c9bba7a with the selected size and a fixed diagnostic-label argument. Its result is stored at saved shared-state offset 12, including a zero result.

The four sizes are 1,572,864, 5,242,880, 6,815,744, and 1,048,576 bytes respectively. They are constants; the function does not multiply dimensions to derive them.

After the allocation-result store, the function reads saved shared offset 12. If it is nonzero, that field is read again for a call to F-0825e0f86333 with these arguments:

1. `0x40000`
2. The backing value from shared offset 12
3. 2
4. 1

The returned four-byte value is stored at saved shared offset 8.

The function then reads shared offset 8:

- Nonzero permits continuation.
- Zero causes a read of shared offset 12.
  - If offset 12 is also zero, return −1.
  - Otherwise, call F-da23f49bf314, then F-feee9156a257. The latter is represented as non-returning.

These are field reads at their respective positions, not permanent conclusions about earlier dependency results. For example, a zero backing result does not force immediate failure: if shared offset 8 remains nonzero, this path continues without calling F-0825e0f86333.

### 5. Per-context construction

After either shared-resource continuation or the reuse bypass, the function reads context offset 4.

If it is nonzero, the function returns −1. This check occurs after initialization, metadata acquisition, and any reached shared-resource work.

If it is zero, metadata format controls construction:

| Metadata format byte | Mode argument | Format argument |
|---|---:|---:|
| 8 | 0 | 4 |
| 9 | 1 | 5 |

For either supported format, the function calls F-8157a180d901 with `0x54`, an 84-byte request. It then reads saved shared offset 12 and calls F-72c8804e33aa with:

1. The 84-byte request result.
2. The address of the local capacity.
3. The selected mode.
4. Height.
5. Width.
6. Width again.
7. The selected format argument.
8. The backing value read after the 84-byte request.

There is no intervening zero-result check on the 84-byte request. F-72c8804e33aa may modify capacity through the supplied address.

Its return value is stored at context offset 4 before success is evaluated.

If metadata format is neither 8 nor 9, neither construction dependency is called. The function returns the earlier initialization result unchanged. The initial descriptor selector and metadata format need not agree.

### 6. Final decision and external invocation

The final success decision requires:

- A nonzero F-72c8804e33aa result.
- The comparison value plus one, evaluated with unsigned 32-bit arithmetic, to be no greater than capacity after that call.

The addition wraps modulo \(2^{32}\). A comparison-value bit pattern of `0xFFFFFFFF` becomes zero. No separate overflow rejection is visible.

If either condition fails, F-02b5cfbe6749 is called with the context address, followed by a return of −1. The function does not explicitly clear the constructor-result store before this call.

If both conditions pass, the established order is:

1. Read the four-byte descriptor word at offset 36.
2. Store it at context offset 384.
3. Write one to context offset 12.
4. Invoke F-D03 with destination address context+24, source address descriptor, and third argument `0x168`.
5. Ignore F-D03’s return value and return zero, provided F-D03 returns normally.

The argument names “destination” and “source” identify their apparent roles only. Copying, access extent, overlap handling, and preservation of prior context writes are not established.

## Side effects

Direct writes by the visible body can affect:

- Context offset 0.
- Context offset 4.
- Context offset 12.
- Context offsets 384–387.
- Saved shared-state offsets 8 and 12.
- The local capacity value.

The body passes context+24 to F-D03 but does not itself establish writes to a 360-byte region there.

Dependencies may alter context fields, descriptor contents, globals, shared fields, or memory reached through supplied addresses. In particular, after F-D03 returns, neither the status byte remaining one nor the separate descriptor-word store remaining intact is guaranteed by the available evidence.

Ownership is unresolved. No supplied contract establishes that provider results, backing allocations, or the 84-byte request result are transferred, retained, or released. F-da23f49bf314 cannot be assumed to release backing memory, and F-02b5cfbe6749 cannot be assumed to restore the context.

There is no visible transactional rollback or synchronization mechanism.

## Dependencies

F-D01, F-D02, and F-D03 are document-local opaque labels for dependencies without supplied function IDs; they do not assert independently established identities.

| Opaque ID | Established invocation or role |
|---|---|
| F-02b5cfbe6749 | Entry call without an explicit argument; final construction-failure call with context |
| F-362dfede5dca | Provider selected by descriptor byte 8 |
| F-10436c42da2e | Provider selected by descriptor byte 9 on the second selection read |
| F-D01 | Initialization dispatch at byte offset 16 |
| F-D02 | Metadata dispatch at byte offset 68, with fourteen output addresses |
| F-c0c0b8633623 | Precedes the small-dimension policy query |
| F-bc4e8f603b51 | Zero/nonzero result selects the small-dimension allocation policy |
| F-49d04c9bba7a | Backing request with fixed diagnostic label |
| F-0825e0f86333 | Receives `0x40000`, backing value, 2, and 1 |
| F-da23f49bf314 | Called on the nonzero-backing/zero-associated-resource failure path |
| F-feee9156a257 | Represented as non-returning after that call |
| F-8157a180d901 | Receives request value 84 |
| F-72c8804e33aa | Constructor-like call with writable capacity address |
| F-D03 | Final external invocation with context+24, descriptor, and `0x168`; return ignored |

## Acceptance examples

These are conditional static expectations, not executed tests. Except where stated otherwise, dependencies return normally, supply valid outputs, and introduce no unmentioned mutations.

1. **Unsupported selector:** With descriptor byte 1 stably equal to 7, the entry dependency runs, then the function returns −1 without a direct context write.

2. **Initialization rejection:** If a selected object’s initialization returns −1, the function writes zero to context offset 0 and returns −1. The zero status-byte write occurred before initialization; preservation of that byte requires the dependency not to change it.

3. **Small boundary:** Dimensions 320 × 240 enter the smallest class. Query result zero selects `0x180000`; nonzero selects `0x100000` and capacity 6.

4. **Crossing one boundary:** Dimensions 321 × 1 select `0x500000`. Dimensions 1 × 481 select `0x680000` and capacity 8.

5. **Reused backing:** A nonzero initial guard bypasses allocation and the offset-8 check. No capacity initialization occurs locally. A defined later capacity comparison requires an external guarantee, such as construction writing capacity.

6. **Selector/metadata mismatch:** Descriptor selector 8 with metadata format 9 selects constructor mode 1 and format argument 5. No mismatch rejection occurs.

7. **Capacity boundary:** A nonzero constructor result, comparison value 8, and final capacity 9 pass. Comparison value 9 with capacity still 9 fails and reaches the context-taking failure dependency.

8. **Unsigned wraparound:** Comparison value `0xFFFFFFFF` increments to zero. With a nonzero constructor result and a defined unsigned capacity, the capacity condition passes even for capacity zero.

9. **Unsupported metadata format:** If initialization returned 3, shared handling permits continuation, context offset 4 is zero, and metadata format is unsupported, the result is 3. No constructor or final external invocation occurs. An initialization result of zero would instead propagate zero without those operations.

10. **Final invocation is not a snapshot guarantee:** After a passing construction decision, the direct word store and status-byte write precede F-D03 with the exact three arguments described above. If F-D03 returns normally, the function returns zero regardless of its return value. The available evidence does not determine the resulting bytes at context+24 or the final status-byte value.

## Uncertainties

The following are **documented_only**:

- All dependency implementations, ownership rules, memory-access extents, and preservation guarantees.
- F-D03’s copying behavior, meaning of `0x168`, overlap semantics, and mutations.
- The entry signature of F-02b5cfbe6749.
- Full metadata semantics and whether every consumed output is initialized.
- Validity and consistency rules for shared offsets 8 and 12.
- Capacity on the shared-reuse path unless a dependency establishes it before consumption.
- Whether the two ordered global references can observe different values; no volatile or concurrency model is supplied.
- Behavior following an invalid or zero provider result.
- Native byte order and alignment requirements.
- Concrete terminal behavior of F-feee9156a257.

## Evidence

This contract uses only the supplied static representation and dependency-ID mapping. Established evidence covers direct field accesses, widths, dispatch offsets, ordered operations, fixed arguments, unsigned comparisons, stores, and return paths.

Allocation, construction, and media-related terminology describe inferred roles rather than proven dependency contracts. In particular, the final external invocation is not evidence of a completed copy.

No runtime tests have been performed. The acceptance examples express conditional consequences of static control flow, not observed executions.
