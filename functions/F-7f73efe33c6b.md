<!-- Trusted provenance: F-7f73efe33c6b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7f73efe33c6b

## Purpose

F-7f73efe33c6b locates a counted table in an input region, allocates storage for expanded records, resolves two character-sized values per source record through F-c841521529ba, and conditionally invokes F-4d426a8054f4 on the resulting array.

Each source record occupies four bytes. Each destination record occupies sixteen bytes and contains two dependency results, a zero-extended sixteen-bit source value, and a zero-valued field.

The function also conditionally invokes F-7e3c181d3a8f before and after expansion. Temporary context selection is an inferred purpose, not an established restoration guarantee.

The fourth argument to F-4d426a8054f4 is read from a global location when preparing that call. It is not established as a fixed callback identity. Earlier dependencies may affect its eventual value.

This contract describes static evidence only. No runtime tests have been performed.

## Interface

### Arguments and accessed layouts

The function accepts three thirty-two-bit values used as addresses.

| Argument | Offset | Accessed value |
|---|---:|---|
| Context | `0x24` | Signed thirty-two-bit entry count |
| Context | `0x28` | Thirty-two-bit address of an array of thirty-two-bit entry addresses |
| Context | `0x5c` | Thirty-two-bit saved-state value |
| Input descriptor | `0x1c` | Thirty-two-bit resource-context value |
| Input descriptor | `0x20` | Thirty-two-bit input-base address |
| Input descriptor | `0x24` | Thirty-two-bit input-limit address |
| Destination descriptor | `0x24` | Thirty-two-bit destination-array address |
| Destination descriptor | `0x28` | Thirty-two-bit destination-record count |

Each context entry is accessed through its address in the entry-address array. The function examines a sixteen-bit field at entry offset `8`, comparing it with `7`.

The input limit acts as an exclusive upper bound in the visible extent comparisons: a read ending exactly at that limit is admitted. This does not establish allocation provenance or protection against inaccessible storage.

There are no explicit null, descriptor-size, alignment, or accessibility checks before descriptor accesses.

### Return value

The return value is sixty-four bits, assembled from two thirty-two-bit components:

- **Low half:** the local status at return.
- **High half:** the original input-descriptor argument until a nonzero parsed count passes the source-payload extent check; zero afterward.

The initial status is zero. Explicit truncation branches assign `2`. F-6dd003795db7 receives the address of this status, and results from F-7e3c181d3a8f are assigned to it.

After the status address has been passed to F-6dd003795db7, the supplied evidence does not exclude retention of that address or later modification through it. In particular, a status that selects cleanup need not equal the status ultimately returned after F-bf7242c98d06.

A zero low half does not imply a zero sixty-four-bit result, a newly allocated destination, or an initialized destination pointer.

### Destination record layout

| Offset | Width | Value before ordering |
|---|---:|---|
| `0` | 32 bits | First F-c841521529ba result |
| `4` | 32 bits | Second F-c841521529ba result |
| `8` | 32 bits | Zero-extended unsigned sixteen-bit source field |
| `12` | 32 bits | Zero |

The semantic meaning of the resolved values and trailing zero is not supplied.

### Address and storage preconditions

Addresses and address calculations are thirty-two-bit. No explicit overflow detection is present. A conventional bounded-buffer interpretation therefore requires accessible storage and nonwrapping address calculations.

Direct sixteen-bit reads include input-base offset `99`, which may be unaligned. The execution environment must support the resulting accesses.

For straightforward record expansion, source storage, destination storage, descriptors, and dependency-accessible state must not overlap in ways that invalidate later reads or writes. The function does not enforce that condition.

## Behavior

### Descriptor capture and locator validation

The function initializes its status to zero, then reads the input base, resource-context value, and input limit, in that order. Subsequent work uses these captured values rather than refreshing them from the input descriptor.

The first extent check requires the limit to reach input base plus `101`. Failure assigns status `2` and enters cleanup, even though no allocation has occurred.

On success, the function reads an unsigned sixteen-bit displacement at input-base offset `99`.

It then checks the following, with short-circuit rejection:

