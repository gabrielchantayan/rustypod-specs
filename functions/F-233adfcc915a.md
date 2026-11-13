<!-- Trusted provenance: F-233adfcc915a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-233adfcc915a

## Purpose

F-233adfcc915a consumes a sequence of values through opaque reader dependencies, selects configuration records using one decoded byte, and incrementally populates a caller-provided destination.

Acceptance depends on reader checks, a restricted normalized category, configuration-controlled branches, two nested decoders, and a final configuration predicate. Failure does not roll back previously written output.

**Established:** the supplied source exposes the call order, destination widths and offsets, configuration accesses, and explicit acceptance predicates.

**Inferred intent:** the routine appears to decode a configuration-dependent record from a bounded input representation. The meaning of that representation and the decoded fields is not established.

**Unknown:** dependency implementations, input encoding, complete destination schema, and the meaning of the successful return word.

## Interface

The routine has six inputs and a 32-bit result.

| Input | Exposed representation | Contract role |
|---|---|---|
| First input | 32-bit opaque value | Copied into local working state; the address of that copy is passed to reader, checker, and nested-decoder dependencies. |
| Second input | 32-bit opaque value | Copied into another local word. Its relationship to the working-state address is not independently specified. |
| Destination | Pointer presented as addressing 16-bit units | Receives byte, 16-bit, and 32-bit writes, plus writes through nested decoders. Offsets below are byte offsets. |
| Configuration | 32-bit address-like input | Supplies two table bases and configuration-controlled decisions. |
| Optional-decoder control | 32-bit integer | Zero skips F-84fc449f1feb; any nonzero value enables it. |
| Mode | 32-bit integer | Equality to 5 enables one additional read. The unchanged value is also passed to F-84fc449f1feb when enabled. |

The result is:

- **Failure:** the 32-bit pattern `0xffffffff`.
- **Accepted path:** the current value of a local 32-bit word initialized to zero.

There is no explicit assignment to that return word after initialization in the visible routine. However, the supplied material does not establish the memory extent or mutation contract of the working state passed to dependencies. Consequently, “every successful return is zero” is not an unconditional guarantee.

### Memory preconditions

The caller must provide valid memory for all accesses taken on the selected path.

- Direct destination writes reach byte offset `0x43`, requiring at least 68 accessible bytes when that path is taken.
- Nested decoders receive destination addresses at offsets `0x20` and `0x2c`. Their actual writable extents are unknown; 68 bytes is not a proven sufficient allocation for the complete operation.
- Configuration contains readable 32-bit table-base values at offsets `0xd0` and `0xd4`.
- Selected table records must support the accesses described below.
- No explicit null, bounds, alignment, overlap, or capacity checks appear.

Byte order and required alignment are not specified by the supplied material. Multi-byte stores use the execution environment’s native memory representation.

### Direct destination layout

Names in this table describe behavior, not established application semantics.

| Byte offset | Width | Direct-write condition |
|---|---:|---|
| `0x00` | 16 bits | Initial decoded value |
| `0x02` | 8 bits | Category, then possible normalization |
| `0x03` | 8 bits | Table selector |
| `0x04` | 32 bits | Configuration-width value |
| `0x08` | 8 bits | Always initialized to zero after the preceding value passes its check; possibly replaced by a one-unit read |
| `0x0a` | 16 bits | Mode equals 5 |
| `0x0c` | 16 bits | Selected secondary-record byte `0x09` equals zero |
| `0x10` | 32 bits | Conditional value in the preceding branch |
| `0x14` | 32 bits | Secondary-record byte `0x09` equals one and byte `0x0b` equals zero; may be overwritten by another read |
| `0x1c` | 8 bits | Primary-record byte `0x3e` is nonzero |
| `0x1d` | 8 bits | Later category-dependent flag |
| `0x1e` | 8 bits | Configuration default, optionally replaced |
| `0x20` | Unknown extent | F-b7c05eb4d0d1 output region |
| `0x2c` | Unknown extent | Optional F-84fc449f1feb output region |
| `0x40` | 8 bits | Trailing decoded value |
| `0x41` | 8 bits | Primary-record byte `0x3c` is nonzero |
| `0x42` | 8 bits | Same condition, and stored byte `0x41` is not 1 |
| `0x43` | 8 bits | Same condition as `0x42` |

