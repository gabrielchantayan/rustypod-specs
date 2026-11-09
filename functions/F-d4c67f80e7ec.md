<!-- Trusted provenance: F-d4c67f80e7ec; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d4c67f80e7ec

## Purpose

F-d4c67f80e7ec populates caller-supplied destination storage through one of three routes:

- Population from a lookup result’s payload.
- Population from a resource-resolution result.
- Population from a fallback source when a signed position check rejects the request.

Every represented normal completion returns the original destination pointer. The function does not visibly allocate the destination.

“Collection,” “payload,” “resource key,” and “tag” describe structural roles. Their application-level meanings are not established. In particular, the destination’s first word receives a value read from one of two global locations, not an established immutable type constant.

This contract distinguishes behavior established by the supplied static representation from inferred intent and unresolved details. No runtime tests have been performed.

## Interface

### Arguments

The interface has four 32-bit argument values:

| Argument | Established role |
|---|---|
| Destination | Pointer to writable result storage. Direct accesses cover its first two 32-bit words; a dependency also receives this pointer. |
| Collection | Pointer to storage containing a signed range bound and an embedded lookup-related field. |
| Selector | Scalar bit pattern compared with `0x00006D1C`. No dereference is visible. |
| Position | Scalar bit pattern interpreted as a signed 32-bit integer for range classification. No dereference is visible. |

Pointer-shaped presentation of the selector and position does not establish a requirement that they designate readable storage.

### Visible layouts

All offsets are byte offsets in the represented 32-bit address model.

| Object | Offset | Width | Established access |
|---|---:|---:|---|
| Destination | `+0` | 32 bits | Receives a value read from the synthesized-tag or copy-style-tag global. |
| Destination | `+4` | 32 bits | Cleared before population, except for exact source/destination identity on a copy-style path. |
| Collection | `+132` | Field address | Passed to F-f9dd96be3e32. |
| Collection | `+136` | 32 bits | Read as the signed position bound for nonnegative positions. |
| Lookup result | `+4` | 32 bits | Read for comparison or copy-style population. |
| Lookup result | `+9` | 8 bits | Tested for zero on the exact-selector path. |
| Fallback source | `+4` | 32 bits | Read for copy-style population when source and destination differ. |

Eight writable destination bytes cover the direct result stores, but do not establish the complete storage requirements of F-151ab4bfe553. Full object extents, alignment requirements, and dependency-specific invariants remain documented_only.

### Global inputs

The representation distinguishes global storage serving these roles:

- An ordinary resource key.
- An exact-selector base resource key.
- A comparison input.
- A synthesized tag.
- A copy-style tag.
- An unresolved pointer-bearing fallback source.

The two tag locations are distinct inputs to destination initialization. Their contents are not established as immutable, distinct from each other, or stable across calls. Unknown dependencies must not be assumed to preserve them.

### Return and preconditions

Normal completion returns the original destination pointer. There is no separate success flag or error code.

This guarantee assumes that dependencies return normally and that every access actually performed is valid. Valid destination storage is required on every represented path. Collection storage is directly read only for nonnegative positions. Source and lookup-result validity requirements depend on the chosen path.

## Behavior

### Position classification

Position is interpreted as a signed 32-bit integer.

A negative position selects fallback without a visible read of the collection bound.

For a nonnegative position, the function reads the 32-bit value at collection offset `+136` and interprets it as signed. Lookup is selected exactly when:

**0 ≤ position < signed bound.**

Consequently:

- Position zero is accepted when the bound is positive.
- Equality with the bound is rejected.
- A position greater than the bound is rejected.
- A zero or negative bound accepts no position.
- A bound with its high bit set is negative under the represented interpretation.

The position is not visibly supplied to the lookup dependency or used in address arithmetic to select an entry. Its established role is range gating. How the returned entry relates to that position is unresolved.

### Accepted-position lookup and resource-key capture

For an accepted position, F-f9dd96be3e32 receives the address at collection offset `+132`.

After lookup returns, the function reads and captures the ordinary resource-key global. This read occurs before the selector-specific branch, including when the exact-selector branch will not use the captured value.

The selector then determines whether processing uses the exact-selector route or ordinary comparison.

### Exact-selector route

When the selector is exactly `0x00006D1C`, F-634a31ef9eb5 is not called.

The function reads the exact-selector base resource-key global before testing the byte at lookup-result offset `+9`.

- A zero byte selects that initially read base value.
- A nonzero byte causes another read of the base-key global and selects that later value minus five.

Thus, the nonzero case is not established as subtraction from the first captured base value. No dependency call is visible between these reads, but the representation does not establish synchronization or a single snapshot of mutable storage.

The subtraction uses a 32-bit integer representation. No explicit overflow check is visible. Behavior when mathematical subtraction by five is outside the signed 32-bit range remains documented_only; this contract does not assign either wrapping or a failure response.

After selecting the key, the function calls F-9d0ecc5f6adf without visible arguments, then calls F-c0b020cd3882 with that return value and the selected key.

