<!-- Trusted provenance: F-3340ee3ffdc5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3340ee3ffdc5

## Purpose

F-3340ee3ffdc5 evaluates a request using a context value, record index, category, and item. It combines category-level gating with item-mask comparisons against baseline and selected payloads. Some requests bypass payload evaluation; some selected-payload matches require an additional dependency call.

The visible returning paths produce 32-bit statuses 0 through 5. Initial-guard failures and missing required wrappers instead invoke a dependency identified as nonreturning.

“Eligibility check” is inferred intent, not an established application-level meaning. The contract below specifies observable calls, reads, arithmetic, decisions, and results without assigning undocumented meanings to categories or payload bits.

## Interface

### Arguments

| Argument | Width and interpretation | Visible use |
|---|---|---|
| Context | 32-bit value | Forwarded unchanged to selected dependencies; not directly dereferenced here. |
| Record index | 32-bit unsigned representation | Compared using signed 32-bit ordering at the initial guard; otherwise passed to dependencies and used in record-address arithmetic. |
| Category | 32-bit unsigned value | Determines a category bit, item normalization, and payload-field selection. |
| Item | 32-bit unsigned value | Passed to the item-mask dependency, possibly after normalization; also selects category 4’s payload layout. |

The signed interpretation of the initial index comparison does not change the argument’s 32-bit representation.

### Results

| Status | Condition on the reached path |
|---:|---|
| 0 | Direct delegation returns normally, the category-bit bypass applies, the baseline payload matches after allowed-mask acceptance, or the final dependency returns nonzero. |
| 1 | The category bit, dependency-produced category mask, and global category mask have an empty three-way intersection. |
| 2 | The item mask has no intersection with the applicable allowed mask. |
| 3 | The allowed mask accepts, the baseline misses, the selected payload matches, and the final dependency returns zero. |
| 4 | The allowed mask accepts, but both payload observations miss. |
| 5 | The addressed record’s first byte is the sentinel represented as signed −2. |

No normal status is established for a failed initial guard or missing required wrapper. Such paths call F-a817d8d58c10.

### Storage and preconditions

The represented address domain is 32 bits. Relevant storage comprises:

- A global root address, whose addressed object contains a 32-bit field at offset `0x04`.
- A global record-table base, with record stride `0x114` bytes.
- A global pointer to a 32-bit category mask.
- Five global 32-bit allowed-mask values, labeled A through E here. Their immutability is **not established**.

Relevant record fields are:

| Offset | Width | Use |
|---|---:|---|
| `0x00` | 8 bits | Sentinel check and later classification |
| `0x04` | 32 bits | Baseline wrapper address in ordinary resolution mode |
| `0x10` | 8 bits | Selects alternative resolution when equal to 2 |

Each wrapper contains a 32-bit payload address at offset zero. Payload access widths and offsets appear below.

Every reached memory access requires valid storage of sufficient size and lifetime. There are no visible alignment, allocation-size, table-bound, or payload-address checks. Even the initial guard requires a valid root address because its field is read before the guard is decided.

## Behavior

### Initial guard and preliminary calls

The function captures the global root address and reads its 32-bit field at offset `0x04`.

It invokes F-a817d8d58c10 if that field is zero or if the record index, interpreted as signed 32-bit, is at least 3. Thus indices 0, 1, and 2 pass when the field is nonzero. Values represented as negative signed integers also pass; this does not establish that they address valid records.

After passing the guard, the function constructs a 32-bit category bit by shifting 1 using the low eight bits of the category as the shift operand. Behavior for shift operands at least 32 remains unresolved.

The following calls occur in order:

1. F-1bc5bcdc9cca receives the initially read root field plus 4 and the record index. Its 32-bit result is retained as a lookup token.
2. F-9d8c91d0e283 receives a newly read field at offset `0x04` of the captured root address, plus 4, and the record index. Its 32-bit result supplies the category mask.

