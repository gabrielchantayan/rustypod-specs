<!-- Trusted provenance: F-a1e33de09da2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a1e33de09da2

## Purpose

F-a1e33de09da2 performs preliminary object initialization and service discovery. After successful underlying initialization, index zero additionally publishes names, flags, bounds, and metrics, then conditionally requests character-map construction.

Three index cases are established:

- A negative index returns after underlying initialization, without the additional publication work.
- Index zero performs that work.
- A positive index produces a diagnostic and returns status 6, but only after successful underlying initialization.

The font-face interpretation is inferred from service names, strings, and field relationships. It does not establish dependency contracts or formal meanings for every published field.

## Interface

### Arguments and access requirements

The supplied representation exposes three arguments:

1. An opaque 32-bit value with no visible direct use.
2. A pointer to a mutable object.
3. A signed 32-bit index.

The function accesses the object through 32-bit, 16-bit, and byte operations. Pointer-bearing words have the supplied 32-bit representation.

There are no visible checks for a null object, sufficient object size, accessible service tables, valid string storage, or string termination. Callers must provide storage supporting all accesses reached on the selected path. Invalid accesses have no established status-return contract.

All offsets below are byte offsets from the object base.

| Offset | Observed role |
|---|---|
| `0x000` | Preliminary field written with 1 |
| `0x004` | Published index-like field |
| `0x008` | Published capability word |
| `0x00C` | Published style word |
| `0x010` | Published count copied from `0x19C` |
| `0x014`, `0x018` | Published family and style pointers |
| `0x01C`, `0x020` | Fields cleared during publication |
| `0x034`–`0x040` | Four published 32-bit bounds |
| `0x044`–`0x052` | Packed 16-bit units and metrics |
| `0x060` | Context pointer for service discovery |
| `0x08C` | Full-name pointer |
| `0x090` | Preferred family pointer |
| `0x094` | Weight-name pointer |
| `0x098` | 32-bit style indicator |
| `0x09C` | Byte selecting an alternative capability value |
| `0x09E`, `0x0A0` | Source 16-bit metrics |
| `0x168` | Fallback family pointer |
| `0x16C` | Character-map selector byte |
| `0x19C` | Source count |
| `0x1C8`–`0x1D4` | Four source 32-bit bounds |
| `0x1E0`, `0x1E4`, `0x210` | Stored service results |
| `0x20C` | 32-bit indicator enabling capability bit `0x100` |

The highest direct object access is the four-byte field at `0x210`. This is not evidence of the complete object size or layout.

### Return value

The result is a 32-bit status:

- A nonzero status from F-1e4d0fba272a is returned unchanged.
- Successful underlying initialization with a negative index returns zero.
- Successful underlying initialization with a positive index returns 6.
- Completion of the index-zero path returns zero.

The optional metric query and character-map request results do not become the returned status. There is no transactional-update guarantee.

## Behavior

### Preliminary setup and index dispatch

Before considering the index, the function performs these operations in order:

1. Writes 1 to `0x000`.
2. Reads the context pointer at `0x060` and calls F-c79a29304dfd with it and `postscript-cmaps`.
3. Stores that result at `0x1E0` and retains the result locally.
4. Reads the current context pointer and its 32-bit field at offset `+4`; calls F-36f29e709002 with that field and `psaux`.
5. Stores that result at `0x1E4` and retains it locally.
6. Reads the context pointer and its field at `+4` again; calls F-36f29e709002 with that field and `pshinter`.
7. Stores the result at `0x210`.
8. Calls F-1e4d0fba272a with the object pointer.

The context is not assumed stable across these calls. In particular, later discovery uses the context then present in the object.

If underlying initialization returns nonzero, the function returns that status. Otherwise:

- A negative index returns zero immediately.
- A positive index calls F-4457a9a44a68 with an invalid-face-index diagnostic, then returns 6.
- Index zero continues with publication.

The locally retained first two service results, rather than later reads of `0x1E0` and `0x1E4`, govern character-map work.

### Capability publication

The read and write order is significant:

1. Copies the 32-bit value at `0x19C` to `0x010`.
2. Loads a first unavailable 32-bit word from global storage.
3. Writes zero to `0x004`.
4. Unconditionally writes the loaded global word to `0x008`.
5. Reads the byte at `0x09C`.
6. If that byte is nonzero, loads an alternative 32-bit word from global storage and writes it to `0x008`.
7. Reads `0x20C`; if nonzero, ORs `0x100` into the current word at `0x008`.

Thus a nonzero selector causes two capability writes, not a single write of a preselected value. The two global words have unavailable values. Their immutability, distinctness, and exact capability meanings are not established.

### Family and style pointers

The function copies `0x090` to `0x014`, then publishes the static string `Regular` at `0x018`.

If the published preferred family pointer is null, a nonnull pointer at `0x168` replaces it. No suffix extraction occurs on this fallback path.

With a nonnull preferred family and a nonnull full-name pointer at `0x08C`, the function scans the two byte strings:

- Each iteration first reads the current full-name byte. A terminating zero ends the scan without assigning a suffix.
- Otherwise, it reads the current family byte.
- Equal bytes advance both cursors.
- For unequal bytes, a space or hyphen in the full name advances only the full-name cursor.
- Otherwise, a space or hyphen in the family name advances only the family cursor.
- Otherwise, a terminating family byte causes the current full-name cursor to be published at `0x018`.
- Any other mismatch ends the scan, leaving `Regular`.

Separators are skipped only on mismatch. This is not an independent normalization pass.

A selected suffix aliases an interior position in the full-name storage. The function does not allocate, copy, or trim it. The published family pointer likewise aliases supplied storage.

### Style flags and weight comparisons

The function writes zero to `0x00C`, then writes 1 there if the 32-bit indicator at `0x098` is nonzero.

It next checks whether the weight pointer at `0x094` is nonnull. If so, it invokes the unidentified comparison dependency, designated here as F-unknown-comparison, with that pointer and an unavailable comparison string.

- A zero result skips the second comparison and causes bit `0x2` to be ORed into the then-current word at `0x00C`.
- A nonzero result causes a new read of the pointer at `0x094`. The function compares this newly read pointer against `Black`.
- There is no repeated nonnull check before the second call.
- A zero second result causes the same OR operation; otherwise no additional style write occurs.

The second comparison can therefore receive a different pointer, including null, if the first dependency changes `0x094`. The behavior of the comparison dependency on such an argument is unresolved. Its case sensitivity and full string contract are also unresolved.

The OR operation uses the style word current after the relevant dependency returns; earlier style initialization is not a guarantee that dependencies leave it unchanged.

### Bounds and retained metric inputs

The function clears `0x01C` and `0x020`, in that order.

It then publishes:

- At `0x034`, the signed 32-bit source at `0x1C8`, arithmetically shifted right by 16.
- At `0x038`, the signed 32-bit source at `0x1CC`, arithmetically shifted right by 16.
- At `0x03C`, the logical right shift by 16 of the source at `0x1D0` plus 65,535, with addition modulo \(2^{32}\).
- At `0x040`, the same modular-addition and logical-shift transformation of `0x1D4`.

The lower-bound transformations preserve sign. The upper-bound transformations produce 32-bit results between 0 and 65,535. They are not equivalent to unbounded signed ceiling arithmetic for all inputs.

Before the metric helper call, the function retains:

- The horizontal upper-bound source read from `0x1D0`, for later use in the metric at `0x04C`.
- A signed 16-bit value formed from the upper half of the lower vertical source at `0x1CC`.
- A signed 16-bit value formed by narrowing the transformed upper vertical source.

If the 16-bit field at `0x044` is zero, the function replaces it with 1000. Otherwise it preserves it. It then writes the retained upper and lower vertical 16-bit values to `0x046` and `0x048`, respectively.

### Height calculation and optional metric query

F-d7be9cfb4000 receives:

- The unsigned 16-bit value at `0x044`, multiplied by 12.
- The integer 10.

The first argument is at most 786,420, so this multiplication does not overflow 32 bits.

