<!-- Trusted provenance: F-c8ad6e35cfe2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c8ad6e35cfe2

## Purpose

F-c8ad6e35cfe2 uses an entry index to decode a selector and two boundaries, conditionally obtains a payload, publishes selected metadata into an object, and invokes object-associated processing.

On the successful nonempty path, a signed descriptor field controls whether an additional indirect operation runs and how the payload view supplied to the final callback is adjusted. After that callback returns normally, the function invokes a release-associated dependency with the original payload address.

These roles are inferred from usage. The established contract is the access sequence, branching, argument construction, status handling, and direct writes described below. A zero return does not prove that a payload was delivered: an empty range ordinarily returns zero without allocation, metadata publication, or either indirect invocation.

## Interface

### Inputs and result

The function takes two 32-bit integer inputs and returns a 32-bit integer:

- **Object address:** interpreted as an address of readable and writable object storage.
- **Entry index:** used in signed integer arithmetic to select an encoded entry. No explicit sign or bounds check is present.

The return value is the final value of a local status slot. Several dependency results are assigned to this slot, and the allocation-associated dependency receives its address. Consequently, a later dependency may affect the returned status through an alias exposed by that dependency.

All offsets below are byte offsets. Explicit object, state, table, and descriptor scalar accesses are four bytes wide. Address-bearing values use the supplied representation’s 32-bit quantities. The declaration and width of the separate global datum used by the optional indirect operation are not established.

### Object layout

| Offset | Observed use |
|---|---|
| `+0x004` | Associated-state address, captured at entry |
| `+0x560` | Destination for the descriptor’s signed adjustment field |
| `+0x564`, `+0x568` | Destinations for two metadata-table words |
| `+0x56C` | Written with zero |
| `+0x570` through `+0x584` | Six consecutive four-byte descriptor-word destinations |
| `+0x5DC` | Indirect callback target, read immediately for the final invocation |

### Associated-state layout

| Offset | Observed use |
|---|---|
| `+0x064` | Opaque allocation/release context, captured initially |
| `+0x088` | Interface address, captured initially; an indirect target is later read at its `+0x010` |
| `+0x11C` | Addend in the initial position |
| `+0x120` | Stride component; low eight bits used by the first decoder; later full-width cursor increment |
| `+0x124` | Other stride component; low eight bits used separately by the second and third decoders |
| `+0x130` | Descriptor-table base, read after metadata-table publication |
| `+0x134` | Position base, read separately for initial positioning and payload loading |
| `+0x13C` | Base of an eight-byte-per-entry metadata table |
| `+0x148` | I/O-associated address, captured initially |

The I/O-associated object supplies a four-byte value at `+0x020`. This initializes a local cursor passed by address to the decoder. Its encoding and underlying storage contract are unknown.

### Preconditions and arithmetic domain

Ordinary execution requires valid storage and callable targets for every access that the selected path reaches. The function does not validate object size, table extent, selector range, target validity, allocation results independently of status, or payload adjustment bounds.

Position calculations, table indexing, length subtraction, and payload adjustment use the displayed 32-bit integer quantities. No explicit overflow checks are present. Defined machine-level wraparound for overflowing signed expressions is not established by the representation.

## Behavior

### Initial capture and preparation

The function reads the associated-state address from the object, then captures the I/O-associated address. It initializes its status slot to zero and captures the allocation/release context and interface address.

It calculates a stride by adding the state words at `+0x120` and `+0x124`. It calls **F-e40a3ceddab2** with:

- The captured I/O-associated address.
- The stride multiplied by the entry index, plus the state values at `+0x134` and `+0x11C`.

The result is assigned to the status slot. If nonzero, the function returns that status without further dependency calls or direct metadata writes.

If zero, it calls **F-d036abe1e536** with the captured I/O-associated address and twice the cached stride. This result replaces the status. A nonzero result ends the path immediately.

The stride is not recomputed after the first call. Later changes to the stride fields do not alter the argument to the second call. The representation does not establish an ordering between operand reads within a single arithmetic expression.

### Decode the selector and range

After both preparation calls return zero, the function initializes a local cursor from the captured I/O-associated object’s `+0x020` field.

It performs these operations in order:

1. Calls **F-d8b79fc4b172** with the cursor’s address and the low eight bits of a fresh four-byte read at state offset `+0x120`. The result is the selector.
2. Calls the same dependency with the same cursor address and the low eight bits of a fresh read at `+0x124`. The result is the starting boundary.
3. Adds a fresh, full-width read at `+0x120` to the cursor value left by the second call.
4. Calls the decoder again, using the cursor address and the low eight bits of another fresh read at `+0x124`. The result is the ending boundary.
5. Subtracts the starting boundary from the ending boundary to obtain the payload length.
6. Calls **F-9787e15c337d** with the captured I/O-associated address.

The decoder’s width argument is masked to the range 0–255. This does not establish that every such value is supported.

All three decoder calls may modify the same cursor. The explicit full-width cursor increment occurs after the second call and before the third. Decoder results are used directly; there is no separate decoder-error check.

