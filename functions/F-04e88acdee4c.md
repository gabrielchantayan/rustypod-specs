<!-- Trusted provenance: F-04e88acdee4c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-04e88acdee4c

## Purpose

F-04e88acdee4c coordinates a restructuring operation over an input object, its associated parent, and up to three neighboring objects. It assembles record-address and length arrays, calculates partitions, selects output objects, and calls dependencies with those partitions and parent-reference arguments.

“Restructuring,” “record,” “separator,” and “snapshot” describe apparent roles. The supplied evidence does not establish a particular storage format or the effects of dependency implementations.

Two paths exist:

- A narrowly gated shortcut returns F-81de0f51ce85’s result.
- The general path acquires neighbors, prepares workspace, calculates partitions, invokes output and parent operations, then performs cleanup calls.

There is no established transaction or rollback guarantee. Checked failures can occur after direct writes and potentially mutating dependency calls.

## Interface

### Input, result, and preconditions

The input is one 32-bit address identifying a mutable object. The result is a 32-bit status.

Explicit status checks treat zero as success. A checked nonzero result becomes the returned status after applicable cleanup. A zero workspace-allocation result selects status 7.

The function does not validate addresses, object layouts, array capacities, or arithmetic bounds. Safe execution requires accessible storage for every subsequent read and write, valid dependency results where used as addresses, and counts compatible with fixed local tables. These requirements are not enforced.

### Observed object fields

Offsets are bytes; this is an access layout, not a complete type definition.

| Offset | Width | Observed role |
|---|---:|---|
| +1 | 8 bits | Parent search-mode selector |
| +2 | 8 bits | Additional-entry count and shortcut predicate |
| +3 | 8 bits | Shortcut predicate |
| +4 | 8 bits | Shortcut predicate, prefix multiplier, format selector |
| +6 | 8 bits | Shortcut predicate and separator-mode selector |
| +8 | 8 bits | Special-record offset component |
| +0x0e | 16 bits | Relative-offset-table location |
| +0x10 | 16 bits | Neighborhood position; directly written in acquired objects |
| +0x14 | 16 bits | Primary-entry count |
| +0x18 onward | 32 bits, stride 8 | Addresses compared during provenance marking |
| +0x1c | 16 bits | Shortcut comparison field |
| +0x40 | 32 bits | Context address |
| +0x44 | 32 bits | Data-buffer address |
| +0x48 | 32 bits | Dependency token |
| +0x4c | 32 bits | Identity/comparison value |
| +0x50 | 32 bits | Parent address |

The context has an 8-bit optional-processing flag at +0x16 and 16-bit quantities at +0x1c and +0x1e.

Addresses and working arithmetic are 32-bit. Stored record lengths are 16-bit. Explicit byte and halfword loads are zero-extended where used numerically. Parent-table offsets are assembled from two bytes, high byte first. Other scalar accesses do not establish portable byte order or alignment.

## Behavior

### Entry checks and shortcut

The context and parent addresses are saved from the input before the first dependency call. The first call is F-81c906bf6e3a with the saved parent’s +0x48 token. Its visible result is 64 bits, but only its low 32 bits control rejection and return.

After a zero low word, the shortcut predicates are evaluated in short-circuit order:

1. Input byte +4 is nonzero.
2. Input byte +3 is nonzero.
3. Input byte +6 is nonzero.
4. Input byte +2 equals 1.
5. Input halfwords +0x1c and +0x14 are equal.
6. The object reached by rereading input +0x50 has a +0x4c value other than 1.
7. F-d1e995cfb2d3, called with the saved parent’s data-buffer address plus its byte +8 plus 8, returns the input’s +0x4c value.

The final comparison reads the input identity after F-d1e995cfb2d3 returns. The parent used for the sentinel comparison can differ from the saved parent if the input’s parent field changed.

If all predicates hold, F-81de0f51ce85 receives the input and saved parent; its result is returned directly. Otherwise, F-81c906bf6e3a receives the input’s +0x48 token. A nonzero result returns immediately. None of these early returns executes general-path cleanup.

### Neighborhood selection and acquisition

If saved-parent byte +1 is zero, the selected position is input halfword +0x10. Otherwise, the input’s +0x4c comparison target is cached once before searching. Each iteration rereads the parent count and, when needed, its table location and data-buffer address. F-d1e995cfb2d3 receives the table-selected record address. Its result is compared with the cached target, not a fresh input identity.

F-7962111faa24 then receives the parent token; its result is ignored. The starting position is selected-position minus one, except that a near-end selection uses parent primary-count minus two. Negative results are clamped to zero using signed 32-bit comparisons. Parent count accesses occur after the preparation call.

