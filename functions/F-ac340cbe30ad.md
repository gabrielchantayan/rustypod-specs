<!-- Trusted provenance: F-ac340cbe30ad; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ac340cbe30ad

## Purpose

F-ac340cbe30ad builds a destination object containing point storage, contour descriptors, circular point links, and direction-related metadata. It then invokes post-construction dependencies and initializes two embedded targets.

The point-and-contour interpretation is supported by coordinate-pair accesses, inclusive endpoint arithmetic, and circular links. The application-level meaning of the object and its classifications is not established.

The function returns a 32-bit status value. Storage acquisition is controlled by a shared, initially uninitialized status slot; final initialization is controlled by dependency return values. Failures do not trigger visible rollback or release.

This contract describes static evidence. No runtime tests have been performed.

## Interface

### Arguments

| Argument | Observed role |
|---|---|
| Destination address | Writable object address passed first to an external operation with length `0x80`, then used for fields and embedded initialization targets. |
| Source-descriptor address | Supplies signed counts and addresses of coordinates, attributes, and contour endpoints. Its address is eventually retained in the destination. |
| Auxiliary configuration base | A 32-bit address-like value used to form six addresses for embedded initialization. |
| Context-holder address | Supplies a 32-bit context value captured after the initial external operation. The holder address itself is retained later. |

There are no explicit checks for nullness, alignment, capacity, overlap, or address validity. Ordinary successful use requires storage compatible with all visible accesses and with the unknown dependency contracts.

### Source descriptor

| Byte offset | Width | Interpretation |
|---|---:|---|
| `0x00` | Signed 16 bits | Contour count |
| `0x02` | Signed 16 bits | Point count |
| `0x04` | 32 bits | Coordinate-array address |
| `0x08` | 32 bits | Attribute-array address |
| `0x0c` | 32 bits | Contour-endpoint-array address |

Coordinates occupy eight bytes per point: two 32-bit components. Attributes occupy one byte per point. Endpoints are signed 16-bit inclusive indices.

The first contour starts at point index zero. Each later contour starts immediately after the preceding endpoint. These interpretations describe the arithmetic, not validated input requirements.

### Destination fields

| Byte offset | Width | Direct use |
|---|---:|---|
| `0x00` | 32 bits | Published point count |
| `0x04` | 32 bits | Published contour count |
| `0x08` | 32 bits | First storage request’s returned value |
| `0x0c` | 32 bits | Second storage request’s returned value |
| `0x10` | 32 bits | Captured context |
| `0x14` | 32 bits | Retained source-descriptor address |
| `0x18` | 32 bits | Retained context-holder address |
| `0x1c` | Embedded target | First final initialization target |
| `0x44` | Embedded target | Second final initialization target |

Published counts are sign-extended from 16 bits. Loop conditions subsequently interpret these destination fields as unsigned 32-bit values.

This is not a complete object layout. The initial operation and later dependencies may affect additional bytes.

### Requested storage layouts

The first storage request specifies a `0x30`-byte element size.

| Point-record offset | Width | Direct use |
|---|---:|---|
| `0x00` | 32 bits | Previous-point address |
| `0x04` | 32 bits | Next-point address |
| `0x08` | 32 bits | Contour-descriptor address |
| `0x0c` | 32 bits | Flags |
| `0x14` | 8 bits | Incoming classification |
| `0x15` | 8 bits | Outgoing classification |

The function does not visibly copy coordinates into these records. Other bytes are not directly initialized by the visible construction logic.

The second request specifies eight-byte contour descriptors:

| Descriptor offset | Width | Direct use |
|---|---:|---|
| `0x00` | 32 bits | First-point address |
| `0x04` | 32 bits | Contour point count |

### Return and status-slot contract

Both storage requests receive the address of the **same local 32-bit status slot**. The function neither initializes it before the first request nor resets it before the second.

After each request, the returned storage value is published before the slot is examined. The slot is examined whether or not the dependency wrote it.

Consequently:

- Defined first-request status handling requires F-6dd003795db7 to initialize the slot before that examination.
- The second request is reached after the slot has been observed as zero. There is no intervening explicit reset.
- If the second request leaves the slot unchanged, zero remains the continuation value; this does not independently establish successful allocation.
- A nonzero observed slot value ends processing and is returned.

