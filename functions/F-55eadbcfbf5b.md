<!-- Trusted provenance: F-55eadbcfbf5b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-55eadbcfbf5b

## Purpose

F-55eadbcfbf5b obtains a record reference through a lookup dependency, performs a preliminary state operation, and chooses between local completion and an indirect operation supplied by the context.

Local completion processes a previously captured name-region address, compares two dependency results, updates record-related state, stores a record reference in the context, and publishes five outputs. Delegation forwards the original eight arguments and returns the indirect operation’s result.

The lookup receives the address of a function-local pointer slot. The supplied evidence does **not** establish that this slot remains unchanged after lookup. Later dependencies could replace its contents through a retained address. Consequently, this contract distinguishes:

- **Current record reference:** the contents of the lookup-output slot when a subsequent operation uses it.
- **Captured name address:** an address obtained before name inspection and retained for later use.
- **Captured second-byte base:** a record base obtained before name inspection and retained for the second special-name comparison.

These references need not continue to identify the same record. Capturing an address does not capture the bytes stored there.

“Record,” “name,” and “selection” are descriptive interpretations. Application-level meanings are not established.

## Interface

### Arguments

| Position | Established role |
|---|---|
| 1 | Pointer to a mutable context containing 32-bit fields and a dispatch-table reference. |
| 2 | Opaque 32-bit lookup argument; also forwarded to the indirect operation. |
| 3 | Opaque 32-bit value supplied to two local-completion dependencies and the indirect operation. |
| 4 | Destination for a one-byte output. |
| 5 | Destination for another one-byte output. |
| 6 | Destination for a 32-bit output. |
| 7 | Destination for another 32-bit output. |
| 8 | Destination for a third 32-bit output. |

The represented address model is 32-bit: context elements occupy four bytes, and local completion stores a record address in a four-byte context field. Wider-address portability is not established.

The function visibly validates neither pointer validity nor alignment, storage extent, overlap, or initialization of the lookup-output slot after a nonzero lookup result. Successful execution requires every access actually reached to be valid. Dependencies can impose additional preconditions.

### Result

The result is a 32-bit value:

- `4` when lookup returns zero.
- `1` when the preliminary dependency returns zero.
- `0` after local completion.
- The indirect dependency’s unchanged 32-bit result after delegation.

These result classes overlap numerically. A delegated result of zero does not establish that this function published its local outputs.

### Context and dispatch layout

Offsets are byte offsets.

| Context offset | Direct use |
|---|---|
| `0x00` | Dispatch-table reference, read when delegation is reached. |
| `0x44` | Address of an embedded region supplied to lookup. |
| `0x60` | Four-byte value read for each call to F-a1b718452f55; overwritten with the then-current record reference during local completion. |

The indirect callable is obtained from offset `0x40` in the dispatch table. Dependency-required extents of the embedded context region and dispatch structure are unknown.

### Record-relative layout

These offsets describe accesses relative to whichever reference supplies the base at that operation, not necessarily one stable record.

| Offset | Use and reference source |
|---|---|
| `0x000` | Leading-byte inspection and possible clear, using the current record reference after the first F-a1b718452f55 call. |
| `0x004` | Preliminary dependency argument, derived from the current record reference. |
| `0x011` | Captured name address, established after the preliminary operation. Used for name inspection, the first special-name byte comparison, and F-658e9ad4fb43. |
| `0x012` | Second special-name byte, accessed through the separately captured second-byte base. |
| `0x120` | Flag byte accessed through the current record reference separately during routing and output publication. |
| `0x128` | Four-byte output source, accessed through the current record reference at publication. |
| `0x24C` | Second argument region for F-f5c1d5368335, derived from the current record reference when preparing that call. |
| `0x254` | First argument region for F-f5c1d5368335 and F-3cfbcf6c86eb, derived anew for each call. |
| `0x25C` | Conditional four-byte destination, derived from the current record reference after the second F-a1b718452f55 call. |

A record base used for the `0x25C` write must provide accessible storage through offset `0x25F`. This does not establish that all other accesses use that allocation or require the same extent. Dependencies may access additional storage.