Up to three consecutive positions are considered. Positions below the current primary count use table-selected addresses; equality uses the special-record address; a position above the count ends acquisition. Parent fields are reread during this process.

For each candidate:

- F-d1e995cfb2d3 receives its record address.
- F-6293178c6e41 receives the saved context, returned value, an object-output location, and saved parent.
- A nonzero acquisition status enters cleanup.
- On success, the obtained object’s +0x10 field is directly overwritten with the position narrowed to 16 bits.

The sizing total adds each obtained object’s primary count, byte +2, and one. Only successful acquisitions enter the tracked acquisition count.

### Workspace and apparent snapshots

Let N be the accumulated sizing total and A be N rounded upward to a multiple of four, with 32-bit wraparound arithmetic. For allocation only, let M₀ be the context halfword +0x1c. The requested size is:

8M₀ + 6A + 264, plus A if the context flag read at this stage is nonzero.

F-391e3d30e634 receives that size. A zero result selects status 7 and cleanup.

For a nonzero workspace address:

- The address array begins at the workspace and reserves 4A bytes.
- The 16-bit length array follows and reserves 2A bytes.
- The first apparent snapshot region follows those arrays.
- Each successive region start, and then the scratch start, is calculated by adding 88 plus a fresh context +0x1c read.
- A separate flag read controls whether a provenance base is assigned. When assigned, it is scratch-start plus five times another fresh +0x1c read.

Thus there is no single stable M governing allocation, region spacing, provenance placement, and later lengths.

For each acquired object, the function makes these calls and write in order:

1. Unmapped operation A receives the region address, acquired-object address, and 84.
2. The region’s +0x44 field is directly set to region-address plus 84.
3. Unmapped operation B receives that latter address, the acquired object’s currently read data-buffer address, and a fresh context +0x1c halfword.

These arguments suggest metadata and auxiliary-data copying. Neither operation has a supplied memory-transfer contract. Whether bytes are copied, preserved, modified, or retained is unknown. Subsequent reads treat the regions as objects, but their contents are not guaranteed to reproduce acquired objects.

### Flattened arrays and separator removal calls

The prefix is cached as four times the full unsigned input byte +4. An independently evaluated mode flag is true when input bytes +6 and +4 are both nonzero.

For each apparent snapshot, primary count plus byte +2 is cached as its enumeration count. F-0f1843fcf6e2 supplies each record address; F-38c1baaad724 receives the region and that address. The returned address and low 16 bits of the returned length are directly stored in the flattened arrays.

Whenever a fresh context-flag read is nonzero, the function directly writes a provenance byte containing the source-region index. It rereads the region’s byte +2 while comparing the record address with 32-bit fields at +0x18, stride eight. A match replaces the marker with 255.

Between regions, F-38c1baaad724 receives the parent and saved intervening-record address.

In ordinary mode:

- Its full returned length advances the scratch cursor.
- The low 16 bits are stored in the length array.
- Operation B receives the scratch destination, parent-record address, and full length.
- The flattened record address is set to scratch destination plus cached prefix.
- Optional marking writes 255 when the context flag currently permits it.
- F-defc2fecf605 receives the parent, unchanged starting parent position, and original full length.
- The stored 16-bit length is reread after that call, reduced by the prefix, and stored modulo 65,536.
- If the source region’s byte +4 is zero, operation B receives the exposed flattened address, the region’s special-record address, and 4. Otherwise an adjusted stored length below four is raised to four.
- The flattened count advances.

In omit-separator mode, no flattened separator is appended; F-defc2fecf605 still receives the same parent-position and length arguments. Its result is ignored in both modes. Neither its apparent removal role nor operation B’s apparent transfers establish actual memory effects.

### Partition calculation

The threshold is a fresh context halfword +0x1e plus cached prefix minus 12.

The forward pass adds each zero-extended stored length plus two using 32-bit arithmetic. When the signed accumulator exceeds the signed threshold, it records:

- A total equal to the accumulator minus the triggering length, retaining the added two.
- A boundary equal to the triggering entry index.

It resets the accumulator. Omit-separator mode revisits the triggering entry; ordinary mode does not. A final partition is recorded even with a zero total.

The backward pass shifts boundaries toward earlier entries. A candidate shift subtracts one length plus two from the left total and adds a mode-dependent neighboring length plus two to the right total. With a nonzero right total, a signed comparison stops a shift that would make the candidate left total smaller than the candidate right total.

