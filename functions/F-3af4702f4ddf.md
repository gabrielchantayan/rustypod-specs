<!-- Trusted provenance: F-3af4702f4ddf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3af4702f4ddf

## Purpose

F-3af4702f4ddf coordinates an opaque working handle, encoded status bytes, a 256-byte automatic workspace, shared-state flags, and dependency calls. It accepts five 32-bit arguments and exposes an unsigned 32-bit result.

**Established static evidence:** the supplied representation defines local initialization, a dispatcher, conditional shared-memory operations, dependency calls, status transformations, and a return region. It also establishes specific boundaries between external values captured before calls and values read afterward. Those boundaries matter because dependency effects on external data are unknown.

**Inferred intent:** the handle-oriented calls and flag tests suggest a staged operation. No application domain, success convention, or resource-management protocol is established.

**Contract boundary:** external dispatch constants are unavailable. Consequently, the complete entry-to-return path, region reachability, repetition, termination, and input-to-result mapping remain **documented_only**. The regional behaviors below are conditional obligations, not an asserted execution sequence.

## Interface

### Arguments and result

| Position | Width and interpretation | Established use |
|---|---|---|
| 1 | Opaque 32-bit value | Retained at entry and later copied into working arguments. |
| 2 | Signed 32-bit integer | Retained at entry; later multiplied by an external arithmetic value. |
| 3 | Opaque 32-bit value | Retained at entry and later copied into a working argument. |
| 4 | Opaque 32-bit value | Retained at entry and later copied into several working values. |
| 5 | Opaque 32-bit value | Copied into working storage in a later region. |

The opaque inputs are not established to be addresses, handles, lengths, or flags. No explicit argument validation is visible. Dependency-specific validity requirements remain unknown.

The result is unsigned and 32 bits wide. A result accumulator starts at zero and may subsequently be replaced or transformed using external data, status arithmetic, or the result of an unmapped dependency. The return region returns its current value. Zero initialization does not establish a zero result for any particular input.

### Storage and layout

The function provides a 256-byte automatic workspace. Its contents are not initialized in full. Two workspace-related pointer slots initially contain null; a setup region assigns the workspace address to both.

An opaque working handle initially contains zero. Two shared-state words are accessed through a base captured from external data at entry:

| Offset from captured base | Access width | Operations |
|---|---:|---|
| 128 bytes | 32 bits | Mask tests and read–OR–write updates |
| 132 bytes | 32 bits | Bit-zero test and read–OR–write update |

For regions accessing these words, the corresponding addresses must support the indicated reads and writes. Alignment, accessibility, synchronization, and any dependency-specific memory requirements are **documented_only**.

A separate region writes one byte at offset 248 from a working pointer. It addresses the automatic workspace only if that pointer has the workspace value when the region executes.

## Behavior

### Entry and dispatch

Entry captures the shared-state base, initializes numerous working values—including the handle and result accumulator—to zero, initializes both workspace-related pointers to null, retains the first four inputs, and obtains an initial dispatch state from external data.

The fifth input is installed later. Entry does not initialize the workspace contents or the shared-state words.

Each dispatcher iteration performs local snapshots and several external reads before selecting a region. In particular, it:

1. Snapshots two local working values.
2. Reads an external default-region transition value.
3. Reads an external transition value used by the `0x01000000` flag-update region.
4. Reads the external argument used by the offset-132 flag notification.
5. Reads the external argument used by the `0x20000000` flag notification.
6. Reads the external argument used by the `0x00400000` flag notification.
7. Narrows the current status to signed 16 bits.
8. Reads the selector baseline and selects a region using the current dispatch state minus that baseline.

These reads occur even when the selected region does not use all the captured values. The selector baseline is read on every iteration; it is not established to be constant throughout the invocation. The return region is also reached through this selection process, after the iteration’s preceding reads.

Region transitions generally return to the dispatcher and therefore cause fresh iteration-level reads. The captured shared-state base, by contrast, is not visibly reloaded.

