<!-- Trusted provenance: F-e55a4ca00dda; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e55a4ca00dda

## Purpose

F-e55a4ca00dda initializes a mutable face record from an input stream. It attempts initialization through a required service, then either completes service-assisted loading or falls back to direct loading after repositioning.

Established behavior includes format and index checks, allocation and attachment of a private object, conditional metadata construction, subordinate-object processing, and conditional character-map creation.

“Face,” “container,” “family,” “style,” and similar terms describe inferred roles. They do not establish dependency contracts, ownership, or format semantics.

This contract describes static evidence only. No runtime tests have been performed.

## Interface

### Inputs and result

The five inputs are:

1. A 32-bit input handle.
2. A 32-bit pointer to a mutable face record.
3. A signed 32-bit face index.
4. A 32-bit auxiliary value.
5. A second 32-bit auxiliary value.

The auxiliary values are forwarded to specified callbacks without local interpretation.

The result is a signed 32-bit value. Explicit rejection results include `2` and `6`. Several dependency failures are returned directly. Successful recognition with a negative index returns literal zero before private allocation.

**The final initialization path returns the then-current value of a mutable status word, not an unconditional zero.** That word contains zero when its address is supplied to the allocation dependency. Successful loading subsequently assigns zero to it. Later dependency calls occur before the final read, and the allocation dependency’s retention and mutation behavior is unknown. A final zero therefore requires that the word remain zero after successful loading.

There is no separate output object. Results are written into the face record and reachable objects.

### Representation

Pointers and ordinary integer fields occupy 32 bits in the supplied representation. Explicit 16-bit and byte accesses take precedence over inferred semantic types.

| Face offset | Width | Observed role |
|---|---:|---|
| `0x00` | 32 bits | Face-count-like value |
| `0x08`, `0x0C` | 32 bits each | Face and style flags |
| `0x10` | 32 bits | Glyph-count-like value |
| `0x14`, `0x18` | 32 bits each | Family and style pointers |
| `0x24`, `0x28` | 32 bits each | Map count and pointer array |
| `0x34`–`0x40` | Four 32-bit fields | Bounding box |
| `0x44` | 16 bits | Units value |
| `0x46`, `0x48`, `0x4A` | 16 bits each | Ascender, descender, height |
| `0x50`, `0x52` | 16 bits each | Underline-related values |
| `0x5C` | 32 bits | Selected-map pointer |
| `0x60`, `0x64` | 32 bits each | Service and allocation contexts |
| `0x94` | 32 bits | Format tag |
| `0x1F8` | 32 bits | Table-selection callback |
| `0x28C` | 32 bits | Private-object attachment |

Private-object offsets below are relative to the pointer returned by allocation and retained locally, not necessarily to the face’s current attachment.

### Preconditions

Reached fields, nested pointers, callbacks, arrays, and strings must support the specified accesses. Locally traversed strings must remain readable through their terminating zero bytes.

No general pointer, record-size, array-bound, or string-length validation is visible. Normal bounded subordinate traversal requires a nonnegative count and sufficient pointer slots. Negative counts do not select a rejection path; they fall outside the supported traversal preconditions.

Dependencies may mutate accessible state. Preservation of globals, face fields, pointed-to objects, allocations, or escaped local storage is not assumed.

## Behavior

### Discovery, positioning, and recognition

F-36f29e709002 first receives the word at offset `0x04` of the object referenced by face `0x60`, together with an unavailable service identifier. A zero result returns `2` immediately.

Otherwise, calls proceed in this order:

1. F-c79a29304dfd receives the current face `0x60` value and `postscript-cmaps`.
2. F-36f29e709002 receives the nested context word reached through a fresh face `0x60` read, together with `pshinter`.
3. F-e40a3ceddab2 receives the input handle and position zero.

The two optional service results are retained. Their absence is not immediately rejected. A nonzero positioning result is returned.

The required service callback F-unresolved-02, at service offset `0x04`, then receives all five original arguments.

On a zero probe result:

- Face `0x94` is compared with an unavailable format constant.
- A mismatch returns `2`. One diagnostic call precedes that return only when the signed diagnostic level at offset `0xA8` through a global pointer is at least two.
- With a matching format, a negative face index returns literal zero.
- A positive face index causes a diagnostic call and returns `6`.
- Index zero records recognized-container state and continues.

These returns precede allocation but do not undo earlier callback effects.

For recognized index zero, F-unresolved-05, read from face `0x1F8`, receives the face, a first unavailable table identifier, the input handle, and zero. Its result chooses the metadata mode:

- Nonzero selects standalone-style metadata and calls F-unresolved-04 at required-service offset `0x20`, with face and input handle.
- Zero selects container metadata and calls F-unresolved-03 at required-service offset `0x08`, with all five original arguments.