The result of F-9787e15c337d is ignored. Its invocation precedes the zero-length decision. Whether it commits cursor state, releases a buffer, synchronizes I/O, or serves another purpose is unknown.

### Empty range, allocation, and loading

If the length is zero, the function skips allocation, loading, metadata publication, both indirect operations, and explicit release. It reaches the final return with the status left by the successful preparation calls, which is zero in the visible path.

Any nonzero length enters allocation, including a negative length under the displayed signed interpretation.

The function calls **F-3ce007b692cc** with the captured allocation context, the computed length, and the address of the local status slot. Its return value is treated as the payload address.

The function then examines the status slot:

- If nonzero, it returns that value without loading or calling the explicit release dependency.
- If zero, it proceeds without a separate null-address check.

Passing the status address exposes a writable alias. No supplied dependency contract says that F-3ce007b692cc cannot retain or expose it for use by later dependencies.

Loading uses **F-871a9759e22a**, receiving:

- The captured I/O-associated address.
- A fresh read of state field `+0x134`, plus the decoded starting boundary.
- The returned payload address.
- The computed length.

The load result replaces the status slot. A nonzero result ends processing without direct metadata publication and without the function’s explicit release call. This establishes the absence of that call, not the allocation’s ultimate ownership or lifetime.

### Publish metadata

Only a zero load result reaches direct metadata publication.

The function reads the metadata-table base from state offset `+0x13C` and selects an entry at eight bytes times the decoded selector. It then:

1. Reads the first entry word and writes it to object offset `+0x564`.
2. Reads the second entry word and writes it to `+0x568`.
3. Writes zero to `+0x56C`.

The selected entry address is already calculated before those stores. The second source word is read after the first destination store; source/destination aliasing can therefore matter.

Next, it freshly reads the descriptor-table base at state offset `+0x130` and selects a descriptor with a stride of `0xFC` bytes.

The descriptor transfers are:

| Descriptor offset | Object offset |
|---|---|
| `+0x0D8` | `+0x570` |
| `+0x0DC` | `+0x574` |
| `+0x0E0` | `+0x578` |
| `+0x0E4` | `+0x57C` |
| `+0x0E8` | `+0x580` |
| `+0x0EC` | `+0x584` |

The read/write order is more specific than a bulk copy:

- Descriptor words at `+0x0DC`, `+0x0E0`, and `+0x0E4` are read and retained first, in that order.
- The `+0x0D8` word is then read and stored to object `+0x570`.
- The three retained words are stored to `+0x574`, `+0x578`, and `+0x57C`, in order.
- Descriptor `+0x0EC` is read and retained.
- Descriptor `+0x0E8` is read and stored to object `+0x580`.
- The retained final word is stored to `+0x584`.

Finally, the function reads the signed descriptor word at `+0x004` and writes it unchanged to object `+0x560`. Earlier writes can affect later reads if the storage aliases. No disjointness guarantee is supplied.

These writes are not rolled back if later processing fails.

### Optional processing and final callback

The signed descriptor word determines the effective adjustment:

- **Negative:** effective adjustment becomes zero; the interface-selected indirect operation is skipped. The negative value remains stored at object `+0x560`.
- **Nonnegative:** effective adjustment is the descriptor value, and the interface-selected indirect operation is invoked. Zero takes this branch.

On the nonnegative branch, the function obtains the third argument from the source’s global datum. This global read occurs **after metadata publication and before the indirect invocation**. It is not captured with the initial context.

The optional operation uses the target at `+0x010` of the previously captured interface address and receives:

- The original payload address.
- The complete payload length.
- The value obtained from that global datum.

The supplied representation does not establish the global datum’s declaration, width, value, mutability, or semantic interpretation. It must not be treated as a fixed constant or necessarily as an address. No stronger ordering between the global evaluation and target lookup within the call expression is established.

The optional operation’s result is ignored. Its side effects and failure mechanisms are unknown.

The function next reads the callback target at object `+0x5DC` and invokes it with:

- The original object address.
- The original payload address plus the effective adjustment.
- The complete length minus the effective adjustment.

The callback target is read after the optional operation returns. Changes to that field by the optional operation can therefore affect target selection.

Neither an adjustment equal to the length nor an adjustment greater than the length suppresses the callback. The latter can produce a negative remaining length under ordinary signed arithmetic. No bounds check is present.

### Release and returned status

The final callback’s result is assigned to the local status slot. The function then calls **F-bf7242c98d06** with the captured allocation context and the original, unadjusted payload address.

The release dependency’s return value is ignored. After it returns normally, the function returns the **resulting value of the status slot**.

Returning the callback result unchanged requires a noninterference assumption: the release dependency must not modify that slot through an alias retained or exposed by F-3ce007b692cc. The supplied contracts provide no such guarantee.

No cleanup for exceptions, nonlocal exits, or abnormal termination is shown.

## Side effects

Direct object writes occur only after successful allocation and loading, at the specified fields from `+0x560` through `+0x584`. Dependencies may perform additional writes or other effects; these are not constrained by the direct-write list.