1. The limit reaches input base plus `99`, plus the displacement, plus `36`.
2. The unsigned sixteen-bit field at input base plus `99`, plus the displacement, plus `18` is at least `18`.

Failure of the first condition prevents the second field read. Failure of either condition returns directly with the current zero status and the original input-descriptor argument in the high half. Neither destination field is directly written, and cleanup is not invoked.

After those checks, four bytes at offsets `32` through `35` from the displacement-adjusted locator are assembled into an explicitly little-endian thirty-two-bit offset. The representation establishes the assembly but does not establish a portable sequencing guarantee among those four byte reads.

Adding that offset to the captured input base produces the table address. If the result equals the input base, the function returns directly without changing the destination. Under nonwrapping arithmetic, this rejects offset zero. It does not otherwise require the table to follow the locator or establish a lower bound for the table.

### Count publication and source extent

The table must provide two readable bytes for its count. If the limit is below table address plus two, the function assigns status `2` and enters cleanup.

Otherwise, it reads the unsigned sixteen-bit count and immediately writes its zero-extended value to destination offset `0x28`.

It next checks that the limit reaches the end of the following `count × 4` source bytes. Failure assigns status `2` and enters cleanup. Thus, this failure publishes the parsed count before cleanup clears the destination fields.

A valid zero count returns directly with zero status and the original input-descriptor argument in the high half. The existing destination pointer remains unchanged. There is no allocation, context scan, resolver call, or ordering call.

The parsed count ranges from zero through `65,535`. Its maximum source-payload size is `262,140` bytes; its maximum sixteen-byte destination-payload size is `1,048,560` bytes. These products fit within thirty-two bits. The surrounding address additions remain unchecked.

### Allocation and status exposure

For a nonzero count whose source extent passed validation, the function first sets the eventual return’s high half to zero.

It calls F-6dd003795db7 with:

1. The captured resource-context value.
2. Element width `16`.
3. Zero.
4. The parsed count.
5. Zero.
6. The address of the local status.

The returned thirty-two-bit value is written to destination offset `0x24` before the status is checked.

A nonzero status selects cleanup. A zero status permits continuation, without a separate check that the returned address is nonzero or suitable for writes.

The previously stored destination pointer is overwritten without an explicit preceding release. Allocation policy, ownership transfer, and retention of the supplied status address are not established.

### Expansion bound and context scan

After allocation reports zero status, the function computes a source-loop endpoint using the count then stored at destination offset `0x28`. This is a fresh descriptor read, not necessarily the originally parsed count.

The earlier source-extent check covered the parsed count. If F-6dd003795db7 or another permitted mutation changes the descriptor count, the subsequently computed endpoint is not independently revalidated. The endpoint calculation uses thirty-two-bit arithmetic and four bytes per count unit.

The function then captures the context field at offset `0x5c`.

Starting at index zero, it scans while the signed index is less than the signed context entry count. For each examined entry, it obtains the entry address through the context’s address array and reads the sixteen-bit field at entry offset `8`.

At the first field equal to `7`, it calls F-7e3c181d3a8f with the context as the only explicit argument shown at this call site. Its result replaces the local status:

- Nonzero status enters cleanup immediately.
- Zero status ends the scan and permits expansion.

No later matching entry is examined. A zero or negative context entry count causes no entry examination.

The saved context value is captured after allocation. The scan does not establish an immutable snapshot of the context count or entry-address array.

### Record expansion

The source cursor begins immediately after the two-byte table count. The destination cursor begins at the allocation result. The endpoint remains the one calculated before the context scan.

For each source cursor below that endpoint, the visible sequence is:

1. Read the source record’s first sixteen-bit unit and convert it to a character-sized value.
2. Call F-c841521529ba with the context and that converted value.
3. Write its thirty-two-bit result at destination-record offset `0`.
4. Read the source byte at offset `1`.
5. Call F-c841521529ba with the context and that byte-sized value.
6. Write its thirty-two-bit result at destination-record offset `4`.
7. Read the unsigned sixteen-bit source field at offset `2`.
8. Zero-extend it and write it at destination-record offset `8`.
9. Write zero at destination-record offset `12`.