The root address is retained across the first call, but its field is reread. Consequently, mutation by F-1bc5bcdc9cca can change the second call’s argument. A change to the global root address itself does not redirect that reread.

After both calls, the function obtains the global record-table base and addresses the record at the base plus the record index multiplied by `0x114`. The represented arithmetic is 32-bit; no explicit overflow check precedes use. The additions of 4 used in the preliminary calls likewise have no visible overflow check.

The first record byte is then read. If it represents signed −2, equivalently byte value `0xfe`, the function returns 5. Both preliminary calls have already occurred.

### Direct delegation

After a nonsentinel observation, category 0 with item 1 or item `0x13` calls F-034cfcd689bb with context and record index. Normal return from that dependency leads to status 0; no dependency result is inspected.

This path precedes the global category-mask gate. It does not resolve wrappers, generate an item mask, or inspect payloads.

### Category gate and bypass

For other requests, the function reads the global category-mask pointer and the 32-bit mask addressed by it. It intersects that mask with the previously computed category bit and the result of F-9d8c91d0e283.

An empty intersection returns 1. This global mask is read after both preliminary dependencies, the sentinel check, and exclusion of direct delegation.

If the intersection is nonempty and the computed category bit intersects `0x20`, execution returns 0 without wrapper resolution or item-mask generation. Category 5 takes this bypass under the ordinary shift interpretation for its count of 5. The branch tests the computed bit, not category equality.

### Wrapper resolution

The function next reads the record byte at offset `0x10`.

**Alternative mode: that byte equals 2.**

The first record byte is read again for classification. If its observed value is 4, 5, or 6, F-7cb60db72009 is called with context, record index, and the lookup token.

If that call is skipped or returns null, F-7cb60db72009 is called with context, record index, and zero. A null fallback result invokes F-a817d8d58c10.

The successful wrapper becomes both baseline and selected wrapper. A successful token-based result avoids the zero-token call.

**Ordinary mode: that byte does not equal 2.**

The baseline wrapper address is first read from record offset `0x04`. F-7cb60db72009 is then called with context, record index, and the lookup token.

A nonnull result becomes the selected wrapper. A null result instead selects the already captured baseline wrapper. If either baseline or selected wrapper is null, F-a817d8d58c10 is invoked. A nonnull resolver result therefore does not compensate for a null baseline.

Only wrapper addresses are checked for null. Their payload addresses are not validated. The ordinary-mode baseline address is not reread after the resolver call.

### Item-mask generation

For categories 2, 9, and 12, item is reduced to its low seven bits. Other categories retain the original item value.

F-5bbdce2c8eb3 then receives the values 1, 0, and the resulting item. Its represented return value is 64 bits, subsequently treated as separate low and high 32-bit halves.

This call occurs after successful wrapper resolution and before all category-specific payload reads and reads of global mask values A through E. No pre-call snapshot of those payload fields or global mask values is established.

### Payload observations and allowed masks

The table gives each category’s payload layout and allowed mask. A mask pair is **low 32 bits / high 32 bits**. Narrow payload fields are zero-extended; single-field categories have a zero high payload half.

Each listed payload layout is read from both baseline and selected payloads.

| Category | Payload offsets and widths | Allowed mask: low / high |
|---:|---|---|
| 0 | `0x00`, `0x04`: two 32-bit fields | `0xffffffff` / A |
| 1 | `0x3a`: 16 bits | B / 0 |
| 2 | `0x38`: 16 bits | C / 0 |
| 3 | `0x08`, `0x0c`: two 32-bit fields | `0xffffffff` / `0x00000001` |
| 4, item below 64 | `0x10`, `0x14`: two 32-bit fields | `0xffffffff` / `0xffffffff` |
| 4, item at least 64 | `0x30`: 32 bits | `0x0000000f` / 0 |
| 6 | `0x42`: 8 bits | `0x0000000f` / 0 |
| 7 | `0x18`, `0x1c`: two 32-bit fields | `0xffffffff` / `0x0000001f` |
| 8 | `0x44`: 8 bits | `0x000000ff` / 0 |
| 9 | `0x20`: 32 bits | `0x01ffffff` / 0 |
| 10 | `0x46`: 8 bits | `0x0000003f` / 0 |
| 11 | `0x3e`: 16 bits | D / 0 |
| 12 | `0x28`: 32 bits | E / 0 |