The result of F-c0b020cd3882 becomes the population input. Only after F-c0b020cd3882 returns does the function read the synthesized-tag global. It writes that read value to destination offset `+0`, clears destination offset `+4`, and calls F-151ab4bfe553 with the destination and population input.

The synthesized tag is therefore not captured at entry, before lookup, or before resource resolution. Changes to its global storage by preceding dependencies can affect the value written.

### Ordinary-selector comparison

For any other selector, F-634a31ef9eb5 receives:

- The 32-bit payload read from lookup-result offset `+4`.
- The value read from the comparison-input global.

Both reads occur before the comparison call. Their relative evaluation order is not established by the supplied representation.

Only zero versus nonzero in the comparison result affects the branch.

#### Nonzero result: copy-style population

After F-634a31ef9eb5 returns nonzero, the function reads the copy-style-tag global and writes that value to destination offset `+0`.

This tag read occurs after comparison, not before it. A comparison dependency that changes the tag global can therefore affect the written value.

The destination is then compared for exact pointer identity with the lookup result.

If they are equal, the function returns without clearing destination offset `+4` and without calling F-151ab4bfe553. The first-word tag write still occurs.

If they differ, the function clears destination offset `+4`, reads the payload at lookup-result offset `+4`, and calls F-151ab4bfe553 with the destination and that payload.

This second payload read occurs after comparison and after the destination clear. It need not equal the payload supplied to comparison: the dependency or intervening aliased stores may have changed it.

#### Zero result: ordinary synthesis

A zero comparison result selects resource resolution using the ordinary key captured immediately after lookup.

The function calls F-9d0ecc5f6adf, then calls F-c0b020cd3882 with its return value and that captured key. The ordinary key is not visibly reread after comparison. A later change to its global location does not substitute a different argument.

After F-c0b020cd3882 returns, the function reads the synthesized-tag global, writes that value to destination offset `+0`, clears destination offset `+4`, and invokes F-151ab4bfe553 with the resolved population input.

The key and tag have different capture times: the key precedes comparison; the synthesized tag follows resource resolution. No preservation of the tag global by comparison or either resolution dependency is assumed.

### Rejected-position fallback

A rejected position bypasses lookup, accepted-path resource-key capture, selector-specific flag inspection, comparison, and resource resolution.

The representation selects a fallback source through an unresolved pointer-bearing value. After selecting that source, the function reads the copy-style-tag global and writes its value to destination offset `+0`.

The tag read and write precede the source/destination identity check and any fallback payload read.

If the fallback source equals the destination, the second destination word is left unchanged and population is skipped.

Otherwise, the function clears destination offset `+4`, reads the fallback payload at source offset `+4`, and calls F-151ab4bfe553 with that payload.

The fallback source’s concrete identity, initialization, lifetime, and validity remain unresolved.

## Side effects

### Direct memory effects and ordering

Every represented normal path writes destination offset `+0` using a global-backed tag read.

The important ordering constraints are:

| Route | Established ordering |
|---|---|
| Either synthesized route | Resource resolution returns; synthesized tag is read and written; destination `+4` is cleared; population is called. |
| Nonzero comparison | Comparison returns; copy-style tag is read and written; identity is checked; if distinct, destination `+4` is cleared, source `+4` is read, and population is called. |
| Fallback | Fallback pointer is selected; copy-style tag is read and written; identity is checked; if distinct, destination `+4` is cleared, source `+4` is read, and population is called. |

The values written as tags may vary across invocations or across paths. Selecting a tag location does not establish a fixed value or a final object type.

F-151ab4bfe553 may further modify the destination. The caller-visible final contents are therefore not limited to the direct initialization stores described here.

### Aliasing

Only exact source/destination identity suppresses copy-style population. There is no visible general overlap check.

For distinct but overlapping objects, the destination tag write or second-word clear can affect the subsequent source payload read. Snapshot copying and overlap-safe behavior are not established.

No independent direct write through the collection or lookup-result arguments is shown, but destination aliasing can make destination stores affect those objects. Aliasing with global storage can also affect subsequent reads according to their established timing.

Synthesized routes do not have the copy-style identity shortcut.

### Ownership and dependency effects

No allocation, release, ownership transfer, reference-count operation, or cleanup protocol is established by this function’s direct operations.

In particular:

- Lookup results are not visibly released here.
- Resource-resolution results are not visibly released after population.
- Clearing destination offset `+4` does not establish release of prior contents.
- There is no visible cleanup before overwriting an already populated destination.
- Dependency names and argument shapes do not establish purity or preservation of global state.

Arbitrary reuse of initialized destination storage is not guaranteed safe without additional lifetime information.

### Errors and incomplete execution

No explicit checks are shown for an invalid destination, invalid collection on the nonnegative-position path, null lookup result, invalid fallback source, or failed resource resolution.

