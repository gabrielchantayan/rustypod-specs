<!-- Trusted provenance: F-28d84eedeb6a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-28d84eedeb6a

## Purpose

F-28d84eedeb6a conditionally processes a nonzero 32-bit value obtained through an object’s association field. An initially zero value produces a successful return without dependency calls. An initially nonzero value requires a nonzero permission byte; otherwise the function invokes a diagnostic dependency and returns one.

When permission is enabled, the function passes the initially observed value to a dependency, rereads the association field, writes four zero bytes to the newly selected slot, and invokes a follow-on dependency before returning zero.

These outcomes assume that all required accesses succeed and invoked dependencies return normally. The explicit zero write is not necessarily to the initially observed slot. Its destination may also overlap the input cell, permission byte, association field, or other storage. The supplied evidence establishes no disjointness requirement.

The diagnostic reference suggests a restriction concerning changes to temporary storage. That is inferred intent. Neither the concrete object type nor the meaning, ownership, or lifetime of the processed value is established.

## Interface

### Input and layout

The function accepts the address of a cell containing a 32-bit object address. It reads that cell once and retains the resulting object address for subsequent direct object-relative accesses and the follow-on call.

The visible layout is:

| Role | Location | Access width | Use |
|---|---|---:|---|
| Input cell | Supplied address | 32 bits | Provides the object address captured at entry |
| Association field | Captured object address plus 8 bytes | 32 bits | Provides an associated-structure address |
| Target slot | Associated-structure address plus 28 bytes | 32 bits | Initially tested and passed to a dependency; later selected again for clearing |
| Permission field | Captured object address plus 28 bytes | 8 bits | Zero rejects processing; any nonzero byte permits it |

These are addressing roles, not proof of distinct storage allocations. In particular:

- The object-relative permission field and associated-structure-relative target slot have the same offset but different address bases.
- Those bases may be equal, making the permission byte overlap the target slot.
- Other address relationships may make a target-slot access overlap the association field or input cell.
- Partial overlaps are not excluded by the supplied evidence.

The target-slot value is 32 bits wide. Its interpretation as a pointer, handle, resource identifier, or ordinary integer remains unknown.

### Output

The explicit result is four bytes wide:

- **Zero:** the initial target-slot value was zero, or permitted processing reached the normal return.
- **One:** the initial target-slot value was nonzero, the permission byte was zero, and the diagnostic dependency returned normally.

Dependency return values do not determine these results. If a dependency does not return normally, the enclosing path does not reach its subsequent explicit result.

### Access preconditions

The caller must supply storage that supports the accesses reached on the selected path:

- Every path requires a readable input cell, a readable association field in the captured object, and a readable initial target slot.
- An initially nonzero target slot additionally requires a readable permission byte.
- The permitted path requires the captured object’s association field to be readable again after the processing dependency returns.
- That second association value must select a writable four-byte target slot.

There are no visible null, bounds, alignment, or validity checks before these accesses. A zero association address is not treated as an empty target slot; the function still attempts the slot read through that address.

No non-aliasing precondition is established. An overlapping arrangement is within the described behavior if it supports the required accesses and satisfies any applicable dependency requirements. Those dependency requirements are unavailable.

## Behavior

### Initial observations

The function performs these observations in order:

1. Reads the 32-bit object address from the input cell.
2. Reads the 32-bit association field at offset 8 in that captured object.
3. Reads the 32-bit target slot at offset 28 in the associated structure selected by that field.

The initial target-slot value controls whether the permission byte is read. The permission byte is not consulted first.

The captured object address remains the basis for subsequent direct object-relative accesses. The input cell is not reread after a dependency call or after the explicit store.

### Initially zero target slot

If the initial target-slot value is zero, the function returns zero.

On this path:

- The permission byte is not read.
- No dependency is called.
- No explicit memory write occurs.

A disabled or unreadable permission byte therefore does not itself prevent this path, provided the preceding required reads are valid.

### Nonzero target slot with permission disabled

If the initial target-slot value is nonzero, the function reads the single permission byte at object offset 28.

If that byte is zero, the function calls F-2022b0f73f3b with:

1. The original input-cell address.
2. An address identifying fixed diagnostic data.

If the dependency returns normally, the function returns one.

This path performs no explicit target-slot clear and calls neither F-7446dea3e07b nor F-aacdb1821b51. It does not guarantee unchanged memory: the diagnostic dependency’s effects are unknown.