## Behavior

### 1. Lookup

F-20f4a7792eb2 receives, in order:

1. The context address advanced by `0x44` bytes.
2. Argument 2.
3. The address of the function-local lookup-output pointer slot.

A zero result immediately produces return value `4`. No later dependency is invoked, and this function has made no direct context, record, or output write. Lookup itself may have side effects.

Every nonzero result continues, without a visible distinction between positive and negative values.

The slot’s address has been exposed to lookup. Neither a prohibition on retaining that address nor a guarantee against subsequent replacement of the slot is established.

### 2. First context-dependent operation and preliminary record handling

The function reads the four-byte context field at `0x60` and calls F-a1b718452f55 with the context and that value. It retains the dependency’s 32-bit result. No result value causes immediate termination.

The leading-byte inspection occurs **after** this call and uses the then-current lookup-output slot. If the dependency replaced the slot, the inspection concerns the replacement record, not necessarily the record initially supplied by lookup.

The leading byte determines the next action:

- If nonzero, the function clears the leading byte and continues without calling F-3f3c73758f26.
- If zero, it calls F-3f3c73758f26 with the current record reference advanced by `0x004`. A zero result produces return value `1`; any nonzero result continues.

There is no explicit leading-byte clear following a successful F-3f3c73758f26 call. That dependency may mutate memory or replace the lookup-output slot. Subsequent captures use the slot after that call.

Returning `1` does not imply unchanged state: lookup and the first F-a1b718452f55 call have already occurred.

### 3. Address capture and name-based routing

After preliminary handling, the function first captures a record base for the possible second-byte check, then captures the address at current-record offset `0x011` for name inspection. No dependency call intervenes between these captures.

Without an intervening external modification, both captures refer to the same record. They remain separate captured values; neither is recomputed from the slot after name inspection.

F-1cb585febd84 receives the captured name address. Its result is interpreted as an unsigned 32-bit value:

- Result `1`: read the byte at the captured name address. Delegate if it equals `0x2E`.
- Result `2`: read the byte at the captured name address. If it equals `0x2E`, read offset `0x012` from the captured second-byte base and delegate if that byte also equals `0x2E`.
- Any other result: neither explicit special-name byte comparison is performed.

For result `2`, a first byte other than `0x2E` prevents the second-byte read.

The byte comparisons happen after F-1cb585febd84 returns. They observe the bytes then present at the captured addresses, not a snapshot taken before the call. If the dependency replaces the lookup-output slot, these comparisons still use the captured addresses.

A matching special-name condition selects delegation without reading flag bit `0x08`.

The interpretation as handling “.” and “..” is inferred intent. The established rule combines the dependency result with particular byte comparisons; no terminator or actual name length is independently verified.

### 4. Flag-based routing

If the special-name checks do not select delegation, the function reads the flag byte at offset `0x120` through the **then-current** lookup-output slot.

- Bit `0x08` set: delegate.
- Bit `0x08` clear: complete locally.

Thus, name comparisons may inspect a captured earlier record while flag routing examines a replacement record. The function does not reconcile their identities.

### 5. Local completion

The following sequence is established:

1. Call F-658e9ad4fb43 with argument 3 and the **captured name address**. Replacing the lookup-output slot does not retarget this argument.
2. Read context offset `0x60` again and call F-a1b718452f55 with the context and this freshly read four-byte value.
3. Compare the second 32-bit result with the retained first result. If unequal, write the first result to offset `0x25C` through the **current lookup-output slot after the second call**. If equal, omit this write.
4. Call F-f5c1d5368335 with addresses at current-record offsets `0x254` and `0x24C`, respectively.
5. Call F-3cfbcf6c86eb with an address newly derived from the current record reference at offset `0x254`, followed by argument 3.
6. Store the then-current record reference as a four-byte address in context offset `0x60`.
7. Read the current record’s flag byte, extract bit `0x10`, normalize it to zero or one, and write one byte through argument 4.
8. Read the current record’s flag byte again, extract bit `0x02`, normalize it to zero or one, and write one byte through argument 5.
9. Read four bytes at current-record offset `0x128` and write that value through argument 6.
10. Write a four-byte zero through argument 7.
11. Write a four-byte zero through argument 8.
12. Return `0`.