The helper result is narrowed to a retained signed 16-bit value. The function computes the promoted signed difference between the two **retained pre-call vertical values**, upper minus lower. It writes the narrowed helper result to `0x04A`, then replaces it with the low 16 bits of that difference if the retained signed helper result is smaller.

Neither subtraction operand is reloaded from `0x046` or `0x048` after the helper call. Mutating either field inside F-d7be9cfb4000 does not change the subtraction operands. The comparison also uses the retained narrowed helper result, not a reread of `0x04A`.

The difference ranges from −65,535 to 65,535 and fits signed 32-bit arithmetic. Narrowing a selected difference can nevertheless produce a stored signed 16-bit interpretation unlike the mathematical difference. The operation is not an unrestricted mathematical maximum.

Next, `0x04C` receives the low 16 bits of the transformed horizontal upper bound, calculated from the source retained before the helper call. Helper mutations of the source or published horizontal bound do not alter this initialization.

F-1fad469a5161 is then called with the object and temporary output-storage address:

- On status zero, the function copies the first two bytes of that storage to `0x04C`.
- On nonzero status, it performs no success-only overwrite.

The success read accesses the output storage itself. It does not dereference a pointer encoded in that storage.

After this dependency returns, the function performs these 16-bit copies in order:

1. Current `0x04A` to `0x04E`.
2. Current `0x09E` to `0x050`.
3. Current `0x0A0` to `0x052`.

These are post-call reads. Dependency mutations can affect them. On query failure, dependency mutations of `0x04C` also remain possible; the function does not restore its earlier initialization.

### Character-map requests

This stage requires both locally retained service results to be nonzero.

The function reads a table pointer at offset `+0x14` of the retained `psaux` result and retains that table pointer. It prepares temporary storage comprising three consecutive 32-bit words:

| Descriptor offset | Value |
|---|---|
| `+0` | Object pointer |
| `+4` | Unavailable word loaded from global storage |
| `+8` | `0x00010003` |

The visible preparation sets the third word, loads and assigns the global second word, then sets the first word.

The first F-e504f72ffcc9 request receives the table entry at byte offset `+12`, zero, the descriptor address, and zero. There is no visible nonzero check on that entry. The returned result is ignored.

Only after this call returns does the function read the current selector byte at `0x16C`. Recognized selectors prepare the following updates:

| Selector | Third descriptor word | Selected table entry |
|---|---:|---:|
| 1 | `0x00020007` | `+8` |
| 2 | `0x00000007` | `+0` |
| 3 | `0x00030007` | `+12` |
| 4 | `0x00010007` | `+4` |

Each recognized branch also loads its associated second descriptor word from global storage **after the first request returns**. These are separate global locations; their values are unavailable and are not proven immutable or numerically distinct.

For selector 1, the second word is assigned before the third. For selectors 2–4, the third word is assigned before the second. The selected table entry is then read through the retained table pointer. Changes to table contents by the first request can affect this read; the table pointer is not reacquired from the service object.

An unrecognized selector returns zero without a second request. For a recognized selector, a zero selected entry skips that request.

Otherwise, F-e504f72ffcc9 is called again. The supplied representation establishes only the selected entry and zero as arguments at this second site. It does not establish whether or how the updated descriptor is supplied. The second result is ignored.

## Side effects

Direct effects include ordered object writes, global-word reads, string-byte reads, service-table reads, and dependency calls.

No explicit allocation, deallocation, string duplication, reference-count adjustment, or service-result release appears in this function. Dependencies may perform such operations.

Ownership is not established:

- Published names alias existing or static storage.
- Stored service results receive no visible retain or release operation.
- The first construction request receives temporary descriptor storage; copying versus retention is unknown.
- Passing output storage to the optional metric dependency does not establish ownership transfer.

No visible rollback occurs. Preliminary writes and dependency effects can remain after initialization failure or positive-index rejection. Dependencies are not assumed to preserve the object, global words, table contents, or other external state.

## Dependencies