A nonzero selected-service result is returned. Otherwise, the current face `0x1F8` callback is read again and called with a second unavailable table identifier. Its nonzero result is returned. The first table-selection failure is therefore a mode selector, not itself a returned failure.

On a nonzero initial probe result, that result is discarded. F-e40a3ceddab2 requests position zero again. Failure is returned; success continues in standalone-style mode without recognized-container state.

### Allocation, escaped status, attachment, and loading

At the allocation point, the status word contains zero because the preceding positioning or table-selection result passed its zero check.

The current face `0x64` value is captured as the allocation context. F-3ce007b692cc receives that context, an unavailable allocation size, and the address of the zero-valued status word.

The function checks the status word after allocation. If nonzero, its value is returned. Otherwise:

1. The returned pointer is written to face `0x28C`.
2. F-dda89080462f receives the input handle, signed face index, and returned pointer.
3. The loader result is assigned to the same status word.
4. A nonzero value returns immediately, without local detachment or release.

A successful loader therefore writes zero to the escaped status word. This does not establish its value for the remainder of execution: the allocator may have retained its address, and later calls may modify it through behavior not supplied here.

After successful loading, the retained private pointer is used to:

1. Store the hinter-service result at private `0x7D4`.
2. Store the character-map-service result at private `0x7D8`.
3. Copy private `0x0C` to face `0x10`.

If private 32-bit `0x580` equals `0x0000FFFF` and the retained character-map service is zero, four diagnostics are issued and literal `2` is returned.

### Metadata modes and arithmetic

Container-metadata mode performs only this local metadata adjustment: if private 16-bit `0x540` is zero, copy face 16-bit `0x44` into it. It then advances to subordinate processing.

Standalone-style mode replaces the following fields:

- Face `0x00` receives private `0x08`.
- Face `0x10` receives private `0x498` when private `0x580` equals `0x0000FFFF`, otherwise private `0x598`.
- Face `0x34` and `0x38` receive signed 32-bit private `0x550` and `0x554`, respectively, arithmetically shifted right by 16.
- Face `0x3C` and `0x40` receive private `0x558` and `0x55C`, respectively, after unsigned 32-bit addition of `0xFFFF` and logical right shift by 16.

The upper-bound additions wrap modulo \(2^{32}\); they are not unlimited-precision ceiling operations.

Zero private 16-bit units at `0x540` become `1000`. The unsigned 16-bit units value is copied to face `0x44`. Face `0x40` and `0x38` are captured and narrowed into face `0x46` and `0x48`.

F-d7be9cfb4000 receives unsigned units multiplied by twelve and the integer ten. The multiplication fits in 32 bits for every 16-bit units value. Its result is narrowed to signed 16 bits and stored at face `0x4A`.

The two captured vertical values are independently interpreted as signed 16-bit values and subtracted in signed 32-bit arithmetic. If this difference exceeds the narrowed helper result, its low 16 bits replace face `0x4A`. Consequently, a positive difference exceeding `32767` can produce a negative stored signed-16-bit height.

Private `0x520` and `0x524` are logically shifted right by 16 and narrowed into face 16-bit `0x50` and `0x52`.

### Names and mutation-sensitive reads

Name construction occurs only in standalone-style mode.

F-e4a129bd349d receives private address `0x14` and the face index. Its result is immediately stored at face `0x14`.

If that result is zero, F-b75751130389 receives private address `0x44`, the identifier from private `0x5A8`, and the retained character-map service. A nonzero result replaces face `0x14`; a zero result causes no replacement store. `Regular` is then requested through F-0486e77c51b0 using the captured allocation context.

For a nonzero initial family result:

1. F-b75751130389 retrieves a full-name-like pointer using private identifier `0x50C`.
2. After that call, face `0x14` is read as the comparison pointer. It need not still contain the initial result.
3. If private identifier `0x510` is nonzero, another retrieval requests an alternative pointer.
4. A nonzero alternative replaces the captured comparison pointer. A zero alternative leaves the captured pointer unchanged, even if that call changed face `0x14`.

When the full-name and effective comparison pointers are both nonzero, local byte traversal occurs:

- The current full-name byte is read first; zero ends traversal without requiring a comparison-byte read at that position.
- Equal bytes advance both pointers.
- Otherwise, a space or hyphen in the full name advances only the full-name pointer.
- Otherwise, a space or hyphen in the comparison string advances only that pointer and retries the same full-name position.
- Other mismatches end traversal. At that mismatch, a zero comparison byte and a remaining nonzero full-name byte cause the full-name suffix to be supplied to F-0486e77c51b0.