The comparison is equality of 32-bit results; no signed ordering or arithmetic difference is used.

Each later dependency may replace the lookup-output slot. In particular:

- F-658e9ad4fb43 may affect which record receives the conditional write.
- The second F-a1b718452f55 may itself replace that destination reference.
- F-f5c1d5368335 may change the record used to form the subsequent F-3cfbcf6c86eb argument.
- F-3cfbcf6c86eb may change the reference subsequently stored in the context.

Arguments already formed for a dependency are not retroactively changed by a slot replacement during that call.

Output reads use the lookup-output slot, not a fresh lookup through context offset `0x60`. The earlier context assignment does not establish an immutable selection for later reads. Output writes can themselves affect subsequent observations if valid aliasing exists.

No visible result check on F-658e9ad4fb43, F-f5c1d5368335, or F-3cfbcf6c86eb changes this sequence.

### 6. Delegation

The function reads the dispatch-table reference from the context, obtains the callable at table offset `0x40`, and invokes it with the original eight arguments in their original order. Its 32-bit result is returned unchanged.

The local completion sequence is skipped. Earlier dependencies and any leading-byte clear have nevertheless already occurred.

The dispatch reference is obtained when delegation is reached; it is not established as a snapshot from entry. Earlier dependencies may have changed it.

## Side effects

### Direct writes and ordering

Direct writes are limited to:

- A conditional leading-byte clear.
- A conditional four-byte write at current-record offset `0x25C`.
- A four-byte record-reference assignment to context offset `0x60`.
- Two byte outputs and three four-byte outputs.

There is no visible rollback. A leading-byte clear remains performed even if routing later delegates.

Writes need not all affect one record. Their destinations depend on the slot contents at their respective uses.

### Aliasing

No overlap rejection is visible. Subject to valid accesses in the target environment:

- Coincident byte-output destinations receive the second byte assignment last.
- Coincident word-output destinations receive the final zero assignment last.
- A first byte output overlapping the flag byte can alter the value used for the second byte output.
- An earlier output overlapping the `0x128` field can alter the later word read.
- If a valid alias permits an output write to alter the lookup-output slot, later record-relative accesses follow the altered slot.

These are ordering consequences, not evidence that arbitrary overlap is intended.

### Ownership and lifetime

No direct allocation or release is visible. Dependencies’ allocation, release, retention, and ownership effects remain unspecified.

Every captured address and every subsequently loaded record reference must remain usable for the access that consumes it. Replacing the lookup-output slot does not remove the later requirement to access the captured name region.

Local completion stores a record reference in the context, but ownership transfer and required subsequent lifetime are unknown. The exposed lookup-output slot belongs to this invocation; the evidence establishes no valid post-return use of its retained address.

## Dependencies

| Opaque dependency | Established interaction |
|---|---|
| F-20f4a7792eb2 | Receives the context subregion, argument 2, and the address of the lookup-output slot. Zero terminates with `4`. |
| F-a1b718452f55 | Called after lookup and again on local completion with the context and a fresh read of context offset `0x60`; 32-bit results are compared. |
| F-3f3c73758f26 | Receives current-record offset `0x004` when the inspected leading byte is zero. Zero terminates with `1`. |
| F-1cb585febd84 | Receives the captured name address. Results `1` and `2` enable special-name comparisons. |
| F-658e9ad4fb43 | Receives argument 3 and the retained captured name address. |
| F-f5c1d5368335 | Receives current-record regions `0x254` and `0x24C`, in that argument order. |
| F-3cfbcf6c86eb | Receives a newly derived current-record region `0x254` and argument 3. |
| Indirect dependency, opaque ID unresolved | Selected through context dispatch-table offset `0x40`; receives all original arguments and supplies the propagated result. |