An unrecognized selector executes the same region as one explicitly listed selector: it sets `0x40000000` at shared offset 128 and dispatches again using the iteration-captured default transition value. It is not an explicit invalid-input return.

No bounded iteration count or guaranteed terminating path is established.

### External-read timing around calls

External-read timing is part of the observable contract. A dependency may change external data; the routine does not uniformly use either an invocation-wide snapshot or post-call reloads.

#### Flag notifications

The three regions that call F-d1228fe0890f first update a shared word. Their first call arguments come from the current iteration’s earlier captures:

| Shared update | First argument timing | Second argument |
|---|---|---:|
| Set `0x20000000` at offset 128 | Captured before region selection | 8 |
| Set `0x00400000` at offset 128 | Captured before region selection | 15 |
| Set bit zero at offset 132 | Captured before region selection | 15 |

Each region reads its external transition value **after** F-d1228fe0890f returns. Thus, the call receives a pre-call captured argument while the following dispatch transition can reflect the call’s changes to external data.

The same captured notification value is also installed into working state in the applicable region; it is not replaced by a post-call reload.

The `0x01000000` update and the default `0x40000000` update differ: neither contains a dependency call, and each uses a transition value captured before region selection.

#### Three-call processing region

The region containing F-f0468087e600, F-46b69b47db18, and F-9976748a38b8 has these established boundaries:

- F-f0468087e600 executes first and supplies the working handle.
- F-46b69b47db18 executes next and supplies an 8-bit value.
- After both calls return, an external multiplier is read and retained.
- The same external multiplier location is read again for scaling a working 32-bit value supplied to F-9976748a38b8.
- F-9976748a38b8 executes with that scaled value and the preceding 8-bit result.
- After the third call, its signed 16-bit result is encoded. Further external arithmetic values and transition data are read.
- The multiplier retained before the third call is reused in the later branch calculation alongside those post-call reads.

A change made by either of the first two dependencies can affect the multiplier reads. A change made by the third dependency can affect the later external reads, but does not retroactively replace the retained multiplier. The two multiplier accesses are distinct reads; their equality is not guaranteed under external mutation.

#### Arguments read after preceding calls

Three other sequences deliberately obtain external arguments after an earlier dependency returns:

- **F-7f457fab7c43 → F-065e64c01704:** the first call receives the handle. External values used to construct the sixteen arguments of the second call are read afterward.
- **F-a7278bfd65cb → F-8b2f26cdf7e0:** the first call receives the handle. External values used in the eleven-argument call are read afterward.
- **F-a860cee834af → unmapped dependency:** the first call receives the handle. The external second argument of the unmapped call is read afterward. That external value is also copied into working storage before the unmapped call.

These argument reads must not be moved before their preceding calls on the assumption that dependencies preserve external data.

For the unmapped call, the representation contains separate accesses to the same external location for the working copy and the call argument. Equality between those accesses requires stability not established here.

Other visible call regions also read transition data after their calls. For example, F-1967e534a137 is followed by external transition reads; F-11bc1f81fac4 and F-fdfa843140d0 are followed by external transition reads after status-related updates. In the F-fdfa843140d0 region, an externally scaled working value is computed before the call.

Where several external accesses occur within one expression, their precise relative evaluation order is not established by this representation. Their placement before or after a separately sequenced dependency call is nevertheless established.

### Shared-state tests and updates

Separate regions test the following masks at offset 128:

- `0x00400000`
- `0x01000000`
- `0x20000000`
- `0x40000000`

Another region tests bit zero at offset 132. Each test chooses between two external-data-derived transitions according to whether the masked result is zero. The resulting next region is unresolved without the external constants.

Update regions set each listed mask at offset 128 or bit zero at offset 132. Every update reads the applicable 32-bit word, ORs in its mask, and writes the result. No matching clearing operation is visible.

For the three notification regions, the shared-word write precedes F-d1228fe0890f. A dependency accessing that memory can therefore observe the updated flag.

These operations are not established to be atomic. They preserve unrelated bits from the value actually read, not necessarily concurrent changes occurring between the read and write. No rollback is visible.