No bounds checks protect these calculations. There are four tracked output-object slots and fixed local partition tables. Excessive partitions, negative indices, or an entry repeatedly exceeding the threshold in omit-separator mode are not safely handled by an explicit rejection path.

### Output selection and ordering

After partitioning, the input’s data-buffer address is reread and its first byte cached.

For each partition, acquired objects are reused first. The object and its previously obtained identity enter output tracking, its acquisition slot is cleared, and F-81c906bf6e3a receives its current token. A failed check still leaves that object tracked for output cleanup.

Additional outputs are requested through F-66c60373345b with the context, object-output and identity-output locations, preceding output identity, and zero. Nonzero status aborts. An object returned by a successful call enters tracking.

F-f6da3c4a0ea5 receives each selected object and the cached input byte; its result is ignored.

Unused acquisitions receive F-ff6acbee5e19. On zero status, F-6cc778185718 receives the object and its acquisition slot is cleared. A nonzero status aborts before that clearing.

Output objects and cached identities are sorted by unsigned identity. Partition boundaries are not reordered. Equal-key ordering is not guaranteed stable.

### Partition calls and parent-separator arguments

For each sorted output, F-46634e860150 receives the object, partition entry count, address-array slice, and length-array slice. Its result is ignored; population of the object is inferred intent, not an established effect.

A fresh context-flag read controls provenance consumption. For each assigned entry, F-22ff747b8f8d is called when its marker is 255 or its source region’s current identity differs from the destination’s current identity. The entry argument is relative to the partition start. Nonzero status aborts.

The flattened cursor is then set to that partition’s boundary. Separator processing occurs only when both conditions hold:

- The current output is not the last tracked output.
- Its boundary is signed-less-than the flattened entry count.

This gate controls all separator calls, additional entry consumption, and parent-position advancement.

On entry to the gate, the initial separator length is the zero-extended stored 16-bit length plus cached prefix, using 32-bit arithmetic. Branches have this precedence:

| Destination condition | Established argument and cursor behavior |
|---|---|
| Byte +4 is zero | Operation B receives destination data-buffer plus 8, current flattened record address, and 4. The later separator call uses that record address, the initial length, and zero scratch argument. |
| Byte +4 is nonzero and omit-separator mode is active | The flattened cursor is decremented before F-200380a3070f receives the destination, preceding flattened record address, and a local temporary-area address. F-5687b5b3f1c8 receives the parent, current scratch address, zero, the length’s address, two local scalar words, three zeros, and the same length address again. Its result is ignored. The post-call length advances the scratch cursor and becomes the later separator length; the scratch argument is zero. |
| Byte +4 is nonzero and ordinary mode is active | The source address is four bytes before the flattened record address. The current scratch address is supplied separately, and scratch is advanced by the initial length. If the stored length equals four, F-38c1baaad724 then receives the parent and adjusted source address; its result replaces the update length without revising that earlier scratch reservation. |

The intermediate temporary layout and production of the two scalar words are unresolved.

F-29265b828f07 receives the parent, current parent position, selected source address, selected length, selected scratch argument, and 4. A nonzero result aborts.

After success, F-0f1843fcf6e2 receives the parent and position. F-cfb711b78946 receives its returned address and the destination object’s freshly read +0x4c identity. Optional ordinary-mode processing calls F-22ff747b8f8d on that parent position and checks its result.

Only after those checks succeed do both the flattened cursor and parent position advance. Consequently, the omit-separator/nonzero-format branch has a decrement followed by an increment; the other separator branches consume an additional flattened entry.

### Tail calls and cleanup

If bit 3 of the cached input byte is clear, operation B receives last-output data-buffer plus 8, last-region data-buffer plus 8, and 4. The resulting memory effect is unknown.

The final parent-record address is selected using current parent fields: equality between the parent position and primary count plus byte +2 selects the special record; otherwise F-0f1843fcf6e2 supplies the address. F-cfb711b78946 receives that address and the last sorted cached identity. This differs from intermediate calls using live output-object identities.

F-e95b781341a1 receives outputs in sorted order, stopping at the first nonzero status. If all succeed, it receives the parent. Only a zero parent result permits F-553007bc22b1(parent, zero), whose result becomes the return status.

General-path cleanup calls, in order:

1. F-94cb53877c15 with the workspace address, possibly zero.
2. F-6cc778185718 for each tracked acquisition slot, including cleared zero slots.
3. F-6cc778185718 for each tracked output.
4. F-6cc778185718 with the saved parent.

Cleanup results do not replace the selected status.

## Side effects