No dependency is assumed to preserve globals, context contents, record contents, ownership, or the exposed lookup-output slot. The record-region arguments to F-f5c1d5368335 are formed before that call; their relative evaluation order is not independently established.

## Acceptance examples

These are conditional examples, not executed tests. Assume valid storage and no mutations or overlap beyond those stated.

1. **Lookup miss.** F-20f4a7792eb2 returns zero. The function returns `4`, invokes no subsequent dependency, and directly writes none of the outputs.

2. **Preliminary failure.** Lookup succeeds; the first F-a1b718452f55 returns; the current leading byte is zero; F-3f3c73758f26 returns zero. The function returns `1` without name inspection.

3. **Replacement before preliminary handling.** Lookup initially supplies record A. The first F-a1b718452f55 replaces the slot with record B. B’s leading byte is inspected; A’s leading byte does not control this branch.

4. **Captured single-dot match despite replacement.** The name address is captured from A. F-1cb585febd84 replaces the slot with B and returns `1`. If the captured byte in A is `0x2E`, delegation occurs without testing B’s flag bit `0x08`.

5. **Captured double-dot match.** Both captures refer to A. Name inspection replaces the slot with B and returns `2`. If A’s bytes at `0x011` and `0x012` are both `0x2E`, delegation occurs regardless of B’s name bytes.

6. **Mixed captured-name and current-flag routing.** Name inspection receives A’s captured name address, replaces the slot with B, and returns `3`. No explicit dot-byte comparison occurs. B’s bit `0x08` controls routing. If clear, F-658e9ad4fb43 still receives A’s captured name address.

7. **Mutation without replacement.** Name inspection changes the byte at the captured name address to `0x2E` and returns `1`. The subsequent comparison observes that changed byte and selects delegation.

8. **Later replacements during local completion.** The retained first comparison result is `7`. The second F-a1b718452f55 replaces the slot with C and returns `9`; C’s `0x25C` field receives `7`. F-f5c1d5368335 then replaces the slot with D; F-3cfbcf6c86eb receives D’s `0x254` region. If that dependency replaces the slot with E, E’s reference is stored in context offset `0x60`.

9. **Equal comparison results.** Both F-a1b718452f55 calls return `7`. The conditional `0x25C` write is omitted, even if the slot now identifies a different record.

10. **Stable local output publication.** During publication, the current record remains unchanged, its flag byte is `0x12`, and destinations do not overlap source storage. Both byte outputs receive `1`; the first word output receives the current `0x128` field; the final two word outputs receive zero.

11. **Delegated status collision.** The indirect dependency returns `4`. The outer result is `4`, despite successful lookup and execution of preliminary handling.

## Uncertainties

The following remain **documented_only**:

- Whether any dependency actually retains or replaces the lookup-output slot.
- Application meanings of records, flags, arguments, outputs, and comparison results.
- Whether F-1cb585febd84 reports a length, and the name representation, encoding, termination, and readable extent.
- Dependency-specific storage, alignment, ownership, retention, and lifetime requirements.
- Which dependencies mutate shared state, release storage, or invalidate captured addresses.
- Synchronization, concurrency, and reentrancy requirements, including changes between separate address captures or accesses.
- Fault, exception, or other failure behavior outside the two explicit zero-result checks.
- The indirect dependency’s identity and output guarantees.
- Endianness, address-wrap behavior, and ABI details beyond the represented widths.

A fixed-record account is valid only under an additional guarantee that the lookup-output slot remains unchanged. That guarantee is not established here. No retry, rollback, or recovery mechanism is visible.

## Evidence

This contract derives solely from static examination of the supplied function representation and opaque dependency mapping.

Established evidence includes argument widths, byte offsets, explicit call order, conditional reads and writes, pre-inspection address captures, subsequent uses of the lookup-output slot, output publication order, and return propagation.

Record-slot replacement is a permitted possibility without a preservation guarantee, not an observed runtime event. Dot-name handling and record selection are inferred intent; the explicit byte conditions and assignments define the established behavior.

No runtime tests have been performed. The acceptance examples state conditional observations, not measured results.