For two-field layouts, the represented payload-read order is baseline low, baseline high, selected low, selected high. For single-field layouts, baseline is read before selected.

The ordering of global mask reads is category-specific:

- **Category 0:** A is read after F-5bbdce2c8eb3 returns, before baseline low, baseline high, selected low, and selected high.
- **Category 1:** B is read after both payload fields.
- **Category 2:** C is read after both payload fields.
- **Category 11:** D is read after both payload fields.
- **Category 12:** E is read after both payload fields.

All five reads therefore occur after item-mask generation, but A’s position relative to payload reads differs from B through E. These are observations of global values, not a guarantee of immutable constants.

Payload addresses are obtained through the corresponding wrappers during field accesses. A single payload-address snapshot shared by all accesses is not established.

For category 4 with item at least 64, the function reads baseline and selected fields at `0x30`, then replaces the entire generated item mask. The replacement is a 32-bit shifted value, zero-extended to 64 bits: 1 shifted by the low eight bits of the 32-bit subtraction of 64 from item. Thus its high half is zero. F-5bbdce2c8eb3 has still been called, and its possible side effects remain.

Categories without a listed payload selection retain zero payload values and a zero allowed mask. If such a request reaches this stage, it returns 2 after item-mask generation. Category 5 ordinarily exits through the earlier bypass.

### Final decision order

The allowed-mask decision comes first. If neither half of the item mask intersects the corresponding allowed-mask half, return 2.

Otherwise:

- Any baseline intersection returns 0.
- A baseline miss followed by a selected-payload miss returns 4.
- A baseline miss followed by a selected-payload intersection calls F-9c4f3d705d74 with context and record index. A zero result returns 3; a nonzero result returns 0.

Allowed-mask acceptance requires only some overlap. It does not filter the item mask before payload comparisons. A payload match may therefore use a different bit from the bit that satisfied the allowed-mask gate.

Even if both wrappers are identical, baseline and selected fields are separate observations. A baseline miss implies a selected miss only if the relevant observed contents remain unchanged.

## Side effects

No direct writes to external global, record, wrapper, or payload storage are visible. Item normalization changes only the function’s working argument value.

Dependencies may mutate storage, allocate or release memory, change object lifetimes, or fail to return normally. Their implementations and preservation guarantees are unavailable. Retaining a wrapper address across a call does not prove that its storage remains live afterward.

No allocation, deallocation, reference-count operation, or ownership transfer is visible in this function. Wrappers are consumed without a visible release; this does not establish who owns them.

There is no visible locking, retry, or consistency protocol. Multiple reads must not be treated as an atomic snapshot.

## Dependencies

| Opaque function ID | Visible interface and role |
|---|---|
| F-1bc5bcdc9cca | Receives a derived 32-bit address and record index; produces the 32-bit lookup token. |
| F-9d8c91d0e283 | Receives a separately derived address and record index; produces the 32-bit category mask. |
| F-034cfcd689bb | Receives context and record index on the direct-delegation path; its result is not inspected. |
| F-7cb60db72009 | Receives context, record index, and token or zero; produces a wrapper address or null. |
| F-5bbdce2c8eb3 | Receives 1, 0, and the possibly normalized item; produces a represented 64-bit mask. |
| F-9c4f3d705d74 | Receives context and record index after a baseline miss and selected hit; its 32-bit result is tested for zero. |
| F-a817d8d58c10 | Called without visible arguments on guard or required-wrapper failure; identified as nonreturning. |

Application semantics, ownership, mutation guarantees, and full control-flow behavior of these dependencies are documented_only unknowns.

## Acceptance examples