| Opaque ID | Established use |
|---|---|
| F-c79a29304dfd | Lookup using the context and `postscript-cmaps` |
| F-36f29e709002 | Lookups using a context-associated word and `psaux` or `pshinter` |
| F-1e4d0fba272a | Underlying initialization; its status controls continuation |
| F-4457a9a44a68 | Invalid-index diagnostic |
| F-unknown-comparison | Weight-string comparisons; documentary label because no mapped ID was supplied |
| F-d7be9cfb4000 | Metric calculation from two integer arguments |
| F-1fad469a5161 | Optional metric query using temporary output storage |
| F-e504f72ffcc9 | Character-map construction requests |

These descriptions specify observed use, not complete callee contracts.

## Acceptance examples

All examples are conditional static expectations, not executed tests.

1. **Initialization failure takes precedence.** If F-1e4d0fba272a returns 9, the result is 9 for any index. Preliminary service operations have occurred; additional publication and the positive-index diagnostic do not occur.

2. **Index dispatch after success.** With underlying status zero, index −1 returns zero without publication; index 2 calls the diagnostic dependency and returns 6.

3. **Capability writes are not collapsed.** If the first global capability word supplies value \(A\), `0x09C` is nonzero, and the alternative supplies \(B\), the function writes \(A\) and then \(B\) to `0x008`, before any conditional `0x100` OR.

4. **Family suffix extraction.** Stable preferred family `Example` and full name `Example-Bold` publish an interior pointer to `Bold`. Equal strings `Example` leave `Regular`. A fallback family does not trigger suffix extraction.

5. **Weight pointer changes between calls.** If the first comparison returns nonzero after replacing `0x094`, the second receives the replacement pointer. If it replaces the field with null, the second still receives null; the resulting callee behavior is unresolved.

6. **Height operands survive helper mutation.** Suppose the retained upper and lower vertical values are 800 and −200. If the helper narrows to signed 900 while changing `0x046` and `0x048`, the comparison still uses difference 1000 and writes 1000 to `0x04A`.

7. **Height narrowing is observable.** Retained values 32,767 and −32,768 yield difference 65,535. A narrowed helper result of 1000 is smaller, so the function stores `0xFFFF` at `0x04A`, whose signed 16-bit interpretation is −1.

8. **Bound arithmetic differs by position.** Lower source `0xFFFF8000` publishes −1. Upper source `0xFFFF0000` publishes 65,535. Upper source `0xFFFFFFFF` publishes zero because the addition wraps.

9. **Optional query failure is not propagated.** A nonzero F-1fad469a5161 result causes no success-only copy to `0x04C` and does not become the final status. Independent dependency mutations remain possible.

10. **Construction observes post-call state.** If the first request changes `0x16C` to 2 and changes that selector’s global descriptor word, the function uses selector 2 and loads the changed global value afterward. A zero selected table entry then skips the second request.

11. **Unsupported selector still follows the first request.** With both retained services nonzero and a post-request selector of 0, the first request occurs and the function returns zero without a second request.

## Uncertainties

The following remain **documented_only**:

- Values and mutability guarantees of capability and descriptor global words.
- Bytes of the first weight-comparison string.
- Complete object, context, service-table, and output-storage layouts.
- Full dependency contracts, including global mutations, ownership, and failure behavior.
- Comparison semantics, including treatment of a null second-call input.
- Internal arithmetic of F-d7be9cfb4000.
- Whether successful optional queries initialize every byte subsequently read.
- The complete argument contract of the second construction request.
- Whether descriptor words represent paired 16-bit identifiers.
- Formal meanings of flags and metrics.
- Behavior for invalid pointers, unterminated strings, inaccessible storage, or violated dependency preconditions.

## Evidence

Established evidence consists of the supplied static function representation and dependency-ID mapping. It supports the access widths, branch conditions, retained values, explicit reloads, global-read timing, write order, and status handling described above.

Font-face initialization, family/style naming, fixed-point bounds, and character-map construction are inferred intent where not directly established by visible strings or operations.

No runtime tests have been performed. Acceptance examples are derived from static evidence; unknown dependency behavior has not been verified.
