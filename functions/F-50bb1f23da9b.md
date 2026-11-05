<!-- Trusted provenance: F-50bb1f23da9b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-50bb1f23da9b

## Purpose

F-50bb1f23da9b searches a context-associated collection for the first examined record whose one-byte discriminator equals a supplied 32-bit unsigned selector. On a match, it invokes a transfer dependency with the caller’s destination, the matched record address, and a length of 12 bytes. If no match is found, it writes a partial default result to the destination. After either result path, it conditionally invokes an indirect operation on the acquired context.

This describes the behavior established by the supplied static evidence. Interpreting the transfer as a byte copy and the final indirect operation as reference release is plausible intent, not an established dependency contract.

## Interface

### Inputs

The recovered interface has three arguments:

| Argument | Recovered representation | Observable use |
|---|---|---|
| First argument | 32-bit value | Not referenced by this function’s visible body. |
| Destination | Pointer to 32-bit storage | Receives either a requested 12-byte transfer or three explicit fallback writes. |
| Selector | Unsigned 32-bit value | Compared with a zero-extended one-byte discriminator. |

The first argument has no established semantic purpose. Its lack of direct use does not establish whether it has an ABI-level role.

The destination is caller-provided storage. The function does not visibly allocate it, return a replacement address, or establish ownership of it. For the matched path, the destination must satisfy the transfer dependency’s requirements for a 12-byte operation. For the fallback path, it must support the specified writes at byte offsets 0, 4, and 8. A uniformly usable destination should therefore provide at least 12 writable bytes, with alignment sufficient for the two 32-bit writes and any additional requirements imposed by the transfer dependency.

### Result

No explicit return value is established. The intended result is communicated through destination storage and dependency side effects.

There is no separate status distinguishing a matched record from a fallback result. In particular, the fallback discriminator value of 6 is data written to the destination, not an independently returned error code.

### Recovered layout model

The visible operations use a 32-bit address and word model:

- The context contains a collection-related region beginning at byte offset 20.
- Each lookup result contains a 32-bit record address at byte offset 8.
- Each referenced record has a one-byte discriminator at byte offset 8.
- The context’s first word supplies an indirect-dispatch table address.
- The final dispatch uses the callable entry at byte offset 4 within that table.

These are access requirements, not complete type definitions. No other field meanings, record invariants, or packing rules are established.

## Behavior

### Context acquisition and initial collection query

The function first calls F-73d4385c28ec without visible arguments and retains its result as the context address.

It then calls F-ef3b831bc76c with the address 20 bytes beyond that context. This occurs before any null check on the context. Consequently, a null acquisition result is not an explicitly supported “empty collection” case. The later null guard does not protect this earlier operation.

The first query result is tested only for being zero or nonzero:

- Zero selects the fallback path immediately.
- Any nonzero 32-bit result enters the search setup.

The initial result is not retained as a fixed search bound. A separate query is made before the first candidate lookup and before each subsequent iteration.

### Search-bound evaluation

The search counter starts at unsigned 32-bit zero. Before each candidate lookup, F-ef3b831bc76c is called again with the same context-relative address.

The current counter is then compared with that query result using unsigned 32-bit ordering. A lookup occurs only when the counter is strictly less than the newly obtained bound.

Observable consequences include:

- A nonzero initial query can be followed by a zero query, producing no lookup.
- The bound is re-evaluated rather than snapshotted.
- Search length may change if the dependency returns different values over time.
- A terminating bound query is still an observable call, even though it does not produce a lookup.

If an iteration completes without a match, the counter advances by one using unsigned 32-bit arithmetic. The visible increment has no saturation check. However, with an ordinary maximum unsigned bound, the counter reaches that maximum and fails the strict comparison before another lookup; wraparound is not needed to explain the normal search sequence.

### Candidate lookup and discriminator comparison

For each admitted iteration, the counter is narrowed to a signed 16-bit value and then widened to the lookup’s integer argument representation. F-0ff4e60503e7 receives:

1. The retained context address.
2. That signed, narrowed index.

Under the recovered 16-bit narrowing model, index 32,767 remains positive, index 32,768 becomes −32,768, and index 65,536 maps to zero. Thus the lookup index is not necessarily the same numeric value as the unsigned search counter.

No visible restriction keeps the bound within the nonnegative signed-16-bit index range. Correctness for larger collections depends on the lookup dependency’s handling of the resulting signed values.

After lookup returns, the function:

1. Reads the 32-bit record address at byte offset 8 of the lookup result.
2. Reads the discriminator byte at byte offset 8 of that record.
3. Zero-extends the byte and compares it with the full 32-bit unsigned selector.

The selector is not narrowed to a byte. Values above 255 cannot match any discriminator through this comparison.

Neither the lookup result nor the referenced record address is checked for null before these reads. Validity of both is a caller/dependency precondition for defined execution of the visible path.

### Matched result

The first examined candidate with an equal discriminator selects the matched path. There are no later collection-bound queries or candidate lookups after that match.

Before invoking the transfer dependency, the function reads the record-address field from the lookup result again. It passes:

- The caller’s destination.
- The record address obtained by this second field read.
- The byte length 12.

The repeated field read matters to the observable access contract. The function does not explicitly preserve the record address used for the discriminator read as the transfer source. In stable, ordinary memory the addresses agree; a guarantee under concurrent or externally changing memory is not established.

The transfer dependency’s return value, if any, is ignored. No fallback writes occur merely because that dependency reports a failure through a return value.

The visible call shape strongly suggests a 12-byte memory transfer, but the dependency implementation is absent. Exact copy behavior, overlap support, alignment requirements, and failure behavior remain dependency-defined.

### Fallback result

Fallback occurs if the initial collection query is zero or if the search terminates without a matching discriminator.

The function performs these writes in order:

| Destination byte offset | Width | Written value |
|---|---:|---:|
| 0 | 32 bits | 0 |
| 4 | 32 bits | 0 |
| 8 | 8 bits | 6 |

There is no explicit write to destination offsets 9 through 11 on this path. The function therefore does not visibly initialize all 12 bytes of a presumed record.

Immediately after these direct writes, and absent other agents or dependency side effects, the first eight destination bytes are zero, byte 8 is 6, and bytes 9–11 retain their previous contents.

### Final indirect operation

After either result path, the retained context address is checked against null.

If it is nonnull, the function reads the context’s current first word as a dispatch-table address, obtains the callable entry at table byte offset 4, and invokes that entry with the context as its visible argument.

The dispatch information is read at this late stage, after result handling. It is not cached when the context is acquired. Any earlier dependency that changes this state can therefore affect the final dispatch.

If the retained context is null and execution nevertheless reaches this point, the indirect operation is skipped. This conditional skip does not make the earlier context-relative operations null-safe.

The final call’s return value, if any, is not used.

## Side effects

Established direct destination effects are limited to the three fallback writes. On a match, destination modification is delegated to the transfer dependency.

Other observable effects include:

- One context-acquisition call.
- One initial collection query.
- Additional collection queries as search conditions are evaluated.
- One lookup per admitted candidate.
- Candidate-address and discriminator reads in the stated order.
- A requested 12-byte transfer on the first match.
- A conditional final indirect call after result handling.

No explicit allocation, destination deallocation, collection mutation, logging, or error reporting appears in the visible body. Dependencies may perform such operations; their effects are unknown.

Ownership is not established by the call pattern. In particular:

- The acquired context is used through the final indirect call, but that call is not proven to release it.
- Lookup results and record addresses are not visibly freed separately.
- The transfer dependency is not proven to preserve or consume its source.
- Dependencies are not assumed to preserve globals or aliased memory.

There are no visible synchronization operations or explicit alias checks. Destination overlap with context, lookup-result storage, or record storage can affect later accesses or dependency behavior. Such overlap must not be presumed supported.

## Dependencies

