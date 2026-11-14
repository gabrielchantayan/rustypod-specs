<!-- Trusted provenance: F-7cdcd05ba671; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7cdcd05ba671

## Purpose

F-7cdcd05ba671 dispatches on an object’s category selector and, for supported categories, submits a category-dependent presentation value through an indirect object operation.

The established behavior is a choice among constant resources, dependency calls, a one-byte selector write, and a final indirect call. Resource names suggest highlighted media-library presentation states: empty content, content without artwork, and content with artwork. Those meanings are **inferred intent**, not evidence that any particular screen, image, or text is displayed.

The function returns a 32-bit value:

- **1** after a supported category completes its dependency calls, presentation submission, and temporary cleanup.
- **0** for an unsupported selector, including categories 0 and 6.

A return of 1 indicates that the selected path completed normally. It does not establish successful rendering, content availability, or successful execution of any operation whose return value is ignored.

## Interface

### Inputs

The supplied representation exposes four arguments:

| Position | Width or representation | Established use |
|---|---|---|
| First | Object pointer in the represented 32-bit model | Supplies the selector, receives a byte write on supported paths, and is the receiver of the indirect presentation operation. It is also passed to F-753f887b468d on most media-category paths. |
| Second | 32 bits | No visible use. |
| Third | 32 bits | Passed twice to F-753f887b468d on the ordinary media-category paths. |
| Fourth | 32 bits | Passed to F-753f887b468d on those paths and copied into temporary storage at entry. |

The second argument is not validated or forwarded by the visible body. The third and fourth arguments have no established semantic type beyond their represented width and call-site use.

### Object layout

Two object-relative locations matter:

- **Byte offset 0xB0:** category-selector storage. Dispatch is represented as reading a 32-bit field and converting its value to an 8-bit character. Supported paths write one byte at this offset.
- **Byte offset 0:** a dispatch-table pointer. The indirect presentation operation is obtained from byte offset **0x11C** within the table referenced at submission time.

The selector write is not a whole-word assignment. It does not directly clear or replace the other three bytes of the represented 32-bit field.

For defined ordinary execution, the object must support the indicated reads and writes. Supported categories also require a valid dispatch-table pointer and callable entry at the specified table offset. Dependencies can impose further requirements that are not available here.

### Outputs and failure interface

There is no explicit error output, diagnostic emission, null check, range error, or dependency-failure conversion.

Invalid object, global, table, or function pointers are not handled locally. Abnormal dependency behavior may prevent the function from reaching its stated return. Exception propagation, fault handling, and unwinding behavior are undocumented.

## Behavior

### Dispatch and selector semantics

The selector is evaluated once for dispatch. The function does not re-dispatch if a dependency later changes the object.

Selectors **1–5 and 7–19** have supported paths. Selectors **0 and 6**, and all other values not matching a supported category, return 0 without a visible dependency call or object write.

Each supported path writes its category number as a byte at object offset 0xB0. Although that number matches the category initially selected, the write remains observable. Its timing relative to dependencies differs by category.

The representation’s conversion to an 8-bit character limits the dispatch value; it is not a comparison against the complete 32-bit field. Character signedness and target byte order are not independently established. In particular, the relationship between the numeric low byte used for dispatch and the addressed byte written at offset 0xB0 depends on byte order.

### Ordinary media categories

The following categories share a three-way resource-selection rule. Each row specifies a category-specific global pointer source, a 32-bit field offset within the referenced object, and an artwork mode.

The global pointer sources are separate references in the evidence. Equal field offsets do not prove that their bases are identical.

| Selector | Inferred category | Field offset | Artwork mode | Resource selected when field is zero |
|---|---|---:|---:|---|
| 1 | Music | 0x3F8 | 0 | EmptyMusicHilited |
| 2 | Videos | 0x424 | 1 | EmptyVideosHilited |
| 7 | Audiobooks | 0x414 | 0 | EmptyAudiobooksHilited |
| 8 | Music playlist | 0x3F8 | 0 | EmptyMusicPlaylistHilited |
| 9 | Artists | 0x3FC | 0 | EmptyArtistsHilited |
| 10 | Albums | 0x400 | 0 | EmptyAlbumsHilited |
| 11 | Compilations | 0x408 | 0 | EmptyCompilationsHilited |
| 12 | Songs | 0x3F8 | 0 | EmptySongsHilited |
| 13 | Genres | 0x40C | 0 | EmptyGenresHilited |
| 14 | Composers | 0x404 | 0 | EmptyComposersHilited |
| 15 | Video playlist | 0x410 | 1 | EmptyVideoPlaylistHilited |
| 16 | Movies | 0x420 | 1 | EmptyMoviesHilited |
| 17 | TV shows | 0x41C | 1 | EmptyTVShowsHilited |
| 18 | Music videos | 0x418 | 1 | EmptyMusicVideosHilited |
| 19 | Rentals | 0x428 | 1 | EmptyRentalsHilited |

For these categories, F-753f887b468d produces a 32-bit value used by subsequent artwork-related calls. Its visible argument sequence is:

- Category 1: literal 1, literal 0, third input, fourth input, third input again.
- Every other category in the table: object pointer, literal 0, third input, fourth input, third input again.

The category-1 first argument is therefore observably different; it must not be normalized to the object pointer.

Resource selection follows these rules:

- If the category’s 32-bit field equals zero, select the row’s empty resource. Neither F-6904540d62b2 nor F-5eac81954d4b is called.
- If that field is nonzero, call F-6904540d62b2 with the result from F-753f887b468d and the row’s artwork mode.
- If F-6904540d62b2 returns zero, select **NoArtMusicHilited** for mode 0 or **NoArtVideosHilited** for mode 1.
- If it returns nonzero, call F-5eac81954d4b with the same result and mode, then select **MediaContentHilited**.

The field tests and dependency-result tests are strictly zero-versus-nonzero. A negative 32-bit value takes the nonzero branch. The evidence does not establish that these fields are valid nonnegative counts.

The result of F-5eac81954d4b is not used to revise the resource choice.

### Ordering within ordinary media categories

For category 1:

1. The selector byte is written as 1.
2. F-753f887b468d is called.
3. The category-specific global pointer and its field are read.
4. Any required artwork calls occur.
5. The selected resource enters the common submission sequence.

For categories 2 and 7–18:

1. F-753f887b468d is called.
2. The category-specific global pointer is captured.
3. The selector byte is written.
4. The field is read through the captured pointer.
5. Any required artwork calls occur.
6. The resource enters the common submission sequence.

These differences matter if dependencies inspect the selector or mutate globals. In particular, the global pointer is not captured before F-753f887b468d.

Category 19 follows the latter ordering, with two additional calls after its selector-byte write and before its field read:

- F-794d66f15c1d.
- F-05af601191f2.

Their return values are ignored. The field at offset 0x428 is then read through the pointer captured before these calls. Consequently, mutations to the captured object can affect the decision, but replacing the global pointer does not by itself change which base is used for that field read. Whether either dependency invalidates that base is unknown.

### Category 3: podcasts

This category uses a separate availability path:

- A 32-bit value is read from a category-specific global source.
- The selector byte is written as 3.
- F-3881271d9a6c is called with the captured global value.
- A zero result selects **EmptyPodcastsHilited**.
- Any nonzero result selects **NoArtPodcastsHilited**.

No call to F-753f887b468d, F-6904540d62b2, or F-5eac81954d4b appears on this path. In particular, the supplied evidence establishes no artwork-present podcast branch.

### Category 4: fixed resource

The selector byte is written as 4, after which a fixed resource is submitted.

The resource’s contents and semantic name are unavailable. No visible content-field test or artwork dependency precedes its submission.

### Category 5: photos

The selector byte is written as 5 before these calls:

- F-6f828c57a429 receives 5 and 0 and returns a 32-bit value.
- F-317690e871b5 receives that result and 0 and returns a represented 64-bit value.

Only the **low 32 bits** of the 64-bit result determine the branch:

- Low half zero: select **EmptyPhotosHilited**, regardless of the high half.
- Low half nonzero: call F-753f887b468d with the low half followed by the high half; pass its result and mode 3 to F-5eac81954d4b; select **PhotosHilited**.

The split is a numeric 64-bit-to-two-32-bit decomposition, with the high half obtained by an unsigned right shift of 32 bits. There is no visible comparison of the full 64-bit value against zero.

No F-6904540d62b2 call appears on this path. Mode 3 must not be replaced with either ordinary-media artwork mode.

### Common submission and cleanup

Every supported category reaches the same observable completion pattern:

- F-9b9c1c7483f4 receives the address of temporary storage and the selected resource reference. Its 32-bit return value becomes the submitted value.
- The object’s current dispatch-table entry at byte offset 0x11C is called with the object pointer and that submitted value.
- F-8a5aadac202f receives the address of the temporary storage.
- The function returns 1.

The indirect target is obtained after resource preparation, not cached at entry. Changes to the object’s dispatch-table pointer by earlier dependencies can therefore affect the target used.

Categories 3 and 4 additionally show a third argument to F-9b9c1c7483f4: an address of adjacent temporary storage. Other paths show two arguments. The significance of this difference is undocumented.

The fourth input is copied into a 32-bit temporary at entry. The ordinary media-category paths visibly pass that temporary’s address to resource preparation and cleanup. The supplied representation uses different temporary locations for categories 3–5; their complete initialization and layout are not established.

## Side effects

### Direct effects

On supported paths, the function directly writes one byte at object offset 0xB0. It also writes temporary storage and invokes the indirect presentation operation.

Unsupported paths have no visible object write or dependency call.

The visible body does not directly write the category-specific global fields. This does not imply global preservation: any dependency or indirect operation may mutate reachable state.

### Ownership and lifetime

The object is caller-supplied. No explicit object allocation or object release is visible.

Selected resources are passed by reference; the function does not visibly allocate or copy their contents itself. Their storage duration and representation are unknown.

