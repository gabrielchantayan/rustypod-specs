<!-- Trusted provenance: F-eb288a1ea3b0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-eb288a1ea3b0

## Purpose

F-eb288a1ea3b0 conditionally updates a shared target from a source associated with a caller-supplied object, then writes a final target-derived value to a caller-supplied output location.

The established behavior includes:

- A preliminary global-state-dependent sequence.
- A source-availability check.
- Comparison of a queried 64-bit value with an object-resident cache.
- Resolution of changed source data into a 32-bit data value and a 32-bit extent.
- One of two target-update paths, selected by another source query.
- A fallback target operation when the source is unavailable or resolution produces a zero extent.
- A final query and one 32-bit output write on every normally returning path.

The 64-bit value appears to serve as a change token. The shared target operations appear to manage some form of loaded content. These are inferred roles, not established domain semantics.

## Interface

### Arguments and result

| Argument | Established use |
|---|---|
| Object base | Address used to access an embedded source at byte offset `0x38` and a cached 64-bit value at byte offset `0x728`. |
| Output address | Destination for the final 32-bit result. It is also copied into a local slot that may later be passed by address to a dependency. |
| Third argument | A 32-bit value copied into a local extent slot. Every path that subsequently consumes that slot first replaces its value. |
| Fourth argument | A 32-bit value copied into a local data slot. Every path that subsequently consumes that slot first replaces its value. |

There is no explicit returned status or value. The caller-visible result is a 32-bit write through the original output address, together with object, target, and dependency side effects.

The third and fourth arguments have no established influence on subsequent dependency arguments or branch decisions. Their local copies are reset to zero before use in source resolution.

### Relevant storage

The object has the following directly accessed regions:

| Byte offset | Width | Use |
|---|---:|---|
| `0x38` | Not established | Address of an embedded source passed to dependencies. |
| `0x728` | 64 bits | Cached source value. |
| `0x728` | 32 bits | First zero store when the source is unavailable. |
| `0x72c` | 32 bits | Second zero store when the source is unavailable. |

The two 32-bit zero stores collectively clear the cached region. They are separate, ordered stores; atomic clearing of the full 64-bit region is not established.

Other participating state consists of:

- A global control-object reference, with a 32-bit flags field at offset `0x20`.
- A global reference used by the preliminary gate and completion operations.
- A global target reference used by most target operations.
- Two global values supplied to the preliminary target operation.
- The address of a fixed fallback datum.

The representation and initialization of these globals are not supplied.

### Preconditions

For defined normal execution:

- Every directly accessed object and global location must be readable or writable as appropriate.
- The output address must permit a 32-bit write.
- The target must support the indirect operation at table offset `0x0c` when that path is reached.
- Dependency-specific pointer, extent, alignment, and lifetime requirements must hold.

The function performs no explicit null, bounds, alignment, allocation-success, or data-range validation. This contract does not add such validation.

The evidence uses 32-bit words and address-bearing values, with a 64-bit queried and cached value. It does not establish a portable ABI beyond those observed widths.

## Behavior

### 1. Preliminary conditional sequence

Before inspecting the embedded source, the function reads the 32-bit flags field at offset `0x20` of the global control object.

If bit zero is set, the preliminary sequence is skipped. Other flag bits do not affect this decision.

If bit zero is clear, the function calls F-b255b26284e6 with the global gate reference. A zero result skips the remaining preliminary operations. Any nonzero result causes these operations, in order:

1. Call F-5b24edfb22c0 with the global target reference.
2. Pass its 32-bit result, followed by two global values, to F-804adc29bbfb.
3. Call F-d08d3ca17b81 with the global gate reference.

Only normal sequential completion is established. There is no visible cleanup mechanism ensuring the third operation occurs if either preceding operation fails to return normally.

This sequence precedes the source check and therefore still occurs when the source is unavailable or its queried value later matches the cache.

### 2. Source availability and cache handling

The function calls F-80f9e436da7e with the address at object offset `0x38`.

#### Zero result: unavailable-source path

The function:

1. Stores a 32-bit zero at object offset `0x728`.
2. Stores a 32-bit zero at object offset `0x72c`.
3. Calls F-438680ffebfe with the global target reference and the address of the fixed fallback datum.
4. Continues to final result production.

No 64-bit source query is made on this path.

#### Nonzero result: source-query path

The function calls F-05cc4e9ef0f4 with the same embedded-source address. The result is treated as a 64-bit value.

After that call, it reads the cached 64-bit value at object offset `0x728` and compares the two values.

If they are equal, it performs no resolution, cache write, indirect target operation, branch-selection query, or fallback operation. It proceeds directly to final result production.