The source and destination cursors then advance by four and sixteen bytes respectively.

The first resolver argument comes from narrowing a sixteen-bit read, not from an independently established byte load at offset zero. On an ordinary little-endian interpretation, its low eight bits correspond to the byte at offset zero. Native byte order and character signedness remain unresolved.

Neither resolver return is assigned to the status. There is no per-record status check. Nevertheless, dependencies are not established to preserve the status through any retained reference.

The function does not snapshot source records before invoking dependencies. A resolver can therefore affect later source reads if it can modify the relevant storage. Changes to the destination descriptor’s pointer do not redirect the already established local destination cursor.

### Post-expansion action and ordering

After expansion, a nonzero saved context value causes another F-7e3c181d3a8f call, this time with the context and saved value as the two explicit arguments shown. Its result replaces the local status.

This call depends on the saved value, not on whether the scan invoked the dependency. A saved zero suppresses it even if the earlier call changed context state. A failing initial context call reaches cleanup without this later call.

If the status is zero, the function prepares a call to F-4d426a8054f4 using:

- The destination pointer then stored at offset `0x24`.
- The count then stored at offset `0x28`.
- Record width `16`.
- The then-current value read from a global location.

The global-value read occurs when preparing this ordering call, after the resolver calls and any post-expansion context call. Its initialization and immutability are not supplied. The function does not capture a callback value earlier and preserve it across dependencies. The supplied representation does not establish a relative read order among the ordering call’s argument expressions.

The ordering dependency’s return value is ignored. The function checks the local status again after the call. Zero returns directly; nonzero enters cleanup. The post-call check matters because the local status address has previously been exposed.

### Cleanup

Every path reaching cleanup performs these actions in order:

1. Call F-bf7242c98d06 with the captured resource-context value and the destination pointer currently stored at offset `0x24`.
2. Write zero to destination offset `0x24`.
3. Write zero to destination offset `0x28`.
4. Return the current local status and current high-half value.

Cleanup does not use the local expansion cursor as its release argument.

Some truncation failures reach cleanup before allocation and pass the caller’s preexisting destination pointer. Other failures pass the pointer published by allocation or a later replacement stored in the descriptor.

The cleanup call precedes the final status read. Following exposure of the status address to F-6dd003795db7, equality between an earlier error status and the returned low half requires that intervening dependencies do not modify that local through a retained reference.

## Side effects

- Locator rejection can leave the destination unchanged while returning a zero low half.
- Count parsing writes destination offset `0x28` before payload validation.
- A valid zero count clears the count but preserves the existing destination pointer.
- Allocation publishes its result before the allocation-status check.
- Expansion writes sixteen bytes per processed record, interleaved with resolver calls.
- Ordering uses fresh destination-field reads and a fresh global-value read; these need not match earlier values.
- Cleanup invokes F-bf7242c98d06 before clearing both destination fields.
- Dependencies may modify context, source storage, globals, descriptor fields, or an escaped status reference where accessible. No preservation contract is supplied.
- No direct input-region write or input-region release is shown.

On a normal nonzero-count success with stable descriptor fields, the allocation remains published and this function performs no release. Responsibility for its eventual release is unresolved.

## Dependencies

| Opaque ID | Visible role | Unestablished contract |
|---|---|---|
| F-6dd003795db7 | Returns an array address and receives a status address | Allocation policy, ownership, failure convention, status-address retention, and mutations |
| F-bf7242c98d06 | Receives a resource context and current destination pointer during cleanup | Actual release behavior, accepted pointers, ownership requirements, and effects through escaped references |
| F-c841521529ba | Produces each of the first two destination fields | Argument interpretation, result semantics, errors, and mutations |
| F-7e3c181d3a8f | Conditional context-related action with differing visible argument forms | Full signature, state semantics, restoration behavior, and mutations |
| F-4d426a8054f4 | Receives array, count, width, and a global-sourced value | Ordering rules, callback interpretation, stability, and mutations |