Fields skipped by a branch are not explicitly initialized by this routine. Gaps between direct fields are likewise not cleared.

## Behavior

### Working state and reader-check discipline

The routine initializes local words with the first input, the second input, a zero-valued auxiliary state, and a zero-valued eventual accepted-path return word. A separate metadata word has no explicit initialization before its address is first supplied to a reader.

Reader operations follow a consistent observable order:

1. Invoke the selected reader.
2. Store the returned value into the destination, narrowing it when necessary.
3. Invoke F-8a872239bf2c.
4. Continue only if the checker returns nonzero.

Thus a failed check ordinarily leaves the just-written value in the destination. The checker is not a precondition checked before writing.

For F-2b01710262ca and F-6609d7dd09ad, the checker receives the current reader-metadata value. For F-d54421325734, it receives the requested width. All receive the working-state address and the address of auxiliary state.

The units of a width request and the meanings of metadata and auxiliary state are unknown.

### Initial values and category acceptance

The first F-2b01710262ca result is stored as 16 bits at destination `0x00`, then checked.

After a successful check, another result is narrowed to eight bits and stored at `0x02`. If that stored byte exceeds 4, the routine subtracts 5 and stores the normalized byte back to the same location. Normalization occurs before the associated checker call.

After the check succeeds, the normalized category must be one of:

- 0;
- 2;
- 5;
- 7.

Any other category fails immediately.

For a reader result considered only through its low eight bits, the original category bytes that pass this gate are exactly 0, 2, 5, 7, 10, and 12. This statement assumes no intervening mutation of the destination by dependencies or aliases.

### Configuration-record selection

The next F-2b01710262ca result is narrowed to eight bits, stored at `0x03`, and checked.

On success:

- Read the 32-bit table base at configuration offset `0xd4`.
- Select a primary record using the unsigned destination byte at `0x03`, with a stride of `0x40` bytes.
- Read the 32-bit table base at configuration offset `0xd0`.
- Read unsigned byte `0x01` of the selected primary record.
- Select a secondary record using that byte, with a stride of `0x300` bytes.

The largest selector-derived displacements are 16,320 bytes for the primary table and 195,840 bytes for the secondary table. Neither selector is range-checked against an actual table count.

The products themselves fit within 32 bits. Base-address addition still requires valid address arithmetic and mapped records; the source does not establish overflow handling.

The selected record addresses are retained for later use, while their individual fields are read as the execution reaches each branch. The routine does not take a complete immutable configuration snapshot.

### Main value and rejection flag

Read unsigned secondary-record byte `0x08`, add 4, and request that width from F-d54421325734. The width therefore lies between 4 and 259 inclusive.

Store the returned value as 32 bits at destination `0x04`, then check using that width.

After success, write zero to destination byte `0x08`.

If secondary-record byte `0x64` is zero:

- Request width 1 from F-d54421325734.
- Store the low eight bits at destination `0x08`.
- Check with width 1.
- Reject if the stored byte is nonzero.

A successful check does not override this explicit zero requirement.

If secondary-record byte `0x64` is nonzero, no corresponding reader call occurs and destination `0x08` remains at its explicitly written zero, absent external mutation.

### Mode-specific and secondary-record branches

If mode equals 5, consume another F-2b01710262ca value, store its low 16 bits at `0x0a`, and check.

If secondary-record byte `0x09` is zero:

- Request width equal to unsigned secondary-record byte `0x0a` plus 4.
- Store the low 16 bits of the F-d54421325734 result at destination `0x0c`.
- Check with the requested width.
- If primary-record byte `0x03` is nonzero and destination byte `0x08` is zero, consume F-6609d7dd09ad, store its 32-bit result at `0x10`, and check.