No explicit error status or rollback is present. If population fails to return normally, the tag write and second-word clear have already occurred on paths that call it. Those stores do not imply recovery or a valid completed result.

## Dependencies

| Opaque ID | Established role | Unresolved contract |
|---|---|---|
| F-f9dd96be3e32 | Receives collection address `+132`; returns the entry-like pointer used afterward. | Selection policy, failure representation, memory effects, ownership, lifetime. |
| F-634a31ef9eb5 | Receives an entry payload and comparison-global value; zero/nonzero return selects synthesis or copying. | Comparison meaning, global mutations, payload mutations, failure behavior. |
| F-9d0ecc5f6adf | Called without visible arguments before resource resolution. | Return type, ownership, lifetime, memory and global effects. |
| F-c0b020cd3882 | Receives the preceding return value and selected resource key; returns population input. | Resolution semantics, failure representation, ownership, memory and global effects. |
| F-151ab4bfe553 | Receives the initialized destination and population input. | Complete destination requirements, final layout, allocation, release, failure behavior. |
| F-b9acea4b06ad | Identifies the shared copy-style region in the supplied representation. | Whether it is independently callable; no separate call to it is established here. |

No recursive call to F-d4c67f80e7ec is established.

Unknown dependency effects are material to this contract. Resource-key capture before a dependency and tag reads after a dependency must not be collapsed into an assumed entry-time snapshot.

## Acceptance examples

These are conditional criteria derived from static evidence, not observed test results. Assume valid required storage and normally returning dependencies unless stated otherwise.

1. **Negative position:** Position `−1` selects fallback regardless of selector. The collection bound is not visibly read, and lookup is not called.

2. **Accepted lower boundary:** Bound `1` and position `0` cause lookup with collection address `+132`. Position zero is not independently rejected.

3. **Rejected upper boundary:** Bound `4` and position `4` select fallback. Bound bits `0x80000000` with position `0` also select fallback because the bound is signed.

4. **Exact selector, zero flag:** For an accepted position, selector `0x00006D1C`, and zero at lookup-result byte `+9`, resolution receives the initially read base key. Comparison is skipped. The synthesized tag is read only after resolution returns.

5. **Exact selector, nonzero flag:** With the same selector and byte `+9` equal to `0xFF`, the key comes from the second base-global read minus five, provided subtraction is within the established signed range. No particular nonzero byte value has special significance.

6. **Captured ordinary key versus late tag:** Suppose lookup is followed by capture of ordinary key `100`, comparison returns zero and changes that key global to `200`, and resource resolution changes the synthesized-tag global to `9`. Resolution still receives key `100`; destination `+0` receives `9` if no further change intervenes before the tag read.

7. **Copy-style tag changed by comparison:** Suppose comparison returns nonzero and leaves the copy-style-tag global containing `12`. The subsequent destination tag write uses `12`, not an assumed earlier value. With distinct pointers, population receives a payload read after comparison and after the destination clear.

8. **Exact self-source:** If a nonzero-comparison lookup result is the destination itself, the newly read copy-style tag is still written. Destination `+4` is not cleared, and population is not called.

9. **Fallback tag timing and identity:** After fallback source selection, the copy-style tag is read and written before identity is checked. If that source equals the destination, the tag write occurs but the payload is neither cleared nor read for population.

10. **Overlapping distinct storage:** If distinct source and destination pointers overlap, acceptance must follow the specified store-before-payload-read order. Preserving the pre-store payload is not a requirement established by this representation.

## Uncertainties

The following remain documented_only:

- Contents and application meanings of both tag globals; whether their values ever coincide.
- Global resource-key values and comparison data.
- Fallback-source identity, lifetime, and validity.
- Full object layouts, alignment, and dependency-required storage extent.
- How position relates to the entry returned by lookup.
- Arithmetic behavior outside the signed subtraction range.
- Dependency failure, allocation, release, ownership, and mutation contracts.
- Thread safety, reentrancy, and synchronization of global reads.
- Safety requirements for initialized or overlapping destination storage.
- Low-level stack and register reconstruction details surrounding the shared copy-style region.

Inferred intent is limited to an object-like result, copy-style payload propagation, and key-based resource synthesis. These interpretations do not establish application semantics, immutable tags, or ownership rules.

## Evidence

Established behavior comes from the supplied static representation: signed range comparisons, 32-bit field accesses, the byte flag test, exact selector equality, dependency arguments, zero/nonzero branching, global-read placement, destination-store order, identity checks, and return-pointer flow.

The synthesized tag is sourced from a global read after F-c0b020cd3882 returns. The copy-style tag is sourced from a different global read after nonzero comparison or fallback-source selection. Neither read establishes immutability or preservation by earlier dependencies.

Notices about omitted unreachable regions and unresolved stack or register expressions limit reconstruction confidence. They do not independently establish additional public arguments, error handling, ownership behavior, or dependency calls.

No dependency inspection, implementation execution, or runtime tests were performed. Acceptance examples are conditional behavioral obligations, not reported observations.