The cache is read after F-05cc4e9ef0f4 returns. A dependency-induced change to the cached region before that read can therefore affect the comparison.

### 3. Resolution of a changed value

When the queried 64-bit value differs from the cache, the function resets two local 32-bit slots to zero:

- A data slot.
- An extent slot.

It then calls F-79fa759bfe87 with:

1. The low 32 bits of the queried value.
2. The high 32 bits of the queried value.
3. The address of the data slot.
4. The address of the extent slot.

This splitting preserves the queried value’s 64-bit bit pattern. It does not establish a signed interpretation of that value.

After resolution, the function captures the global target reference for use in locating a possible indirect operation, then examines the extent slot.

#### Zero extent

If the extent is zero:

- The function does not write the newly queried value to the cache.
- It does not invoke the indirect target operation.
- It does not query which update path to use.
- It calls F-438680ffebfe with the global target reference and the fixed fallback datum address.
- It proceeds to final result production.

The data slot does not affect this decision.

The function itself leaves the previous cache unchanged on this path. Consequently, a later invocation can attempt resolution again if the same queried value still differs from that cache.

#### Nonzero extent

If the extent is nonzero, the function first stores the queried 64-bit value into the cache at object offset `0x728`.

It then invokes an indirect target operation selected from table offset `0x0c`, passing the global target reference as its sole explicit argument.

The cache write occurs before this indirect operation and before all subsequent update work. There is no rollback if later operations reject the data or fail to return normally.

The extent test is only a zero/nonzero test. A 32-bit extent with its high bit set still takes this path; no positive-range or upper-bound check is visible.

### 4. Selection of the update path

After the indirect target operation returns, the function calls F-b271d393c898 with the embedded-source address.

#### Zero result: allocation-and-copy path

The following operations occur in order:

1. Call F-fab3799b3190 with the extent.
2. Retain its 32-bit result as a temporary storage value.
3. Call an unidentified dependency with that storage value, the resolved data value, and the extent.
4. Set a local output slot to zero.
5. Call F-63011374610b with the global target reference, temporary storage value, extent, and address of the local output slot.
6. Call F-58ba61416cc9 with the temporary storage value.

The unidentified dependency occupies a copy-like position in this sequence, but its exact semantics are not supplied.

The local output slot initially held a copy of the caller’s output address. It is explicitly reset to zero before F-63011374610b receives its address. Any value that dependency writes into the slot is not subsequently used by this function.

In particular, that local write cannot redirect the final caller-output store: the final store uses the original output argument.

There is no check of the result from F-fab3799b3190 before subsequent calls use it.

#### Nonzero result: validation-gated path

The function calls a second unidentified dependency with the resolved data value and extent.

If it returns zero, the function performs no further update operation on this path and does not invoke the fallback operation.

If it returns nonzero:

1. Call F-9f8c9e1a456f with the resolved data value and extent.
2. Call F-5f2661b4a1e8 with the global target reference, resolved data value, and the preceding call’s 32-bit result.

In either case, the cache has already been updated and the indirect target operation has already occurred.

The second unidentified dependency acts as a gate. Calling it a content validator is an interpretation, not an established semantic guarantee.

### 5. Final result production

Every normally returning path converges on:

1. Call F-44362605ab79 with the global target reference.
2. Store its 32-bit result through the original output address.
3. Return without an explicit result value.

No result from an earlier operation is substituted for this final query.

## Side effects

### Direct writes

The established direct writes to caller-accessible memory are:

- Two ordered 32-bit zero stores to the cached region when source availability returns zero.
- One 64-bit cache store when a changed source value resolves to a nonzero extent.
- One final 32-bit store through the original output address.

No non-aliasing requirement is enforced. If the output overlaps the cache or another participating object field, the final store occurs after the earlier path-specific operations and may overwrite part of that state.

### Dependency-visible local storage

F-79fa759bfe87 receives addresses of two local 32-bit slots. F-63011374610b receives the address of a separate local slot reset to zero.

These addresses are valid for the immediate calls. Whether dependencies retain them is unknown; no lasting ownership transfer is established.

### Ownership and lifetime

The allocation-and-copy path pairs a call to F-fab3799b3190 with a later call to F-58ba61416cc9 using the same returned value. This supports an allocation/release interpretation, but does not prove either operation’s semantics.

The function does not establish:

- Ownership of the resolved data.
- Whether target-update dependencies copy or retain supplied data.
- Whether any dependency releases input memory.
- Whether the temporary storage remains valid after F-58ba61416cc9.
- Whether any target operation invalidates previously obtained values.