The global location supplying the fourth ordering argument is a separate data dependency. Its value is not established as constant, initialized to a particular callback, or preserved by other dependencies.

## Acceptance examples

These are conditional examples, not executed tests. Unless otherwise stated, they assume accessible nonoverlapping storage, nonwrapping arithmetic, and dependencies that do not mutate unrelated relevant state.

1. **Input ends before offset `101`.**  
   Status becomes `2`. Cleanup receives the preexisting destination pointer, then both destination fields become zero. The high half remains the input-descriptor argument.

2. **Displacement-dependent extent is insufficient.**  
   The displacement is read, but the later size field is not. The destination remains unchanged, the low half is zero, and cleanup is not called.

3. **The locator size field is below `18`.**  
   With the enclosing extent valid, the function returns without reading the table-offset bytes or changing the destination.

4. **The table offset is zero.**  
   The calculated table address equals the input base. No table count is read, and the destination remains unchanged.

5. **A valid table declares zero records.**  
   The destination count becomes zero; its pointer is preserved. No allocation, context action, resolver call, or ordering call occurs.

6. **Two records are declared but only one is available.**  
   The count is first published as `2`. Status becomes `2`; cleanup receives the existing destination pointer and clears both fields. The high half remains the input-descriptor argument.

7. **One record completes normally.**  
   Assuming little-endian direct reads and ordinary character conversion, source bytes `5`, `9`, `0x34`, `0x12` produce resolver inputs corresponding to `5` and `9`. Before ordering, the destination contains their two results, `4,660`, and zero. With status remaining zero, the complete return is zero.

8. **Allocation reports an error.**  
   Its returned pointer is published before the nonzero status selects cleanup. Cleanup receives the then-current destination pointer and clears both fields. The high half is zero. The returned low half equals the allocator-supplied status only if cleanup and intervening effects do not modify the local status through an escaped reference.

9. **The initial context action fails.**  
   Expansion, the later saved-state call, and ordering are skipped. Cleanup runs. The eventual low half is the status remaining after cleanup, not unconditionally the context dependency’s earlier result.

10. **The saved context value is zero.**  
    No post-expansion context call occurs, even if the initial scan invoked F-7e3c181d3a8f.

11. **A resolver changes the ordering global.**  
    If the ordering path is reached and no later effect changes that location again, F-4d426a8054f4 receives the updated value, not an entry-time callback snapshot.

12. **Ordering changes status through a retained reference.**  
    If status is zero before ordering but nonzero afterward, cleanup runs despite the ordering return value being ignored. The final low half is read after cleanup.

## Uncertainties

- **documented_only:** Native byte order for direct sixteen-bit reads; only the assembled table offset is explicitly little-endian.
- **documented_only:** Character signedness and promotion of resolver arguments.
- **documented_only:** Full dependency signatures, especially the two visible F-7e3c181d3a8f argument forms.
- **documented_only:** Initialization, identity, validity, and mutation rules of the global-sourced ordering argument.
- **documented_only:** Comparator semantics, ordering direction, and stability.
- **documented_only:** Whether the allocator retains the status address and whether later dependencies modify status through it.
- **documented_only:** Status meanings other than the locally assigned `2`.
- **documented_only:** Ownership obligations and release-dependency tolerance of null, preexisting, or replaced pointers.
- **documented_only:** Intended interpretation of the return’s high half.
- **documented_only:** Behavior under wrapped addresses, inaccessible storage, concurrency, or unrestricted aliasing.

Parsing, expansion, temporary context selection, and ordering are inferred purposes. They do not establish stronger validation, preservation, ownership, or restoration guarantees.

## Evidence

Established static evidence comprises the supplied function representation’s access widths, offsets, short-circuit branches, count publication, arithmetic, explicit byte assembly, dependency calls, record writes, global-value read, cleanup sequence, and return-half construction.

The representation explicitly exposes the local status address to F-6dd003795db7 and calls F-bf7242c98d06 before reading the status for return. It supplies no non-retention or preservation guarantee.

No dependency implementation, global initializer, callback implementation, external format specification, or runtime trace was supplied. No runtime tests have been performed.