The diagnostic reference supports an inferred connection to a restriction on changing temporary storage. The complete diagnostic contents, encoding, presentation, and severity are not established.

### Nonzero target slot with permission enabled

Any nonzero permission byte selects the permitted path. The value need not be one.

The function calls F-7446dea3e07b with the initial 32-bit target-slot value. That argument is the previously observed value, not a new slot read performed after the permission check.

If F-7446dea3e07b returns, the function:

1. Rereads the association field at offset 8 in the captured object.
2. Uses that newly read address to select a target slot at offset 28.
3. Writes a 32-bit zero to that selected slot.
4. Calls F-aacdb1821b51 with the captured object address and a zero-valued second argument.
5. Returns zero if the follow-on dependency returns normally.

The second association read is observable behavior. If F-7446dea3e07b changes the captured object’s association field, the explicit clear follows the changed association. The initially selected structure is not necessarily the destination.

Changing the input cell does not redirect this later association read or the follow-on call. Both continue to use the object address captured at entry.

### Aliasing during the explicit clear

The clear targets four bytes beginning at the freshly selected associated-structure address plus 28. No evidence establishes that this range is disjoint from other named locations.

Consequently:

- If the freshly selected associated-structure address equals the captured object address, the clear also sets the permission byte to zero and clears the following three bytes.
- If the clear’s destination coincides with the association field, the clear also sets that field to zero.
- If its destination coincides with the input cell, the clear also overwrites the stored object address with zero.
- Partial overlap modifies the overlapping bytes of the affected location.

These are effects of the function’s own explicit store, not effects that must be attributed to a dependency.

An overlapping clear does not retroactively change values already captured or tested. In particular, clearing the input cell does not change the retained object address passed to F-aacdb1821b51. Clearing the association field does not redirect the store already selected from its preceding read.

No direct reread of the permission byte or association field occurs between the clear and the follow-on call. Whether the resulting state is acceptable to that dependency is unknown.

### Widths and arithmetic

The input-cell read, association-field reads, initial target-slot read, and explicit clear are each four bytes wide. The permission read is one byte wide. Testing that byte distinguishes zero from all nonzero byte patterns; signed-character interpretation does not alter this distinction.

The address relationships use fixed byte offsets of 8 and 28. There is no visible iteration, size computation, or explicit overflow handling. The source representation uses 32-bit address-bearing values, but the supplied evidence does not independently specify machine behavior for wrapping arithmetic, invalid addresses, unaligned accesses, or access faults.

The four-byte zero store specifies the value and width of the explicit clear. It does not establish atomicity.

## Side effects

The function has one explicit store, reached only on the permitted path: a four-byte zero written to the target slot selected through the post-processing association read.

The input cell, permission field, and association field are not separately targeted by additional explicit stores. They may nevertheless be modified by that single store through full or partial aliasing. Other overlapping storage may also be modified.

The visible body contains no separate allocation or release operation. Passing the initial slot value to F-7446dea3e07b does not establish that the dependency frees, consumes, retains, or otherwise owns anything.

The permitted-path order is:

- Processing dependency call.
- Fresh association-field read.
- Four-byte zero write.
- Follow-on dependency call.
- Zero return, if execution reaches it.

The processing dependency runs before the explicit clear. The follow-on dependency runs after it and can observe whatever state the clear produced, subject to its own behavior and the execution environment.

A zero return does not guarantee that the selected slot remains zero at return. F-aacdb1821b51 may modify it after the clear. Likewise, a one return does not guarantee that the diagnostic dependency preserved the input cell or any other state.

No dependency is assumed to preserve globals, object fields, associations, memory contents, or resource lifetime. Successful return from a dependency alone does not establish that later memory accesses remain valid.

No concurrency, synchronization, reentrancy, or transactional guarantee is visible. The separate reads, calls, and write must not be collapsed into one atomic state transition.

## Dependencies

| Opaque ID | Invocation condition | Arguments | Established relationship to this function |
|---|---|---|---|
| F-2022b0f73f3b | Initial slot nonzero and permission byte zero | Original input-cell address; fixed diagnostic-data address | Called before the explicit return of one |
| F-7446dea3e07b | Initial slot nonzero and permission byte nonzero | Initially observed 32-bit slot value | Called before the association reread and clear |
| F-aacdb1821b51 | Permitted processing returns and the clear completes | Captured object address; zero | Called after the clear and before the explicit return of zero |