Separately, the routine reads secondary-record byte `0x09` again. If it is one, it reads secondary-record byte `0x0b`. When that byte is zero:

- Consume F-6609d7dd09ad, store 32 bits at destination `0x14`, and check.
- If primary-record byte `0x03` is nonzero and destination byte `0x08` is zero, consume another F-6609d7dd09ad value, overwrite the same 32-bit field at `0x14`, and check again.

The two writes to `0x14` are distinct observable writes. The second value replaces the first; there is no separate direct destination field retaining both.

Under stable configuration, secondary-record byte `0x09` values other than zero or one enter neither branch. No explicit rejection is attached to those other values.

### Additional flag and default replacement

If primary-record byte `0x3e` is nonzero, consume F-2b01710262ca, store its low eight bits at destination `0x1c`, and check.

Next, copy primary-record byte `0x2c` into destination byte `0x1e`.

Reload destination category byte `0x02`. If it is 0, 3, or 6:

- Request width 1 from F-d54421325734.
- Store its low eight bits at `0x1d`.
- Check with width 1.
- If stored byte `0x1d` is nonzero, consume F-2b01710262ca, replace destination byte `0x1e` with its low eight bits, and check.

Without mutation after the earlier category gate, only category 0 can reach this branch. Categories 3 and 6 are nevertheless explicit later comparisons; they must not be described as accepted initial categories.

### Nested decoders and trailing values

Invoke F-b7c05eb4d0d1 with:

- the working-state address;
- destination address `0x20`;
- the current eight-bit category at `0x02`.

A zero result rejects the operation.

If the optional-decoder control is nonzero, invoke F-84fc449f1feb with the working-state address, destination address `0x2c`, and the original mode. Its zero result also rejects the operation. A zero control skips this call entirely.

After successful nested decoding:

- Consume F-6609d7dd09ad.
- Store its low eight bits at destination `0x40`.
- Check.

If primary-record byte `0x3c` is nonzero:

- Consume F-2b01710262ca, store its low eight bits at `0x41`, and check.
- Unless destination byte `0x41` equals exactly 1, consume two F-6609d7dd09ad values in sequence.
- Store their low eight bits at `0x42` and `0x43`, respectively, checking after each store.

A value of 1 at `0x41` skips both later reads and leaves `0x42` and `0x43` untouched by these direct writes.

### Final acceptance

After every required preceding operation succeeds, the final configuration gate accepts when:

- primary-record byte `0x04` is zero; or
- primary-record byte `0x05` is below 3; or
- primary-record byte `0x05` is above 5.

Otherwise it rejects. Thus a nonzero byte `0x04` combined with byte `0x05` equal to 3, 4, or 5 causes failure, even after all decoding and output writes have completed.

The comparisons against byte `0x05` are separate source reads. Treating this as a single stable-value predicate requires that the record not change between those reads.

## Side effects

### Incremental output and failure

Output publication is nontransactional.

- Failure can occur after any checked read.
- The value associated with a failed check has already been stored.
- Normalization can produce two writes to destination `0x02` before its check.
- Configuration defaults can later be replaced.
- The first value at `0x14` can be overwritten.
- Final-gate failure can leave an otherwise fully populated destination.
- No rollback, clearing, or restoration occurs.

Consumers must not treat the destination as valid merely because some fields appear populated.

### State, aliasing, and ownership

Dependencies receive writable local-state addresses, and nested decoders receive writable destination addresses. Their additional mutations are unknown.

No allocation, release, ownership transfer, global access, or configuration write is explicit in this routine. That observation does not establish that its dependencies lack those effects.

No non-aliasing requirement is enforced. Destination writes could affect subsequent configuration or destination reads if caller-provided regions overlap. The branch descriptions assume stable, non-overlapping regions unless otherwise stated; behavior with overlapping memory must follow actual read/write order, not an imagined immutable input snapshot.

## Dependencies