Those requirements must come from the dependency contracts.

### Global state and failures

Global references are used again at later call sites. The function does not establish one immutable target identity across all calls.

For the indirect operation, the dispatch table is selected using a captured target reference, while the explicit argument is obtained from the global target reference. Concurrent mutation is not synchronized by any established mechanism.

Most dependency results are ignored. There is no explicit error return, exception handling, rollback, retry loop, or guaranteed cleanup on abnormal exit.

## Dependencies

| Opaque ID | Established role at this call site |
|---|---|
| F-b255b26284e6 | Preliminary gate; zero skips the guarded sequence. |
| F-5b24edfb22c0 | Produces a 32-bit value used by the preliminary target operation. |
| F-804adc29bbfb | Receives that value and two global values. |
| F-d08d3ca17b81 | Final operation in the guarded preliminary sequence. |
| F-80f9e436da7e | Tests embedded-source availability by zero/nonzero result. |
| F-05cc4e9ef0f4 | Produces the 64-bit value compared with the cache. |
| F-79fa759bfe87 | Receives split 64-bit input and addresses of data and extent slots. |
| F-b271d393c898 | Selects between the two update paths. |
| F-fab3799b3190 | Produces temporary storage from an extent. |
| F-63011374610b | Receives target, temporary storage, extent, and a local output-slot address. |
| F-58ba61416cc9 | Receives the temporary storage after the preceding update call. |
| F-9f8c9e1a456f | Produces a value from resolved data and extent. |
| F-5f2661b4a1e8 | Receives target, resolved data, and that produced value. |
| F-438680ffebfe | Performs the fallback operation using a fixed datum address. |
| F-44362605ab79 | Produces the final 32-bit caller-output value. |

Two direct dependencies lack supplied opaque IDs: the copy-like operation and the validation gate. The indirect operation at table offset `0x0c` also has no supplied opaque ID. Their internal behavior remains documented_only.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. They assume dependencies return normally and do not introduce unrelated state changes unless stated.

1. **Source unavailable.** Availability returns zero. The two cache words are cleared in increasing offset order, fallback is invoked, and the final query result is written to the output. No 64-bit query or resolution occurs.

2. **Source unchanged.** Availability is nonzero and the queried 64-bit value equals the cache when compared. Resolution and all update/fallback operations are skipped. The preliminary sequence may already have run. The final query still occurs.

3. **Changed value resolves to zero extent.** Resolution receives zero-initialized slots and leaves the extent zero. Fallback occurs, but the function does not replace the old cache with the queried value.

4. **Changed value takes the allocation-and-copy path.** Resolution produces nonzero extent and branch selection returns zero. The cache store precedes the indirect target operation. Temporary storage production precedes the copy-like call, target update, and matching final temporary-storage call. Final output follows all of them.

5. **Validation gate rejects.** Resolution produces nonzero extent, branch selection is nonzero, and the validation gate returns zero. The cache remains updated; neither fallback nor F-5f2661b4a1e8 is invoked. A subsequent call may take the cache-equality shortcut despite no accepted update on this invocation.

6. **High-bit extent.** Resolution produces extent `0x80000000`. It is treated as nonzero and forwarded without visible range validation. Whether downstream operations accept that bit pattern is unspecified.

7. **Local output-slot replacement.** F-63011374610b writes a different address-like value into its local output slot. The final result is nevertheless stored through the original caller-output address.

## Uncertainties

The following remain documented_only:

- Domain meaning of the source, target, cached value, resolved data, extent, and fallback datum.
- Exact dependency side effects, failure conventions, memory ownership, and retention rules.
- Identities and implementations of the two unmapped direct dependencies and the indirect operation.
- Whether the preliminary sequence implements locking, initialization, reference management, or another protocol.
- Whether the extent is a byte count or another measure.
- Alignment, endianness, atomicity, and concurrency guarantees beyond the stated accesses.
- Behavior when invalid pointers, unsupported extents, or unsuccessful temporary-storage production reach dependencies.
- Whether cache-before-update ordering intentionally suppresses repeated attempts after a rejected update.

No callee is assumed to preserve globals or to acquire, retain, or release memory merely because of its position in the sequence.

## Evidence

The established behavior is derived solely from the supplied static function body and opaque dependency mapping: branch predicates, field offsets, access widths, argument order, local-slot initialization, and call ordering.

“Change token,” “availability,” “allocation-and-copy,” and “validation” are descriptive interpretations of those relationships, not independently verified dependency semantics.

No runtime tests have been performed. The acceptance examples describe consequences of the visible control flow under their stated conditions; they are not observations of execution.