The supplied evidence establishes argument selection and call order, not dependency internals.

No ownership transfer, retention rule, release obligation, or lifetime guarantee is established for any argument. Dependency return values are not visibly inspected. Error signaling, exceptions, termination, and other nonlocal exits remain unspecified.

## Acceptance examples

These are conditional criteria derived from static evidence, not executed tests. Unless an example states otherwise, required accesses are valid and invoked dependencies return normally.

1. **Initially empty slot.**  
   The initial target-slot read yields zero. The result is zero, the permission byte is not read, no dependency is called, and no explicit store occurs. This remains true whether the permission byte would contain zero or a nonzero value.

2. **Occupied slot with permission disabled.**  
   The initial slot is nonzero and the permission byte is zero. F-2022b0f73f3b receives the original input-cell address and diagnostic-data address. The function returns one without its explicit clear or either permitted-path dependency call. Memory preservation is not required of the diagnostic dependency by this contract.

3. **Noncanonical enabled permission byte.**  
   The initial slot is nonzero and the permission byte contains 128. F-7446dea3e07b receives the initial slot value. After it returns, the function rereads the association, clears the selected four-byte slot, calls F-aacdb1821b51 with the captured object address and zero, and returns zero.

4. **Association changed during processing.**  
   The initial slot belongs to one structure. F-7446dea3e07b changes the captured object’s association to select another valid structure. The explicit clear targets the second structure’s slot. The function does not additionally clear the first slot; however, preservation of the first slot requires that the clear destination not overlap it and that dependencies not modify it.

5. **Input cell changed by the processing dependency.**  
   F-7446dea3e07b replaces the object address stored in the input cell. The subsequent direct association read and follow-on argument still use the object address captured before that call.

6. **Clear overlaps the permission field.**  
   The initial conditions permit processing, and the association reread selects the captured object itself as the associated structure. The four-byte clear begins at object offset 28, clearing the permission byte and three adjacent bytes. This is a direct effect of F-28d84eedeb6a. The permission change does not cause rejection because its branch decision has already occurred.

7. **Clear overlaps the association field.**  
   The freshly selected target slot coincides with the captured object’s four-byte association field. The clear sets that field to zero. The function then calls F-aacdb1821b51 with the captured object address; it performs no intervening association reread. Normal completion remains conditional on the dependency tolerating the resulting state.

8. **Clear overlaps the input cell.**  
   The freshly selected target slot coincides with the four-byte input cell. The explicit clear zeroes that cell, but the follow-on call still receives the previously captured object address.

9. **Follow-on dependency changes cleared storage.**  
   After the explicit clear, F-aacdb1821b51 writes a nonzero value to the same slot. The function still returns zero. A requirement that the slot remain zero at return would exceed the established behavior.

10. **Processing dependency does not return.**  
    F-7446dea3e07b does not return normally. The later association read, explicit clear, follow-on call, and normal zero return are not reached.

## Uncertainties

The following remain **documented_only**:

- Concrete types, complete layouts, allocation extents, and alignment requirements.
- Whether particular callers enforce disjointness among the named storage roles.
- The meaning and ownership of the target-slot value.
- Whether F-7446dea3e07b releases, consumes, or retains a resource.
- Dependency side effects, return contracts, and lifetime requirements.
- The follow-on dependency’s behavior when aliasing has cleared the permission byte, association field, or input cell.
- Complete diagnostic contents and whether the diagnostic is displayed, recorded, or fatal.
- Machine-specific behavior for invalid, wrapping, unaligned, or concurrently modified addresses.

The temporary-storage interpretation is inferred intent. Access widths, conditional reads, argument selection, the association reread, the explicit store, and call order are established static behavior. Aliasing effects follow from the established store destination and width; the actual occurrence of any particular overlap is not established.

## Evidence

This contract is based on the supplied function excerpt and opaque dependency mapping. No dependency bodies or independent implementation access were available. No runtime tests have been performed.

Static evidence establishes one input-cell read, the initial association and slot reads, the conditional permission-byte read, three explicit normal-return paths, and the permitted-path sequence of processing call, association reread, four-byte zero store, and follow-on call.

The evidence establishes only one explicit store and no disjointness guarantee for its destination. It therefore supports describing other named locations as not separately targeted, but not as protected from modification through aliasing. Dependency internals and final memory state beyond the function’s ordered operations remain unresolved.