| Opaque ID | Observed use |
|---|---|
| F-2b01710262ca | Produces values stored at several widths; receives working-state, metadata, and auxiliary-state addresses. Encoding and complete effects are unknown. |
| F-d54421325734 | Produces a value using an explicit width request; receives working-state and auxiliary-state addresses. Widths used here are 1 or an unsigned configuration byte plus 4. |
| F-8a872239bf2c | Post-read checker. Zero rejects; nonzero permits continuation. Receives either reader metadata or a width. |
| F-6609d7dd09ad | Produces values stored either as 32 bits or narrowed to eight bits; uses the same metadata/check pattern as F-2b01710262ca. |
| F-b7c05eb4d0d1 | Required nested decoder at destination `0x20`; zero rejects. Receives the current category. |
| F-84fc449f1feb | Optional nested decoder at destination `0x2c`; zero rejects. Receives mode unchanged. |

None of these dependencies is established to preserve globals, avoid allocation, retain no pointers, or refrain from releasing memory.

## Acceptance examples

These are conditional specification examples, not executed tests. Unless stated otherwise, assume valid memory, stable configuration, no unexpected dependency mutation, and successful checks.

1. **Normalized category acceptance.** An original category byte of 10 is stored, then replaced with 5. It passes the category gate. An original byte of 8 becomes 3 and fails that gate.

2. **Write before failed validation.** The initial reader returns a value whose low 16 bits are `0x1234`, and the checker returns zero. The routine returns `0xffffffff`; destination `0x00` has already received `0x1234`.

3. **Explicit flag rejection.** Secondary-record byte `0x64` is zero, the width-1 read stores 1 at destination `0x08`, and its check succeeds. The routine nevertheless returns `0xffffffff` because the stored byte is nonzero.

4. **Skipped mode field.** Mode is not 5. No mode-specific read occurs, and destination `0x0a` is not directly written by that branch. Optional nested decoding, if enabled, still receives that mode.

5. **Overwrite at `0x14`.** Secondary-record bytes `0x09` and `0x0b` are 1 and 0, primary-record byte `0x03` is nonzero, and destination `0x08` is zero. Two F-6609d7dd09ad reads occur. Their results are written successively to the same 32-bit destination field. If the second check fails, the second write remains.

6. **Trailing sentinel.** Primary-record byte `0x3c` is nonzero and the read at destination `0x41` produces 1. After its successful check, neither trailing read for `0x42` or `0x43` occurs.

7. **Late rejection.** All required decoding succeeds, but primary-record byte `0x04` is nonzero and byte `0x05` is 4. The routine returns `0xffffffff` without undoing output.

8. **Accepted return qualification.** The final gate accepts and the return word has remained unchanged. The result is zero. If dependency-accessible state can change that word, the returned value instead follows its resulting contents; that possibility is unresolved.

## Uncertainties

The following are **documented_only**, not promises filled in by assumed behavior:

- The input representation, encoding, cursor model, and width units.
- The exact aggregate layout and extent of local state exposed through its first address.
- Whether dependencies update the zero-initialized accepted-path return word.
- Metadata initialization requirements and whether every reader produces valid metadata on failure.
- Complete nested-decoder destination extents.
- Table allocation sizes and selector validity beyond eight-bit truncation.
- Pointer-overflow behavior, alignment requirements, and byte order.
- Thread safety, reentrancy, configuration stability, and aliasing expectations.
- Dependency ownership, pointer retention, allocation, release, global mutation, and failure-side-effect contracts.
- Semantic names for the decoded fields and application-level error categories.

No runtime tests have been performed.

## Evidence

This contract is based solely on the supplied static source body.

Established evidence includes explicit call order, direct memory-access widths and offsets, truncation points, table strides, zero/nonzero checks, exact category comparisons, conditional overwrites, and final acceptance conditions.

Interpretations such as “decoder,” “category,” and “configuration record” are descriptive labels, not independently verified domain semantics. No dependency implementation, external schema, execution trace, or runtime behavior was available.
