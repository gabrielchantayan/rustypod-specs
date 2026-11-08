<!-- Trusted provenance: F-ff4df0a761aa; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ff4df0a761aa

## Purpose

F-ff4df0a761aa performs a staged update on a mutable object:

1. Conditionally resolves a related object and passes it to a binding dependency.
2. Performs two table-driven lookups and, for each successful lookup, requests storage and initializes a helper associated with an embedded region of the object.
3. Invokes an unconditional virtual callback.
4. Conditionally performs an additional lookup, passes a successful result to another dependency, and invokes a final virtual callback.

These stages and their ordering are established by the supplied static evidence. Describing the related object as a “binding” and the two constructed values as “helpers” is inferred intent, not an established domain meaning.

The function has no explicit status result and no visible rollback. A failed lookup generally skips work rather than reporting an error. Invalid pointers, unavailable virtual targets, and allocation failure are not handled explicitly by the visible function.

## Interface

### Entry and return

The recovered interface accepts one pointer to a mutable object and returns no value.

The supplied representation uses 32-bit object words and 32-bit address-bearing values. The following offsets are byte offsets from the input object’s base. They describe the accesses made here, not a complete object definition.

| Offset | Accessed width | Role in this function |
|---|---:|---|
| `0x000` | 32 bits | Virtual-table address used for dispatch. |
| `0x034` | 32 bits | Optional associated-object address. |
| `0x038` | 32 bits | Context passed to both table-driven lookups. |
| `0x044` | 32 bits | Gate for the final conditional stage. |
| `0x048` | 32 bits | Flags; only bit zero is tested here. |
| `0x0B4` | 32 bits | Related-object result slot. |
| `0x0B8` | Low eight bits of a 32-bit value | Selects one of two table-base displacements. |
| `0x0BC` | 32 bits | Initial resolution gate and fallback lookup argument. |
| `0x0D8` | 32 bits | First helper-result slot. |
| `0x0DC` | Address only | Start of the first embedded region passed to initialization. |
| `0x1AC` | 32 bits | Second helper-result slot. |
| `0x1B0` | Address only | Start of the second embedded region passed to initialization. |
| `0x26C` | Eight bits | Unsigned table selector, read separately for each iteration. |

The embedded regions begin `0xD4` bytes apart. The helper-result slots also lie `0xD4` bytes apart. The function supplies the region addresses to a dependency; the visible evidence does not establish how many bytes that dependency accesses.

### Preconditions

For defined completion, the object must support every access and dispatch actually reached:

- The input pointer must designate suitably accessible object storage.
- Each reached virtual dispatch must have a valid table and callable entry.
- The fallback resolution path requires its intermediate returned object to support dispatch; no null check protects that intermediate result.
- The selected table entries must be readable.
- Dependencies must accept the arguments passed to them, including a zero storage result if the storage-request dependency can return zero.

These are requirements for safe execution, not validations performed by F-ff4df0a761aa.

## Behavior

### 1. Conditional related-object resolution

The function first reads the full 32-bit value at `0x0BC`.

If that value is zero, the entire resolution stage is skipped. In particular, the function does **not** clear or otherwise directly replace the existing value at `0x0B4`.

If the value is nonzero:

1. The function writes zero to `0x0B4`.
2. It reads the associated-object address at `0x034`.
3. It attempts the associated-object route when that address is nonzero; otherwise it takes the fallback route.

#### Associated-object route

The associated object supplies a virtual target through its table entry at byte displacement `0x0E4`. The recovered call shows no explicit arguments. Its precise calling convention and any implicit receiver are not established by the supplied evidence.

If this call returns zero, the provisional related-object result is zero.

If it returns a nonzero object address, the function calls that returned object’s virtual entry at displacement `0x014`, passing:

- The returned object as receiver.
- The 32-bit constant `0x1180`.

The resulting 32-bit value becomes the provisional related-object result.

The function writes the provisional result to `0x0B4`. A nonzero result completes resolution. A zero result proceeds to the fallback route.

#### Fallback route

The function calls the input object’s virtual entry at displacement `0x05C`, passing the input object as receiver.

It then uses the returned address as an object and invokes its virtual entry at displacement `0x0E4`, passing:

- That returned object as receiver.
- The value read from the input object’s `0x0BC` field for this call.

There is no visible null check between these two dispatches.

The fallback argument is a fresh field read after the first virtual call. It is not necessarily equal to the nonzero value that originally enabled the stage: an intervening call could have changed it.

The fallback result is written to `0x0B4`. If it is zero, resolution ends without calling the binding dependency. If it is nonzero, resolution proceeds to binding.