After construction, the first final initializer’s return value replaces the slot. If that value is zero, the second final initializer’s return value replaces it. The final slot value is returned.

Zero therefore means that the observed continuation checks passed. It does not prove allocation validity, geometry validity, zero-initialized storage, or undocumented dependency behavior.

## Behavior

### Initial operation, capture, and requests

The first external operation receives the destination address and `0x80`. Its semantics are unavailable. Clearing that region is plausible but not established.

After it returns, the function reads the context-holder’s 32-bit value and writes it at destination offset `0x10`. The captured value, rather than a later reread of either location, is used for both storage requests and both final embedded initializers.

The first F-6dd003795db7 call receives, in order:

- Captured context.
- Element size `0x30`.
- Zero.
- The source point count, read at this time and sign-extended to 32 bits.
- Zero.
- The shared status-slot address.

Its returned value is stored at destination offset `0x08`, then the slot is checked.

Only if that check observes zero does the function read the source contour count for the second request. That call has the same argument roles, except that the element size is eight and the count is the newly read, sign-extended contour count. Its returned value is stored at destination offset `0x0c`, then the same slot is checked again.

Neither returned address receives an independent null check.

### Count publication and construction captures

After both slot checks observe zero, the counts are read again:

1. Read the source point count and write its sign-extended value at destination offset `0x00`.
2. Then read the source contour count and write its sign-extended value at destination offset `0x04`.

These are not necessarily the counts supplied to the requests. Dependency-induced mutation may separate allocation-time values from publication-time values. Because the point-count write precedes the contour-count read, aliasing may also affect the value published as the contour count.

After publication, contour construction captures the destination point-storage field. Its descriptor cursor starts from the second request’s **retained returned pointer**, not a reread of destination offset `0x0c`.

Thus a dependency or overlapping write can make the destination storage fields differ from the addresses actually used during construction.

### Contour construction

Each contour-loop condition rereads destination offset `0x04` as an unsigned 32-bit bound. It is not a snapshot of either source count or the originally published count.

For each entered iteration, the source endpoint-array address is reloaded. The selected signed 16-bit endpoint is extended and increased by one to obtain an exclusive end index. Subtracting the preceding exclusive end yields the contour’s point count.

The descriptor writes occur in this order:

1. Write the contour point count at descriptor offset `0x04`.
2. Write the computed first-point address at descriptor offset `0x00`.

That order applies even if the two storage regions overlap other relevant objects.

For a positive contour count, the visible link-write order is:

- Write the first point’s contour-descriptor address.
- Write its previous link to the contour’s last point.
- For each remaining point, write the preceding point’s next link, then the new point’s previous link, then the new point’s contour-descriptor address.
- Write the final point’s next link back to the first point.

With ordinary nonoverlapping storage, this produces a circular doubly linked list. A one-point contour has both links directed to itself.

A zero or negative contour count still produces both descriptor writes but no link or ownership writes. The descriptor cursor advances eight bytes, and the exclusive end becomes the next contour’s starting index.

There is no endpoint validation: negative endpoints, repeated or descending endpoints, endpoints outside the point count, and incomplete coverage are not rejected. Overlapping storage may also alter the live contour-loop bound or future endpoint addresses.

### Point-processing captures and read timing

After contour construction, the function reloads destination offset `0x08` as the point-storage base. This may differ from the base captured for contour construction.

It then captures the source coordinate-array address once. That address is reused throughout point processing; the coordinate contents themselves are read as needed.

Each point-loop condition rereads destination offset `0x00` as an unsigned 32-bit bound. The point cursor advances by `0x30` per completed iteration, independently of later writes to the destination point-storage field.

For each point, the significant sequence is:

1. Read the point’s previous link.
2. Call F-870c4fcf7295 with its 32-bit difference from the captured point base and divisor `0x30`.
3. After that call returns, read the point’s next link.
4. Call F-870c4fcf7295 with that link’s difference from the same base and divisor.
5. After the second call returns, reload the source attribute-array address and read the current point’s attribute byte.

The returned values are used as coordinate indices. Division-like index recovery is inferred from these arguments; the dependency’s exact arithmetic is unavailable.

This sequencing matters if a dependency mutates links, source pointer fields, coordinates, flags, or destination count fields.

### Vectors, classifications, and flags