### Workspace and handle state

One setup region assigns the automatic workspace address to both pointer slots and installs working values derived from retained inputs and external data.

A distinct region assigns encoded status `0x67` and writes byte value 1 at offset 248 from the working workspace pointer. Setup-before-use is not established for every path. If that pointer is still null or otherwise invalid, valid completion of the write is not established.

The workspace is passed to several dependencies. Their accessed ranges, initialization behavior, pointer retention, and bounds compliance are unknown. Its storage lifetime ends with the invocation; any safe pointer-retention policy requires an additional dependency contract.

Two calls replace the working handle:

- F-ea5d40b0d1b0 receives six arguments, including literals 2 and 10 and a workspace-related pointer.
- F-f0468087e600 receives six working arguments, including a workspace-related pointer and status-related values.

No immediate explicit null-handle check follows either assignment. Other dependencies receive the handle, but acquisition, transfer, release, cancellation, and validity rules are not established. No ownership guarantee follows from call ordering alone.

### Status and result arithmetic

Several dependencies supply signed 16-bit results. Encoding multiplies the result by 161 and retains the low eight bits. Decoding multiplies a signed-16-bit narrowing of stored status by 97 and retains the low eight bits.

Because 161 × 97 is congruent to 1 modulo 256, encoding followed by decoding recovers the original low byte when the stored value has not otherwise changed. It does not recover the original signed 16-bit result.

F-fdfa843140d0 supplies a 32-bit result that is retained separately. Its status encoding uses a signed-16-bit narrowing, then retains only the low byte after multiplication. Higher result bits do not survive in that encoded status.

Separate regions install status constants `0x67` and `0xA1`.

Branch and result calculations also use external 32-bit arithmetic values, products, complements, OR operations, narrowing, and shifts. Some select transitions using the high bit of an unsigned 32-bit intermediate. These operations do not establish conventional success predicates such as “zero succeeds” or “negative fails.”

Unsigned 32-bit truncation and logical shifts of unsigned intermediates must be preserved. The supplied representation does not fully establish the signedness of every external datum or the machine-level treatment of every mixed signed expression. Those remaining arithmetic details are **documented_only**; unbounded-integer arithmetic is not an established substitute.

## Side effects

Directly visible effects include:

- External-data reads at entry, before every region selection, and within regions.
- Shared 32-bit reads and writes at offsets 128 and 132 from the captured base.
- The conditional one-byte write through the working pointer at offset 248.
- Dependency calls with opaque arguments, workspace-related pointers, encoded or decoded status, and the working handle.
- Local state changes affecting later arguments, dispatch, and return values.

There are no explicit direct writes through the five inputs. Dependencies may interpret forwarded inputs as writable addresses, so caller-visible read-only behavior is not established.

No explicit allocator or identifiable release operation appears in the supplied body. Dependencies may allocate, release, modify globals, or retain pointers. Their effects must not be assumed absent.

Changes to external data can affect later iterations because dispatch baselines, transition data, and argument data are repeatedly read. Changes to the external location originally supplying the shared-state base do not visibly cause that base to be recaptured.

## Dependencies

Only opaque IDs are used for mapped dependencies.

| Dependency | Established interaction |
|---|---|
| F-fdfa843140d0 | Handle and decoded status input; retained 32-bit result, also narrowed for status encoding. |
| F-1967e534a137 | Handle input; no result visibly consumed; followed by external transition reads. |
| F-f0468087e600 | Six arguments; replaces handle; precedes the other two calls in its region. |
| F-46b69b47db18 | Working workspace pointer; supplies an 8-bit value. |
| F-9976748a38b8 | Three arguments, including a scaled 32-bit value and the preceding 8-bit result; supplies signed 16-bit status. |
| F-c8e906e0b5f9 | Working workspace pointer; supplies signed 16-bit status. |
| F-7f457fab7c43 | Handle input; precedes external argument reads for F-065e64c01704. |
| F-065e64c01704 | Sixteen arguments, including workspace pointer and externally derived values; supplies signed 16-bit status. |
| F-a7278bfd65cb | Handle input; precedes external argument reads for F-8b2f26cdf7e0. |
| F-8b2f26cdf7e0 | Eleven arguments, including literals 2 and 10 and workspace pointer last; supplies signed 16-bit status. |
| F-a860cee834af | Handle input; precedes external argument reads for the unmapped call. |
| F-d1228fe0890f | Iteration-captured first argument and literal 8 or 15; follows a shared flag write. |
| F-ea5d40b0d1b0 | Six arguments, including literals 2 and 10; replaces handle. |
| F-11bc1f81fac4 | Handle and decoded status input; supplies signed 16-bit status. |