Established direct writes include acquired-object positions, workspace address and length entries, region +0x44 fields, optional provenance bytes, partition tables, cursor state, and tracking-slot changes.

Operation A and operation B are not established copies. Calls to them must not be treated as proof of snapshot preservation, scratch-record copying, or four-byte field replacement.

Likewise, calls suggesting output population, separator construction, parent updates, reference writes, or finalization establish invocation and argument flow—not those effects.

Context +0x16 is reread independently for allocation, provenance-base setup, marker production, output consumption, and parent processing. Changes across dependency calls can enable accesses to unprovisioned or uninitialized provenance storage. Changes to +0x1c can make region layout and requested auxiliary-data lengths inconsistent with allocation.

Ownership, retention, aliasing, release behavior, zero-argument tolerance, and preservation of globals or reachable fields are not established.

## Dependencies

| Opaque ID | Established call-site role |
|---|---|
| F-81c906bf6e3a | Checked token operation |
| F-81de0f51ce85 | Shortcut delegate |
| F-d1e995cfb2d3 | Record-address input; comparison/acquisition-value result |
| F-7962111faa24 | Unchecked parent-token operation |
| F-6293178c6e41 | Checked neighbor request with object-output location |
| F-391e3d30e634 | Workspace-size request; zero result handled |
| F-94cb53877c15 | Cleanup call with workspace |
| F-0f1843fcf6e2 | Object/index input; address-valued result |
| F-38c1baaad724 | Object/address input; length-valued result |
| F-defc2fecf605 | Unchecked parent/position/length call |
| F-66c60373345b | Checked additional-output request |
| F-f6da3c4a0ea5 | Unchecked object/saved-byte call |
| F-ff6acbee5e19 | Checked unused-acquisition call |
| F-6cc778185718 | Object cleanup call |
| F-46634e860150 | Unchecked partition-array call |
| F-22ff747b8f8d | Checked optional object/index call |
| F-200380a3070f | Unchecked separator-intermediate call |
| F-5687b5b3f1c8 | Unchecked separator-related call with aliased length pointers |
| F-29265b828f07 | Checked parent/position/data/length/scratch call |
| F-cfb711b78946 | Unchecked address/identity call |
| F-e95b781341a1 | Checked finalization-stage call |
| F-553007bc22b1 | Final parent/zero call |

The two unmapped operations lack supplied opaque IDs and implementation contracts; A and B distinguish their observed call sites without assigning semantics.

## Acceptance examples

These are conditional static cases, not executed tests.

1. **Early rejection:** A nonzero low word from the initial token call is returned without general cleanup.

2. **Shortcut:** When all predicates hold, only the shortcut delegate determines the result after the initial check; no general workspace is requested.

3. **Cached search target:** If a search dependency changes input +0x4c, subsequent comparisons still use the pre-search cached target.

4. **Allocation failure:** A zero workspace result selects 7. Cleanup receives zero workspace, tracked acquisitions, and saved parent.

5. **Changing context sizing:** If +0x1c changes during allocation, later spacing and auxiliary-data arguments use later reads. Matching the allocation-time value is not required by the observed execution.

6. **Changing optional flag:** Allocation with the flag clear followed by consumption with it set does not trigger a consistency rejection. Safe provenance access is not guaranteed.

7. **Separator gate:** A last output never performs separator processing. A nonlast output with boundary equal to flattened count also skips it; neither case advances the parent position.

8. **Branch precedence:** With destination byte +4 zero, even omit-separator mode uses the zero-format branch: no cursor decrement, zero scratch argument, and retained initial length.

9. **Length aliasing:** In the omit/nonzero-format branch, a dependency-written replacement length controls both subsequent scratch advancement and the checked separator call.

10. **Late failure:** A failed F-29265b828f07 result enters cleanup without the following reference call or cursor increments. Earlier mutations are not rolled back by any visible mechanism.

## Uncertainties

The following remain documented_only:

- Domain meaning and complete layouts of objects and records.
- All unsupplied dependency effects, including both apparent transfer operations.
- Ownership and recovery after partial execution.
- Required count, length, scratch, alignment, and arithmetic invariants.
- Whether external constraints prevent partition overflow or invalid backward indices.
- Separator-intermediate layout and scalar-output relationships.
- Concurrency, reentrancy, and field-stability guarantees.
- Meaningful failure results from unchecked calls.

## Evidence

The contract rests solely on the supplied static listing and opaque dependency map. Established statements concern visible access widths, direct writes, arithmetic, control flow, call arguments, result checks, and subsequent uses.

Apparent intent is explicitly distinguished from dependency guarantees. No runtime tests have been performed.