If attribute bit zero is clear, the entire 32-bit flags field is replaced with one. This is not an OR operation. If that attribute bit is set, this step does not write flags.

The incoming vector is the current coordinate pair minus the previous point’s coordinate pair. F-92f5794be4fc receives its two 32-bit components, and the low eight bits of its result are stored at point offset `0x14`.

Only after that call does the function read coordinates for the outgoing vector: next point minus current point. The dependency receives those components, and the low eight bits of its result are stored at offset `0x15`.

These vectors need not represent one coordinate snapshot. The first classification call can affect later coordinate reads. The component values already calculated for each vector are retained for any subsequent angular calls.

After the second classification call and its byte store, the function reads the 32-bit flags field. The flag decision uses:

- That captured flags value.
- The incoming classification byte reread from point offset `0x14`.
- The retained outgoing classification result, rather than a fresh read of offset `0x15`.

The distinction permits a dependency or aliased write to change the incoming byte used in the comparison.

The function sets flags bit one when:

- Captured flags bit zero is set; or
- Bit zero is clear and the compared classification bytes are equal but not four; or
- Bit zero is clear, both compared bytes are four, and the special comparison below succeeds.

Other cases perform no direct flags write. No decision branch explicitly clears bit one, although the earlier assignment of one can clear it.

For the first two setting cases, the write is the captured flags value ORed with two.

### Special comparison

When flags bit zero is clear and both compared classification bytes equal four:

1. F-39d3b95c505b receives the retained incoming vector.
2. F-39d3b95c505b receives the retained outgoing vector.
3. The two 32-bit results are subtracted.
4. A negative difference is conditionally negated.
5. A difference strictly greater than `0x00b40000` is replaced by `0x01680000` minus that difference.
6. The resulting signed value must be strictly less than `0x000b4000` to set bit one.

On this successful branch, the flags field is reread after both angular calls, and that fresh value is ORed with two. An unsuccessful comparison performs no direct flags write.

The constants suggest 16.16 angular units: a 360-unit turn, 180-unit half-turn, and 11.25-unit tolerance. Units and valid result ranges remain inferred. The operation performs one fold, not unrestricted modular normalization.

### Final initialization

After the point loop exits, the function:

- Stores the original source-descriptor address at destination offset `0x14`.
- Stores the original context-holder address at offset `0x18`.
- Calls F-fe1c70369baf with the destination and zero.
- Calls F-e711a790ae5e with the destination.
- Calls F-c9b21b2fc47d with the destination target at `0x1c`, auxiliary addresses at offsets `0x10`, `0x1c`, and `0x28`, and the captured context.
- If that result is zero, calls F-c9b21b2fc47d with the destination target at `0x44`, auxiliary addresses at offsets `0x34`, `0x40`, and `0x4c`, and the same captured context.

The first two dependency results are not checked. The first nonzero embedded-initializer result prevents the second initializer from running.

## Side effects

Direct writes affect the destination, point storage, and contour storage. No direct writes target the source descriptor, coordinate array, attribute array, endpoint array, or auxiliary configuration.

Aliasing can invalidate that separation. No overlap is excluded, including overlap between returned storage regions or between the destination and source. The initial external operation can affect inputs before their first read if they overlap its target.

The function retains addresses rather than visibly copying the source descriptor or context holder. Required lifetimes are unknown without their later consumers.

No release or restoration operation is visible:

- Second-request failure does not visibly release first-request storage.
- Final-initializer failure does not visibly release either storage region.
- Partially written descriptors, links, flags, and embedded state are not restored.

Dependencies may have additional effects. Their preservation of globals, caller-visible memory, or acquired resources must not be assumed.

## Dependencies

| Opaque identifier | Observed role | Documented-only unknowns |
|---|---|---|
| F-6dd003795db7 | Two storage requests sharing one status-slot address | Allocation semantics, status writes, initialization, capacity, ownership, cleanup |
| F-870c4fcf7295 | Converts point-address differences using divisor `0x30` | Exact arithmetic, signedness rules, rounding, exceptional inputs |
| F-92f5794be4fc | Classifies vector components; low byte is used | Classification meanings, zero-vector behavior, side effects |
| F-39d3b95c505b | Produces values for the special comparison | Units, range, overflow guarantees, side effects |
| F-fe1c70369baf | Post-construction operation with destination and zero | State changes, requirements, resource effects |
| F-e711a790ae5e | Subsequent post-construction operation | State changes, requirements, resource effects |
| F-c9b21b2fc47d | Initializes two embedded targets in sequence | Layout, status meanings, resource ownership, failure effects |