These are conditional consequences of static evidence, not runtime results. All examples require valid reached accesses and normal return from dependencies other than F-a817d8d58c10. Stage-specific prerequisites below are additional requirements.

1. **Absent root field.** An initially observed zero root field invokes F-a817d8d58c10 before either preliminary lookup.

2. **Index boundary.** With a nonzero initial root field, index 2 passes the initial guard and index 3 fails it. The representation of signed −1 passes that comparison but has no established valid-record guarantee.

3. **Sentinel precedence.** After the initial guard passes and both preliminary calls return, an observed first record byte of `0xfe` returns 5, including for category 0 and item 1.

4. **Direct delegation.** After passing the guard and observing a nonsentinel record, category 0 and item `0x13` call F-034cfcd689bb and return 0 on normal return, regardless of what the category-mask gate would have decided.

5. **Category rejection.** After passing the guard and sentinel check, for a request outside direct delegation, an empty three-way category intersection returns 1 without wrapper resolution.

6. **Category-5 bypass.** After reaching the category gate with category 5 and obtaining a nonempty intersection, return 0 without wrapper resolution or item-mask generation.

7. **Normalization.** If category 2 and item `0x82` reach item-mask generation—requiring the initial guard, sentinel check, category gate, and required-wrapper resolution to permit progress—F-5bbdce2c8eb3 receives item 2.

8. **Allowed-mask rejection.** If category 6 reaches payload evaluation after those preceding stages, and its generated item mask contains only low bit 4, return 2 regardless of observed payload bits. The allowed low mask is `0x0f`.

9. **Baseline acceptance.** If category 8 reaches payload evaluation, the generated mask contains only low bit 2, and the observed baseline byte contains bit 2, return 0 without F-9c4f3d705d74.

10. **Selected-only match.** If category 8 reaches payload evaluation with only low bit 2 in the item mask, the observed baseline misses, and the observed selected byte contains bit 2, call F-9c4f3d705d74. Zero yields 3; nonzero yields 0.

11. **Category-4 alternate field.** If category 4 reaches payload evaluation with item 64, read offset `0x30` and replace the generated mask with low bit 0; subsequent status depends on payload observations and possibly the final dependency. If it instead reaches that stage with item 68, the replacement is low bit 4 and status is 2.

12. **Global-mask timing.** If category 0 reaches payload evaluation and F-5bbdce2c8eb3 has changed A before returning, the subsequent read observes A after that call and before payload fields. For category 1, B is instead read after both payload observations; it is not captured before item-mask generation.

13. **Required baseline.** If ordinary wrapper resolution is reached with a null baseline address, even a nonnull resolver result leads to F-a817d8d58c10 rather than payload evaluation.

## Uncertainties

- **documented_only — Shift semantics:** Shift operands are explicitly reduced to eight bits. Behavior for 32-bit shifts by counts at least 32 is not independently established; modulo-32 behavior must not be assumed.
- **documented_only — Global mask values:** A through E have unavailable values and no established immutability or synchronization guarantee. Their read timing is specified above.
- **documented_only — Memory validity:** Record-table extent, valid index domain, payload sizes, alignment, and lifetime guarantees are unavailable.
- **documented_only — Dependency contracts:** Preservation of globals, storage ownership, allocation, release, and abnormal control flow remain unknown.
- **documented_only — Nonreturning behavior:** F-a817d8d58c10 is identified as nonreturning, but its implementation is unavailable.
- **documented_only — Representation details:** Payload byte order is not independently specified. Low and high halves here follow the represented field accesses.
- **Inferred intent:** Category eligibility, capabilities, and activation are possible interpretations, not established application semantics.

## Evidence

The supplied static function representation supports the argument widths, represented access widths and offsets, call sequence, branch precedence, mask operations, global-mask read timing, and returning statuses described here. The prior draft and reviewer feedback are not independent evidence.

No dependency implementation, runtime execution, or tests were available or performed. Acceptance examples are conditional static expectations, not observed outcomes.