Observable dependency interactions include preparation, decoder cursor mutation, allocation, loading, optional whole-payload processing, adjusted-view callback processing, and release-associated handling of the original payload address.

The function does not visibly transfer ownership to either indirect operation. The release-associated call receives the original address even when the final callback receives an interior address. Retaining payload access beyond that call requires an independently established lifetime guarantee.

The associated-state address, I/O-associated address, allocation context, and interface address are captured initially. Later reads still use the captured state address, even if the object’s state pointer changes. Fields and table contents read later may reflect intervening mutations.

The global datum and final callback target are not initially captured. Unknown dependencies must not be assumed to preserve them, reachable storage, or the exposed status slot.

## Dependencies

| Opaque ID or indirect selection | Observed contract |
|---|---|
| **F-e40a3ceddab2** | Receives I/O-associated address and calculated position; result replaces status; zero permits continuation |
| **F-d036abe1e536** | Receives I/O-associated address and twice the cached stride; result replaces status |
| **F-d8b79fc4b172** | Three calls with a mutable cursor address and masked eight-bit argument; returns selector or boundary |
| **F-9787e15c337d** | Called after boundary subtraction, including for an empty range; result ignored |
| **F-3ce007b692cc** | Receives context, length, and status-slot address; returns payload address; may retain or expose the status alias absent a contrary contract |
| **F-871a9759e22a** | Receives I/O-associated address, payload position, payload address, and length; result replaces status |
| **F-bf7242c98d06** | Receives captured context and original payload address after callback return; result ignored; status noninterference is not established |
| Interface-selected indirect dependency; ID unavailable | Called for nonnegative adjustment with original address, full length, and the then-obtained global datum value |
| Object-selected indirect dependency; ID unavailable | Receives object, adjusted address, and reduced length; result is assigned to status before release |

No dependency bodies or independent IDs for the indirect targets are supplied.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Arithmetic examples assume representable results and valid accessed storage.

1. **First preparation fails.** If F-e40a3ceddab2 returns 7, the function returns 7 without the second preparation call, decoding, allocation, or direct metadata writes.

2. **Second preparation fails.** If the first call returns zero and F-d036abe1e536 returns 9, the function returns 9 without decoding or allocation.

3. **Equal boundaries.** Both preparation calls return zero and the decoded boundaries are equal. All three decoder calls and F-9787e15c337d occur. Allocation and metadata publication do not occur; the visible status remains zero.

4. **Allocation reports failure.** For a nonzero length, F-3ce007b692cc leaves status 12. The function returns 12 without loading or explicitly releasing its returned address.

5. **Loading fails.** Allocation leaves zero status, but loading returns 14. The function returns 14 without metadata publication or explicit release.

6. **Negative adjustment.** A successfully loaded descriptor contains −1. The object receives −1 at `+0x560`; the optional operation and its global-datum evaluation are skipped. The final callback receives the original address and full length.

7. **Zero adjustment.** The optional operation runs with the complete payload and the global datum obtained on that branch. The callback receives the original address and full length.

8. **Interior view.** With length 20 and adjustment 4, the optional operation receives length 20. The final callback receives the address advanced by four bytes and length 16. Release receives the original address.

9. **Empty callback view.** With length 20 and adjustment 20, the callback is still invoked with the end address and length zero, subject to address-validity requirements.

10. **Callback result without interference.** If the callback returns 23 and release does not modify the status slot, the function returns 23. Published metadata is not restored.

11. **Release changes exposed status.** If the callback returns 23, but release uses a status alias exposed by allocation to write 31 before returning, the function returns 31. The release dependency’s own return value remains irrelevant.

12. **Late callback selection.** If the optional operation changes object `+0x5DC`, the subsequent callback lookup uses the changed target, assuming the write and target are valid.

## Uncertainties

The following remain **documented_only**:

- Overflow behavior for signed sums, products, subtraction, indexing, and address adjustment.
- Decoder encoding, byte order, supported widths, cursor advancement, and error signaling.
- Valid selector and boundary ranges, including treatment of negative nonzero lengths.
- Validity requirements for zero-status allocation results and excessive adjustments.
- Allocation ownership, release semantics, and reclamation on failure paths.
- Retention or exposure of the status-slot address and resulting dependency interference.
- The optional operation’s global datum: declaration, width, value, mutability, and meaning.
- Indirect-operation effects, ownership requirements, and abnormal-exit behavior.
- Aliasing between object, state, tables, payload, and dependency-accessible storage.
- Status-code meanings beyond the visible zero/nonzero continuation decisions.

## Evidence

Established claims derive from the supplied static representation: explicit access widths and offsets, branch conditions, dependency arguments, access and call ordering, status assignments, and final status retrieval.

“Position,” “payload,” “metadata,” “descriptor,” “allocation,” and “release” are inferred role labels, not independently verified dependency semantics.

No runtime tests have been performed. No dependency implementations, independent global declaration, or independently verified machine execution were supplied.