#### Binding

After either route produces a nonzero result, F-8f9bd59e3ec5 is called with:

- The value read from `0x0B4`.
- The input object address.

No return value from this dependency is used.

The visible function does not restore `0x0B4` afterward. Consequently, any changes made by the dependency remain visible unless later operations change them.

### 2. Two table-driven lookup and initialization attempts

After the resolution stage, including any binding call, the function obtains:

- A shared 32-bit lookup argument from a global data location.
- A global table base.
- A mode derived from whether the low eight bits of the object value at `0x0B8` are nonzero.

Only those eight bits determine the mode. For example, a 32-bit value of `0x00000100` selects the same mode as zero.

The mode selects a table displacement of either zero or `0x10` bytes. That displacement is retained for both iterations; this function does not reevaluate the mode between them. The shared lookup argument is likewise retained.

Exactly two iterations are attempted, in first-slot then second-slot order, provided calls return normally.

For iteration number \(j\), where \(j\) is zero or one, the selected 32-bit table entry is located at:

\[
\text{table base} + \text{mode displacement}
+ 8 \times \text{unsigned selector}
+ 4 \times j.
\]

The selector is the byte at object offset `0x26C`. It is read separately for each iteration, so a dependency called during the first iteration may affect the entry chosen in the second.

The selector is zero-extended: values `0x80` through `0xFF` are not interpreted as negative indices. The greatest displacement permitted by these widths is `0x80C`, for the nonzero mode, selector `0xFF`, and the second iteration. A table accommodating every possible combination would therefore need a readable 32-bit entry through byte displacement `0x80F`. The function performs no selector-range check.

Each iteration calls F-5710a8628cba with:

- The current 32-bit value at object offset `0x038`.
- The retained shared lookup argument.
- The selected 32-bit table entry.

The context at `0x038` is read for each call rather than captured once for the pair. The evidence does not establish a meaningful relative evaluation order between that context read and the corresponding table-entry read; both supply the same lookup invocation.

If the lookup result is zero, the iteration ends without a storage request, initialization call, or direct write to its helper-result slot. In particular, a preexisting helper-result value is not cleared.

If the lookup result is nonzero:

1. F-8157a180d901 is called with the size argument `0x44`, or 68 bytes.
2. F-f8beff1830ad is called with:
   - The storage-request result.
   - The address of the corresponding embedded region.
   - The nonzero lookup result.
3. The initialization dependency’s return value is written to the corresponding helper-result slot.

The first iteration uses region `0x0DC` and result slot `0x0D8`. The second uses region `0x1B0` and result slot `0x1AC`.

There is no visible check for a zero storage-request result before initialization. There is also no check of the initialization result before storing it: a zero return overwrites the previous slot value with zero.

### 3. Unconditional virtual callback

After both lookup iterations, the function invokes the input object’s virtual entry at displacement `0x134`, passing the input object as receiver.

This callback occurs regardless of:

- Whether resolution was enabled.
- Whether resolution succeeded.
- Whether either table-driven lookup succeeded.
- Whether either stored initialization result was zero.

These statements assume earlier calls return normally.

### 4. Final conditional stage

After the unconditional callback returns, the function reads the full 32-bit field at `0x044`.

If it is zero, the function returns without the final lookup or final virtual callback.

If it is nonzero:

1. The function reads the flags at `0x048`.
2. If bit zero is clear, it skips the additional lookup.
3. If bit zero is set, it reads a global argument and calls F-8967e4fb3bba with that argument.
4. If this lookup returns nonzero, it calls F-197837317fab with the input object and the returned value.
5. It invokes the input object’s virtual entry at displacement `0x0D4`, passing the input object as receiver.

The final virtual callback is controlled by the earlier nonzero read of `0x044`, not by lookup success. It still occurs when bit zero is clear or the additional lookup returns zero.

The function does not recheck `0x044` after F-8967e4fb3bba or F-197837317fab. Changes to that field during those calls therefore do not cancel the already-selected final callback.

## Side effects

Direct writes by this function are limited to the following visible object fields:

- `0x0B4`: cleared when resolution is enabled, then replaced by route results.
- `0x0D8`: replaced only when the first table-driven lookup succeeds.
- `0x1AC`: replaced only when the second table-driven lookup succeeds.

All dependency calls may have additional effects that are not established here. No dependency is assumed to preserve object fields, virtual tables, global state, or memory contents.

The function requests storage up to twice. It contains no explicit release operation, including when replacing a nonzero helper-result slot. The evidence does not establish whether initialization returns the requested storage, transfers ownership, releases anything internally, or returns a different object.

