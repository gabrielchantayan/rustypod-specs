<!-- Trusted provenance: F-2d353fe6cfb2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2d353fe6cfb2

## Purpose

F-2d353fe6cfb2 conditionally copies selected state from a source record into a destination record. Some differences trigger immediate dependency calls; others contribute to two change masks that control a later notification. Several writes contribute to neither mask.

The behavior distinguishes four categories:

- A 32-bit field at `0x68`, whose modification triggers two immediate calls.
- Individually tracked fields whose differences contribute to change masks.
- A five-word group that is rewritten as a unit when any member differs and then triggers one call.
- Three bits at `0x8F`, copied through unconditional writes after the initial gate succeeds.

A notification is therefore not a complete indication that destination memory changed. Conversely, a successful gate can lead to destination writes even when every compared value matches.

**Inferred intent:** the records appear to represent state associated with a handle, with selected changes requiring additional processing. Application meanings of the fields, handles, and dependent operations are not established.

This contract describes static evidence. No runtime tests have been performed.

## Interface

### Arguments and result

The function accepts two arguments and has no explicit return value.

| Argument | Established use |
|---|---|
| Source record address | Supplies byte, 16-bit, and 32-bit values at fixed offsets. |
| Destination record address | Supplies and receives selected record fields. Its first 32-bit word supplies a handle. The address is also passed to F-ebd0ee53699f. |

The supplied representation uses 32-bit integer words and 32-bit address-bearing values. All record offsets in this contract are byte offsets.

No record size, version, or validity indicator is supplied. Paths that reach the synchronization operations require the accessed source and destination locations to be readable and the written destination locations to be writable. The highest accessed field occupies offsets `0x104` through `0x107`. This does not establish that every intervening byte is accessed.

### Initial destination handling

If the destination address is nonnull, its first 32-bit word is captured as the initial handle. If the destination is null, the initial handle is zero.

F-06f4ceb6a4f2 receives that initial handle. A zero result ends processing before the subsequent direct record accesses. A nonzero result permits processing to continue.

This is not an unconditional null-destination guard. If the gate returns nonzero for a null destination, subsequent destination dereferences are attempted. There is no explicit source-null check.

### Accessed fields

| Offset | Width | Direct role |
|---|---:|---|
| `0x00` | 32 bits | Destination handle; captured at entry and reread at selected later call sites. |
| `0x14` | 8 bits | Only bit `0x04` is conditionally copied. |
| `0x60`, `0x64` | 32 bits each | Individually tracked values. |
| `0x68` | 32 bits | Value with immediate dependent-call behavior. |
| `0x7E` | 8 bits | Individually tracked value. |
| `0x82` | 16 bits | Individually tracked value. |
| `0x8F` | 8 bits | Bits `0x02`, `0x04`, and `0x08` copied through three writes. |
| `0x90`, `0x98`, `0xA0`, `0xA8` | 32 bits each | Individually tracked values. |
| `0xF4`, `0xF8`, `0xFC`, `0x100`, `0x104` | 32 bits each | Group compared conditionally and, on a mismatch, rewritten in full. |

The byte and 16-bit fields are compared for equality and copied at their stated widths. Their signed application-level interpretations are unknown. Equality and copying do not require choosing such an interpretation.

## Behavior

### Entry gate and retained handle

Two internal 32-bit change masks begin at zero. This contract calls them **primary** and **secondary**; these are descriptive names, not application terminology.

The initial handle is captured before F-06f4ceb6a4f2 is called. If that dependency returns zero, the function returns without any subsequent direct record synchronization or additional dependency call.

The gate itself may have side effects. Nothing establishes that it preserves records, global data, or the destination handle.

The initially captured handle is retained for the five-word-group call described below. That call does not substitute a later handle value.

### Field at `0x68`

The source and destination 32-bit values at `0x68` are compared. If equal, this stage performs no write or call.

If different:

1. The source value is read again and written to destination `0x68`.
2. The current destination handle is read and passed to F-fd61d89535ad together with `1`.
3. F-ebd0ee53699f is called with a value formed from a newly read destination handle plus `0x54`, an argument obtained through the data reference at `0x0806AD70`, and the destination address.

Neither change mask is modified by this stage.

The assignment rereads the source rather than necessarily using the value observed during comparison. The two calls use separate destination-handle reads, separated by a dependency invocation. Their handle values can therefore differ.

The data reference at `0x0806AD70` is evaluated for the F-ebd0ee53699f call after the preceding F-fd61d89535ad invocation. Its argument value is not established as fixed, immutable, or distinct from the argument obtained through the other notification data reference. The relative evaluation order of arguments within this call is not established.

### Individually tracked fields

The function processes the following fields in order:

| Order | Offset | Width | Contribution on difference |
|---:|---|---:|---|
| 1 | `0x7E` | 8 bits | Secondary `0x00000002` |
| 2 | `0x90` | 32 bits | Primary `0x20000000` |
| 3 | `0x98` | 32 bits | Primary `0x40000000` |
| 4 | `0xA0` | 32 bits | Secondary `0x00400000` |
| 5 | `0xA8` | 32 bits | Secondary `0x00800000` |
| 6 | `0x60` | 32 bits | Primary `0x00400000` |
| 7 | `0x64` | 32 bits | Primary `0x00800000` |
| 8 | `0x82` | 16 bits | Primary `0x10000000` |

For each field, a difference causes the sampled source value to be written to the destination and the indicated bit to be included in its mask. An equal field is not written and contributes no bit. Contributions accumulate by bitwise OR.

Each conditional write uses the source value already sampled for that field’s comparison; there is no additional source read for that write. The field’s conditional write precedes processing of the next field.

For `0x7E` and `0x82`, the source sample precedes the destination comparison read. For the six 32-bit fields in this table, the destination sample precedes the source sample.

These operations occur after any calls associated with `0x68`. They observe the records as they then exist, not necessarily their entry contents.

### Five-word group

The group contains the five 32-bit fields at `0xF4`, `0xF8`, `0xFC`, `0x100`, and `0x104`.

The comparison sequence is conditional:

- `0xF8` is compared first.
- `0xFC` is compared only if `0xF8` matches.
- If those two match, `0x104` and then `0x100` are compared. Both comparisons occur, even if `0x104` differs.
- `0xF4` is compared only if the preceding four fields match.

If every comparison matches, no group field is written and no group-related call occurs.

If any compared field differs, all five fields are refreshed. The established access order is:

1. Read source `0xF8`, then write destination `0xF8`.
2. Read source `0xFC`, then write destination `0xFC`.
3. Read and retain source `0x104`.
4. Read source `0x100`, then write destination `0x100`.
5. Write the retained value to destination `0x104`.
6. Read source `0xF4`, then write destination `0xF4`.
7. Call F-fd61d89535ad with the entry-captured handle and `1`.

The refresh rereads source fields rather than reusing comparison samples. It is not an atomic snapshot. The group contributes nothing to either change mask.

### Bit `0x04` at `0x14`

Only bit `0x04` of the source and destination bytes is compared. Other differing bits at this offset do not trigger an update.

If that bit differs, the function rereads the destination and source bytes and writes a destination byte containing:

- Bit `0x04` from the update-time source read.
- All other bits from the update-time destination read.

Secondary mask bit `0x00000040` is then included. The comparison and update use separate reads, so the written bit need not reflect the source bit seen during comparison.

### Three writes at `0x8F`

Every path that passes the gate and reaches this stage performs three destination-byte writes, whether or not the relevant bits already match.

The destination byte is captured once. Subsequent writes preserve unaffected bits from that captured byte:

1. The first write replaces bit `0x02` using a source-byte read.
2. The second replaces bits `0x02` and `0x04`, retaining the first source sample’s bit `0x02` and obtaining bit `0x04` from another source-byte read.
3. The third replaces bits `0x02`, `0x04`, and `0x08`, retaining the first two sampled bits and obtaining bit `0x08` from a third source-byte read.

No fresh destination read supplies the unaffected bits for the second or third write.

With stable, nonoverlapping records, the final byte is the captured destination byte with mask `0x0E` replaced from the source. With mutation or overlap, the three selected source bits may reflect different states.

These writes do not change either mask.

### Final dependent calls

If both masks remain zero, the function returns after the `0x8F` writes.

Otherwise, F-fd61d89535ad is called with the current destination handle and `1` if either condition holds:

- The secondary mask intersects `0xFF3FFFFF`.
- The primary mask intersects `0x9FFDFFFF`.

For the bits this function can generate, changes limited to `0x90`, `0x98`, `0xA0`, and `0xA8` do not cause this additional call. Any tracked change at `0x7E`, `0x60`, `0x64`, `0x82`, or bit `0x04` of `0x14` does.

Every nonzero-mask path then calls F-ebd0ee53699f with:

- A value formed from a newly read destination handle plus `0x54`.
- An argument obtained through the data reference at `0x0805D3F4`.
- The destination address.
- A pointer into local stack storage.

The second argument is evaluated at this final call site, after all earlier calls and any conditional pre-notification F-fd61d89535ad call. No preceding dependency is assumed to preserve the referenced data.

The references at `0x0806AD70` and `0x0805D3F4` are different symbolic data references. This does not establish that their supplied argument values differ. Their contents, types, mutability, and equality are unknown.

The stack-referenced payload’s extent, contents, and layout are not established. In particular, the existence of two masks does not prove their placement or accessibility through that pointer.

## Side effects

### Memory and state

Direct writes target only the destination locations described above. There is no visible direct write through the source address, but overlapping records can make destination writes affect subsequent source reads.

Writes are incremental. There is no visible rollback, transaction boundary, or restoration on failure.

Calls may alter records, handles, global data, or other state. The function’s direct write set does not constrain effects inside dependencies.