After an entered traversal finishes, a nonzero alternative pointer is passed to F-bf7242c98d06, then the full-name pointer is passed to it. Both calls also receive the captured allocation context. A nonzero suffix-acquisition result is selected; otherwise `Regular` is requested.

If either pointer needed to enter traversal is zero, `Regular` is requested without visible release-like calls for those retrieved pointers on that branch.

The selected style pointer, including zero, is stored at face `0x18`. These acquisition results do not directly select an error return.

### Face and style flags

Standalone-style mode replaces face `0x08` with an unavailable base flag word. Recognized-container state changes this to the base OR its signed arithmetic right shift by eight. A nonzero private byte at `0x518` adds bit `0x04`.

The style accumulator starts at exactly one if private 32-bit `0x51C` is nonzero, otherwise zero.

F-b75751130389 retrieves a weight-like pointer using private identifier `0x514`. For a nonzero result, F-unresolved-01 compares it with an unavailable text address. A zero result sets bit `0x02` and skips the next comparison. Otherwise, the same retained pointer is passed with `Black`; zero sets bit `0x02`.

F-bf7242c98d06 then receives the captured allocation context and retrieved pointer, including a zero pointer.

If bit `0x02` remains clear and face `0x18` is nonzero, F-fca51b715561 receives a fresh face `0x18` read, the unavailable text address, and `4`. Zero sets bit `0x02`. Otherwise, a second call receives another fresh face `0x18` read, `Black`, and `5`; zero sets the bit. The second pointer may differ from the first, including becoming zero.

The completed accumulator replaces face `0x0C`. Equality and prefix-comparison interpretations remain inferred because the comparison implementations are unavailable.

### Subordinate objects and sentinel timing

The signed 32-bit count at private `0x734` is captured once. Traversal descends from that count through one, reading an object pointer at private offset `0x734 + 4 × index`.

Every loop-condition evaluation reads the global sentinel before checking whether the index is zero. This includes the terminating evaluation and the initial evaluation for count zero. Dependency calls during one iteration may affect the next sentinel capture.

For an object with zero 16-bit `0x40`:

- Parent `0x534`, `0x538`, and `0x53C` are captured before parent `0x530` is copied to object `0x30`.
- The captured words are written to object `0x34`, `0x38`, and `0x3C`, in order.
- Parent `0x548` is captured before parent `0x544` is copied to object `0x44`.
- The captured word is written to object `0x48`.

Otherwise, F-73d099a86adb receives the parent’s current 16-bit units. A four-word temporary has its result in the first and fourth words and zero in both middle words. F-03aeabcbc56c receives the temporary’s address and object address `0x30`. F-baed69327072 then receives object address `0x44` and that same temporary’s address. Their results are not checked; preservation of the temporary across the first call is not guaranteed.

The terminating sentinel capture is retained for synthetic-map eligibility. A separate global-sentinel read is used immediately afterward: equality with private `0x580` causes `0x200` to be ORed into current face flags. No dependency call intervenes between these two sentinel observations; asynchronous changes remain uncharacterized.

### Character maps and completion

Existing maps are scanned using an ascending unsigned 32-bit index. Each condition reads face `0x24` as the bound; entries are reached through face `0x28`. An entry qualifies when its 16-bit `0x08` is zero, or when `0x08` is three and 16-bit `0x0A` is one.

A qualifying entry skips synthetic creation and reaches the optional-map step.

Without a qualifying entry, synthetic creation is attempted:

- Always in container-metadata mode.
- In standalone-style mode only if a fresh private `0x580` read equals the terminating subordinate-loop sentinel capture.

F-e504f72ffcc9 receives an unavailable class descriptor, zero, a three-word temporary’s address, and zero. The temporary contains the face pointer, an unavailable encoding value, and packed selector `0x00010003`.

The old map count is saved before this call. Its return value is ignored. Afterward, if face `0x5C` is zero and face `0x24` differs from the saved count, face `0x5C` receives the array entry indexed by the old count. This does not establish that the count increased or that exactly one map was appended.

The optional-map step is reached only through an existing qualifying map or the synthetic-creation branch. If private `0x7C` is nonzero, another four-argument F-e504f72ffcc9 call uses a second class descriptor and an analogous record:

| Private `0x78` | Packed selector | Encoding |
|---|---:|---|
| Zero | `0x00000007` | Unavailable constant A |
| One | `0x00010007` | Unavailable constant B |
| Other | `0x00020007` | Unavailable constant C |

That return value is also ignored.

Completion reads and returns the escaped signed 32-bit status word. Ignoring map-helper return values does not imply that these calls, or earlier dependencies, cannot indirectly affect that word.

## Side effects

Observable effects include positioning requests, diagnostics, private attachment before loading, metadata replacement, release-like calls, subordinate writes, flag updates, map-creation requests, and conditional selected-map assignment.