| Opaque identifier | Visible role | Established boundary |
|---|---|---|
| F-73d4385c28ec | Context acquisition | Called once without visible arguments; result is used as a context address. |
| F-ef3b831bc76c | Collection-bound query | Receives context address plus 20 bytes; initial result is tested for nonzero, later results serve as unsigned bounds. |
| F-0ff4e60503e7 | Candidate lookup | Receives context and a signed-16-bit-narrowed index; result must support the record-address read at offset 8. |
| D-01 | Unmapped transfer dependency | Receives destination, reread record address, and length 12. |
| D-02 | Runtime-selected indirect dependency | Selected through the context’s first word and table offset 4; receives the context. |

D-01 and D-02 are document-local opaque labels, not supplied provenance identifiers.

## Acceptance examples

These are conditional static acceptance scenarios, not executed tests.

### Empty initial query

Given a usable context and an initial collection query of zero:

- No candidate lookup occurs.
- D-01 is not called.
- The destination receives the two zero words and discriminator byte 6, in that order.
- D-02 is invoked afterward if the retained context is nonnull.

If destination bytes 9–11 initially contain distinct sentinel values, the direct fallback writes leave them unchanged. Their values at function completion additionally depend on D-02 and any aliasing.

### First candidate matches

Given a nonzero initial query, a subsequent bound greater than zero, and a valid first candidate whose discriminator equals the selector:

- F-0ff4e60503e7 receives index zero.
- The record-address field is read for comparison and reread for transfer.
- D-01 receives length 12.
- No second candidate is queried.
- No fallback writes occur.

If D-01 is independently established as a conventional byte-copy operation, the destination receives those 12 source bytes before final dispatch.

### Bound shrinks before lookup

Given an initial query of three followed by a search-condition query of zero:

- No lookup occurs.
- Fallback is written.
- The initial value of three does not force any iteration.

### Selector outside byte range

Given selector 256 and a finite, stable search that terminates normally:

- No discriminator can compare equal.
- Each admitted candidate is examined.
- Fallback is selected after the terminating bound query.

A discriminator of zero does not match selector 256.

### Signed index boundary

Given bounds admitting counters 32,767 and 32,768:

- The corresponding lookup arguments are 32,767 and −32,768 under the recovered narrowing model.
- The function does not reject or correct the negative lookup argument.
- Continued behavior depends on F-0ff4e60503e7 accepting that argument and returning valid storage.

### Duplicate matching discriminators

Given multiple examined candidates with matching discriminators, the first match in the actual lookup sequence triggers D-01. Later candidates are not examined. This is an ordering guarantee over lookup calls, not a guarantee about an independently defined collection order.

## Uncertainties

The following are **documented_only**:

- The semantic purpose of the unused first argument.
- The concrete context, collection, lookup-result, and record types.
- Whether collection queries are stable, pure, or synchronized.
- Whether negative or repeated narrowed indices are valid.
- D-01’s actual transfer semantics and overlap behavior.
- D-02’s identity, ownership effects, and relationship to destruction or reference counting.
- Allocation, global-state changes, and memory ownership inside dependencies.
- Defined behavior for null, invalid, misaligned, or concurrently modified addresses.
- Exception, fault, or nonlocal-exit behavior. No recovery path is visible.
- Whether destination contents remain unchanged after result handling; final dispatch can have unknown side effects.

Likely intent is record selection with a default result followed by context release. That interpretation is not substituted for the established access and call behavior.

## Evidence

This contract is based solely on the supplied static function body and dependency mapping.

Static evidence establishes the branch structure, direct read/write widths and offsets, repeated bound queries, unsigned comparison, signed-16-bit index conversion, full-width selector comparison, transfer-call arguments, partial fallback initialization, and final indirect-dispatch sequence.

Dependency implementations and runtime observations were not supplied. No runtime tests have been performed. No dependency purity, ownership transfer, successful copying, or cleanup semantics are claimed beyond the visible call boundaries.