### Ownership and lifetime

No explicit allocation or release operation is visible in this function. This does not establish whether dependencies allocate, release, retain, or take ownership of anything they receive.

No ownership transfer is visibly specified. F-ebd0ee53699f receives the destination address and, on the final path, a local-stack pointer. Retention of that stack pointer beyond the function’s lifetime is not established as safe.

### Arithmetic and failure conditions

Offsets and the handle-derived `+0x54` operation are expressed in a 32-bit representation. No range, overflow, or address-validity check is visible. The evidence does not justify specifying portable signed-overflow behavior or guaranteeing a valid result.

The only dependency result used for branching is the initial gate result. Other results are not checked. No explicit error result, retry, exception handler, or partial-update recovery is visible.

## Dependencies

| Opaque ID | Established call-site behavior |
|---|---|
| F-06f4ceb6a4f2 | Receives the initial handle. Zero stops subsequent processing; nonzero permits it. Other effects are unknown. |
| F-fd61d89535ad | Receives a handle and `1`. Used after a `0x68` update, after a group refresh, and conditionally before the final notification. Its result is unused. |
| F-ebd0ee53699f | Immediate call has three visible arguments; final call has four. Receives a handle-derived value, a call-site data-reference value, the destination address, and, for the final call, a stack pointer. Complete signature and argument semantics are unknown. |

Neither notification data reference is a dependency function ID. They identify the two argument sources, not proven constants or distinct notification kinds.

## Acceptance examples

These are conditional static expectations, not executed tests.

**For examples 2–9, F-06f4ceb6a4f2 must return nonzero.** Unless an example states otherwise, also assume valid nonoverlapping records, stable source contents, normal dependency returns, and dependencies that do not modify either record. Those assumptions do not establish that dependencies preserve the notification argument data references.

1. **Gate returns zero.**  
   Processing ends after the gate. No later direct record access, synchronization write, or dependency call occurs. Effects inside the gate remain unknown.

2. **Gate succeeds; all compared values and copied bits match.**  
   Three writes still occur at destination `0x8F`. Neither F-fd61d89535ad nor F-ebd0ee53699f is subsequently called.

3. **Gate succeeds; only `0x68` differs.**  
   The word is updated. F-fd61d89535ad runs, followed by the immediate F-ebd0ee53699f call using the argument obtained through `0x0806AD70` at that call site. No final notification occurs because both masks remain zero.

4. **Gate succeeds; only `0x90` differs.**  
   The word is updated and primary bit `0x20000000` is recorded. The final F-ebd0ee53699f call occurs using the argument obtained through `0x0805D3F4`. No pre-notification F-fd61d89535ad call occurs.

5. **Gate succeeds; only `0x60` differs.**  
   The word is updated. F-fd61d89535ad is called before the final F-ebd0ee53699f call.

6. **Gate succeeds; only one group member differs.**  
   All five group fields are rewritten in the specified order. F-fd61d89535ad receives the initial handle. This group difference alone causes no final notification.

7. **Gate succeeds; only bit `0x04` at `0x14` differs.**  
   That bit is replaced while the other destination bits are preserved. The additional F-fd61d89535ad call and final F-ebd0ee53699f call both occur.

8. **Gate succeeds; only bits under `0x0E` at `0x8F` differ.**  
   The three writes synchronize those bits. They cause no dependency call by themselves.

9. **Gate succeeds; the destination handle changes before a required group-refresh call.**  
   Relax the unchanged-destination assumption for this handle mutation. The group call still receives the entry-captured handle. Calls that explicitly reread destination `0x00` instead use the value present at their respective reads.

A further conditional consequence applies when both notification call sites execute: equality of the two data-reference values is compatible with the evidence, as is inequality. A dependency-induced change before either evaluation may affect that call’s argument.

## Uncertainties

The following remain **documented_only**:

- Application meanings of the fields, handles, masks, and dependent operations.
- Contents, types, mutability, and equality of the values obtained through the two notification data references.
- Final stack-payload contents, size, layout, and lifetime requirements.
- Dependency mutation, reentrancy, ownership, retention, and failure behavior.
- Whether the gate always rejects a zero handle.
- Supported overlap, concurrency, alignment, and address-space conditions.
- Exact argument-evaluation ordering within individual call expressions.
- Byte order and any external serialization format.
- Overflow behavior and validity of handle-derived addresses.
- Whether external observers depend on the intermediate `0x8F` writes.

These unknowns must not be resolved by assuming a particular object model, notification framework, or global-state preservation rule.

## Evidence

Established evidence is limited to the supplied static representation: access widths and offsets, retained versus repeated reads, conditional branches, ordered writes, mask constants, call arguments, and call-site data references.

“Synchronization,” “tracked change,” and “notification” describe the visible operation patterns; they are not established application terminology.

No dependency implementation, runtime trace, execution result, or test evidence was supplied. No runtime tests have been performed.