F-9b9c1c7483f4 appears to prepare a temporary presentation value, and F-8a5aadac202f appears to clean up associated temporary state. Those roles are inferred from the call pattern. It is not established whether either operation allocates, retains, transfers ownership, or releases memory.

The indirect presentation operation receives the prepared value before cleanup. Whether it copies or retains that value is unknown. No contract may assume that the submitted value remains usable after cleanup without additional dependency evidence.

There is no visible rollback. If a dependency fails abnormally after the selector write, the function provides no established restoration of the selector or prior presentation state.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-753f887b468d | Produces a 32-bit value used by artwork-related calls. Visible argument forms differ between category 1, ordinary categories, and nonempty photos. |
| F-6904540d62b2 | Receives that value and mode 0 or 1. Its zero/nonzero result controls no-art versus media-content selection. |
| F-5eac81954d4b | Called for ordinary media with a nonzero artwork-query result, or for photos with a nonzero low result half. Receives mode 0, 1, or 3. Its return is ignored. |
| F-3881271d9a6c | Receives a captured global value for podcasts. Its zero/nonzero result selects the podcast resource. |
| F-6f828c57a429 | Receives 5 and 0 on the photo path. |
| F-317690e871b5 | Receives the preceding result and 0; exposes the represented 64-bit photo result. |
| F-794d66f15c1d | First additional rental-path call, before the rental field read. |
| F-05af601191f2 | Second additional rental-path call, before the rental field read. |
| F-9b9c1c7483f4 | Receives temporary storage and the selected resource; produces the submitted value. |
| F-8a5aadac202f | Receives temporary storage after the indirect operation returns. |

The indirect presentation target has no supplied opaque function ID. It is identified only by its object-relative dispatch-table slot. Its return value is ignored.

None of these dependencies is established to preserve globals, object fields, resources, or memory ownership.

## Acceptance examples

These are conditional contract examples, not executed tests. They assume valid required storage and normal return from all invoked dependencies.

1. **Unsupported category:** A selector converting to 6 returns 0. No selector write, resource preparation, indirect submission, or cleanup occurs.

2. **Music state visible before acquisition:** With selector 1, F-753f887b468d is entered after the byte write of 1 and receives literal 1 as its first argument. If the subsequently read field at offset 0x3F8 is zero, EmptyMusicHilited is submitted without an artwork query.

3. **Video acquisition can affect availability:** With selector 2, if F-753f887b468d changes the category-specific global pointer, the later global-pointer read observes the resulting value. A nonzero field at offset 0x424 leads to an artwork query with mode 1.

4. **Negative field is nonempty for branching:** With selector 10 and a field value whose signed interpretation is −1 at offset 0x400, the artwork query still occurs. A zero query result selects NoArtMusicHilited.

5. **Artwork preparation does not determine selection:** With selector 16, a nonzero content field and nonzero artwork-query result cause F-5eac81954d4b to run with mode 1. On normal return, MediaContentHilited is selected regardless of its return value.

6. **Podcast result does not enable artwork:** With selector 3, any nonzero result from F-3881271d9a6c selects NoArtPodcastsHilited. No ordinary artwork-query call occurs.

7. **Photo high half alone is insufficient:** With selector 5, a represented 64-bit result with high half 1 and low half 0 selects EmptyPhotosHilited. F-753f887b468d and F-5eac81954d4b are skipped.

8. **Nonempty photos preserve both halves:** With selector 5 and a nonzero low half, F-753f887b468d receives the low and high halves in that order. F-5eac81954d4b then receives its result and mode 3 before PhotosHilited is submitted.

9. **Rentals retain the captured base:** With selector 19, the two rental-specific dependencies run in the stated order. If they replace only the global pointer while leaving the captured object valid, the availability field is still read from the previously captured object.

10. **Cleanup follows submission:** On any supported path that completes normally, the indirect operation receives the preparation result before F-8a5aadac202f runs. The final return is 1 even if the indirect operation returns zero.

## Uncertainties

The following are **documented_only**:

- Exact dependency prototypes and calling conventions, especially the differing visible argument counts.
- Full temporary-storage layout, initialization, and destruction semantics.
- Character signedness, target byte order, and independently verified machine-level selector-load width.
- Contents and meaning of the fixed category-4 resource.
- Whether separately referenced global sources alias.
- Whether the availability fields are counts, flags, or another representation.
- Whether dependencies mutate, replace, or invalidate captured pointers and objects.
- Whether resource preparation or indirect submission allocates, retains, copies, or releases memory.
- Reentrancy, concurrency, atomicity, and synchronization requirements.
- Behavior on invalid pointers, abnormal dependency termination, or failures not represented by an explicit local branch.

The highlighted-media interpretation is inferred from resource labels. It does not establish an external UI protocol.

## Evidence

The established contract derives solely from the supplied static function representation and opaque dependency mapping.

Static evidence supports the category coverage, byte writes, field offsets, zero/nonzero decisions, represented argument and result widths, dependency ordering, resource choices, indirect dispatch slot, cleanup ordering, and explicit return values.

Resource labels support inferred presentation intent, but dependency bodies, resource contents, global layouts, and executable observations were not supplied. No runtime tests have been performed.