Errors do not imply rollback. Loader failure occurs after attachment, with no local detachment or release. Whether the loader preserves that attachment or allocation is unknown. Missing-character-map-service rejection follows successful loading and the service-pointer and glyph-count stores.

F-bf7242c98d06 is release-like by use; its ownership and null-handling contracts are unavailable. No local final release of stored names, the private object, or created maps is visible.

The allocation call exposes a local status address. Its permitted retention, later mutation, and use after return are not established. The contract does not authorize post-return access or assume that the dependency refrains from retaining it.

## Dependencies

| Opaque ID | Observed use |
|---|---|
| F-36f29e709002 | Service lookup |
| F-c79a29304dfd | Character-map service lookup |
| F-e40a3ceddab2 | Positioning request |
| F-4457a9a44a68 | Diagnostics |
| F-3ce007b692cc | Allocation; receives address of zero-valued status word |
| F-dda89080462f | Private-object loading |
| F-d7be9cfb4000 | Height-related computation |
| F-e4a129bd349d | Initial family candidate |
| F-b75751130389 | Identifier-based string retrieval |
| F-0486e77c51b0 | String acquisition |
| F-bf7242c98d06 | Release-like operation |
| F-fca51b715561 | Three-argument comparison-like operation |
| F-73d099a86adb | Transform-input computation |
| F-03aeabcbc56c | Transform-related operation |
| F-baed69327072 | Offset-related operation |
| F-e504f72ffcc9 | Map-creation-like operation |

Document-local IDs F-unresolved-01 through F-unresolved-05 identify, respectively, the unresolved two-argument comparison helper, required-service callbacks at offsets `0x04`, `0x08`, and `0x20`, and the face table-selection callback. They do not assert resolved identities.

## Acceptance examples

These are conditional static expectations, not executed tests.

1. **Missing required service:** A zero first lookup returns `2` before positioning.
2. **Index handling:** Successful recognition with a matching format and index `−1` returns literal zero; index `1` produces a diagnostic and returns `6`. Neither reaches allocation.
3. **Probe fallback:** A failed probe followed by successful repositioning can reach direct loading. The probe failure is not propagated.
4. **Allocation entry:** When allocation is reached, its status-address argument points to a signed 32-bit word containing zero. A nonzero post-call status is returned before attachment.
5. **Loader failure:** A nonzero loader result is assigned to the status word and returned. Attachment preceded the call; its continued presence requires loader preservation.
6. **Conditional zero completion:** If loading succeeds, completion is reached, and the escaped status remains unchanged afterward, the final result is zero.
7. **Escaped-status mutation:** If allocation retains the status address and a later dependency changes that word to `17` after successful loading, with no subsequent change before completion, the final result is `17`.
8. **Arithmetic wrap:** An upper-bound bit pattern `0xFFFF8000` becomes zero after unsigned addition of `0xFFFF` and logical shift by 16.
9. **Mutation-sensitive names:** A family-field change during full-name retrieval is observed by the subsequent face `0x14` read. A nonzero alternative result then overrides that captured pointer.
10. **Style suffix:** Stable comparison and full-name strings `Example` and `Example-Bold` cause acquisition of suffix `Bold`. A zero acquisition result leads to a `Regular` request.
11. **Changing sentinel:** A subordinate helper’s sentinel change is captured by the next condition evaluation. Synthetic eligibility uses the terminating capture.
12. **Ignored map result:** A nonzero F-e504f72ffcc9 return is not assigned to the status word. Final zero still requires that the escaped word remain zero.

## Uncertainties

The following remain documented_only:

- Unavailable constants, allocation size, table identifiers, class descriptors, and encodings.
- Comparison-helper semantics and memory-read extents.
- Dependency mutation, allocation, ownership, release, and null-handling contracts.
- Whether allocation retains the status address, and whether later calls modify that escaped word.
- The final status value without a preservation condition after successful loading.
- The relationship between explicit `0x0000FFFF` checks and the global sentinel.
- Behavior outside valid pointer, string, count, and addressing preconditions.
- Asynchronous mutation and effects not represented by the supplied accesses.

## Evidence

Established evidence consists of explicit access widths, offsets, arithmetic, branch conditions, call arguments and order, stores, and return paths in the supplied representation.

In particular, allocation receives the status word by address while it contains zero; the loader result is subsequently assigned to that word; later calls precede its final read. These facts establish conditional zero completion, not unconditional zero completion.

Semantic terminology and dependency purposes are inferred where implementation contracts are unavailable. The supplied type-propagation warning limits confidence in semantic types beyond explicit accesses.

No dependency implementation inspection or runtime execution supports stronger claims.