The initial external operation has no supplied opaque identifier. Only its destination argument, length `0x80`, and position before the context read are established.

## Acceptance examples

These are conditional static expectations, not executed tests. Except where mutation is the subject, assume valid nonoverlapping storage, stable relevant memory, sufficient capacity, and dependencies that provide the results stated.

1. **Three-point contour.** Point count three, contour count one, and endpoint two produce descriptor count three followed by the first-point address. Links form the forward cycle zero → one → two → zero and the matching reverse cycle. All ownership fields identify that descriptor.

2. **Single-point contour.** Counts of one and endpoint zero produce self-directed previous and next links. Both coordinate differences are zero. Classification and flags still depend on dependency results and initial flag contents.

3. **Attribute bit zero clear.** Flags are replaced with one. If subsequent dependencies leave flags unchanged, bit zero causes the final flag write to produce three.

4. **Equal ordinary classifications.** With attribute bit zero set, flags zero, and compared classifications both two, the function writes flags value two.

5. **Exact angular boundary.** With flags zero and compared classifications four, a folded result of exactly `0x000b4000` causes no flag write. One unit less sets bit one.

6. **First request leaves the slot unwritten.** The returned storage value is still published. No deterministic status result is established because the function did not initialize the slot. This is not specified as either success or failure.

7. **Second request leaves the slot unchanged.** If the first request established zero and the second leaves that slot unchanged, processing continues after publishing the second returned value. The function does not separately validate that value.

8. **Observed second-request failure.** A nonzero slot value after the second request is returned. Both storage values have been published, but this function has not yet explicitly published either count. No release is visible.

9. **Allocation and publication differ.** If the first request receives point count two but a dependency changes the source point count to three before publication, the destination publishes three. If its live bound remains three, point processing attempts three iterations; no allocation-capacity comparison intervenes.

10. **Mutation between link reads.** If the first F-870c4fcf7295 call changes the current point’s next link, the second call uses the changed link. The previous-link argument was already captured before the first call.

11. **Zero counts.** Both requests still occur. If their slot checks observe zero and the published bounds remain zero, both loops are skipped, but address retention and final initialization still occur.

12. **First embedded initializer fails.** Both post-construction dependencies have already run. The second embedded initializer is skipped, and the first initializer’s nonzero result is returned.

## Uncertainties

### Established arithmetic and input risks

Source counts and endpoints are signed 16-bit values. Published counts and addresses occupy 32 bits. Loop comparisons use unsigned destination counts, so a negative published count can become a very large bound.

Point offsets use stride `0x30`; coordinate offsets use stride eight; endpoint offsets use stride two. Pointer differences, vector components, angle differences, and relevant address calculations are represented as 32-bit quantities.

The static representation does not establish a portable overflow contract. In particular, coordinate subtraction, angular subtraction, negation of the most-negative signed value, and out-of-range address calculations cannot be assigned unrestricted mathematical behavior.

Valid ordinary operation requires compatible counts, endpoints, capacities, and links. None is validated. Live destination bounds and separately timed source reads prevent treating the input as one immutable snapshot.

### Inferred intent

Zero-initialized point storage would explain reliance on flags left untouched for some attributes, but allocation initialization is not established. Neither is clearing by the initial external operation.

Geometric contours, division-like index recovery, and fixed-point angular units are structural interpretations, not dependency guarantees.

### Documented-only unknowns

Complete layouts, alignment requirements, status-code meanings, ownership transfer, destruction responsibilities, dependency side effects, initial-operation semantics, and behavior outside representable arithmetic remain documented_only.

## Evidence

The supplied function body establishes field widths and offsets, call order, shared status-slot use, read and capture timing, live loop bounds, write ordering, branch conditions, constants, and the absence of explicit validation or rollback.

Dependency implementations were not supplied. The prior draft and feedback are not independent evidence for dependency semantics.

No runtime tests have been performed. Established static behavior, inferred intent, and documented-only unknowns are separated above.