Observable ordering includes:

- Resolution and binding precede the shared lookup-argument and mode capture.
- First-slot initialization, if reached, precedes the second lookup.
- Both lookup attempts precede the unconditional callback.
- The final-stage gate and flags are read after that callback.
- Any successful final lookup is passed to F-197837317fab before the final virtual callback.

There is no visible transaction, exception handling, error reporting, or rollback.

## Dependencies

| Opaque function ID | Established use |
|---|---|
| F-8f9bd59e3ec5 | Receives the resolved nonzero value and input object. Return value ignored. |
| F-5710a8628cba | Performs each table-driven lookup using context, shared argument, and selected entry. Zero skips initialization. |
| F-8157a180d901 | Receives a 68-byte storage request. Its result is forwarded without a null check. |
| F-f8beff1830ad | Receives requested storage, an embedded-region address, and a nonzero lookup result. Its return is stored. |
| F-8967e4fb3bba | Conditional final lookup using a global argument. Zero suppresses only the subsequent dependency call. |
| F-197837317fab | Receives the input object and a nonzero final-lookup result. Return value ignored. |

Virtual targets have no supplied opaque function IDs. They are identified here only by receiver role and virtual-table byte displacement: `0x05C`, `0x0E4`, `0x014`, `0x134`, and `0x0D4`. Equal displacements on different receivers do not establish identical targets or behavior.

## Acceptance examples

These are conditional contract examples derived from static evidence, not executed tests. Any claim about an unchanged field assumes dependencies do not independently modify it.

1. **Resolution disabled.** With `0x0BC` equal to zero and `0x0B4` initially nonzero, resolution dispatches and F-8f9bd59e3ec5 are skipped. This function does not clear `0x0B4`. Both table-driven iterations still occur.

2. **Associated route succeeds.** With resolution enabled, a nonzero associated object, a nonzero first dispatch result, and a nonzero second dispatch result, the latter value is stored at `0x0B4` and passed to F-8f9bd59e3ec5. The fallback route is not invoked.

3. **Associated route fails.** If either associated-route result is zero, zero is stored at `0x0B4` before fallback begins. A nonzero fallback result replaces it and enables binding; a zero fallback result does not.

4. **Fallback argument changes during dispatch.** If the initial gate is nonzero but the `0x05C` callback changes `0x0BC` to zero, the subsequent `0x0E4` fallback dispatch receives zero. The resolution stage is not restarted or canceled.

5. **First lookup fails, second succeeds.** The first helper-result slot receives no direct write. The second success requests 68 bytes, initializes using region `0x1B0`, and stores the returned value at `0x1AC`.

6. **Initialization returns zero.** Following a successful table-driven lookup, an initialization return of zero is stored into that iteration’s result slot. The next stage still runs if the call returns normally.

7. **Mode is stable but selector changes.** If first-iteration work changes `0x0B8` and `0x26C`, the second iteration retains the original mode displacement but uses the new selector byte.

8. **Unconditional callback enables the final stage.** If the callback at `0x134` changes `0x044` from zero to nonzero, the final stage is entered using the post-callback state.

9. **Final lookup is disabled or unsuccessful.** With `0x044` nonzero, the callback at `0x0D4` occurs whether bit zero is clear or the enabled lookup returns zero. F-197837317fab occurs only for a nonzero lookup result.

## Uncertainties

The following remain **documented_only**:

- Domain meanings of the object fields, table entries, constants, and callbacks.
- Exact calling convention of the associated object’s `0x0E4` dispatch.
- Dependency ownership, allocation, release, failure, and mutation semantics.
- Valid selector values and actual table extent.
- Sizes and layouts of the embedded regions beyond their starting addresses.
- Whether callback targets may throw, terminate, reenter, or fail to return.
- Whether the requested 68 bytes correspond to a complete object.
- Any concurrency guarantees or synchronization requirements.

No assumptions about these matters are required to describe the visible branches and direct writes. Safe runtime use may require additional constraints unavailable in the supplied evidence.

## Evidence

Established behavior comes solely from static inspection of the supplied function representation and dependency-ID mapping. The evidence supports the field widths, byte offsets, branch conditions, call order, constants, direct stores, and absence of visible checks described above.

The representation does not provide dependency bodies, a complete object layout, virtual-target identities, or authoritative type and ownership contracts. Names such as “binding,” “lookup,” and “helper” describe apparent roles rather than proven application semantics.

No runtime tests have been performed. The acceptance examples are conditional predictions of the visible behavior, not observed execution results.