One additional dependency has no supplied opaque ID. It receives two arguments after F-a860cee834af, and its result replaces the result accumulator. Its identity remains **documented_only**; it is not equated with any mapped dependency.

All dependency ownership, memory, synchronization, and error contracts remain unknown.

## Acceptance examples

These are conditional static criteria, not executed tests.

### Status information loss

If a signed 16-bit dependency result is −1, its encoded byte is `0x5F`; decoding produces `0xFF`. A result of 255 produces the same encoded byte. Acceptance requires preserving this low-byte behavior, not reconstructing which signed result occurred.

### Flag write and notification timing

Suppose the `0x20000000` update region executes, its shared word reads as `0x00000005`, and no intervening writer changes it. The direct write is `0x20000005`, before F-d1228fe0890f receives second argument 8.

If that call changes the external location supplying its first argument, the current call still uses the value captured before region selection. If it changes the transition-data location, the subsequent transition read observes the then-current value, subject to any other external mutation.

### Retained multiplier versus post-call arithmetic

Suppose the three-call processing region reaches F-9976748a38b8 after retaining multiplier value M. If that dependency changes the multiplier’s external location, the later expression still uses retained M where specified. Other arithmetic operands read after the call use their post-call values. Replacing every operand with a pre-call snapshot, or reloading every operand after the call, fails this distinction.

### Post-call argument construction

If F-7f457fab7c43 changes external data used to construct arguments for F-065e64c01704, those argument reads occur after the change. Capturing those values before F-7f457fab7c43 would violate the established sequence.

The corresponding requirement applies to F-a7278bfd65cb before F-8b2f26cdf7e0 and F-a860cee834af before the unmapped call.

### Per-iteration baseline

If a dependency changes the selector baseline, the next dispatcher iteration reads that location again before selecting its region. An invocation-wide cached baseline does not satisfy the visible behavior.

### Workspace marker and default dispatch

If the marker region executes with its working pointer addressing the automatic workspace, byte 248 becomes 1. No initialization of the other 255 bytes follows from that write.

If an unrecognized selector is selected, the function sets `0x40000000` at shared offset 128 and dispatches again. An immediate error return is not the established response.

## Uncertainties

The following remain **documented_only**:

- External-data values and the resulting reachable region graph.
- Termination, repetition, and complete cross-region call ordering.
- Input semantic types and validity constraints.
- Success, failure, and error-code meanings.
- Whether every workspace use follows pointer setup.
- Dependency memory effects, pointer retention, handle validity, and ownership.
- Synchronization, reentrancy, and concurrent-mutation guarantees.
- Exact machine semantics of unresolved mixed signed arithmetic.
- Relative ordering of external accesses within individual expressions.
- The identity and contract of the unmapped dependency.

These unknowns do not erase established sequencing boundaries. In particular, pre-selection captures, post-call argument reads, retained arithmetic values, post-call transition reads, and repeated selector-baseline reads are distinct source-visible behaviors.

## Evidence

This contract derives from static inspection of the supplied function representation and opaque dependency mapping. No runtime execution or tests have been performed.

The supplied material establishes widths, storage extent, memory offsets, masks, regional call order, explicit capture and reload boundaries, low-byte status arithmetic, and the unresolved dispatcher structure. It does not establish a domain-specific purpose, complete operational trace, dependency side-effect guarantees, or resource-lifetime correctness.
