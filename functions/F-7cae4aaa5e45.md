<!-- Trusted provenance: F-7cae4aaa5e45; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7cae4aaa5e45

## Purpose

F-7cae4aaa5e45 coordinates a stateful operation using a captured context, a request, and an optional requested position. It can attempt continuation from request state, select another position, or invoke discovery. Accepted paths may publish a result record and obtain additional values for an optional data-related operation.

Established behavior includes:

- Two immediate-return conditions that do not perform this function’s output clearing.
- Branch selection using a dependency-produced flag.
- Conditional position lookup and discovery.
- A continuation-specific increment of a 32-bit context field.
- Publication of a 16-bit quantity when the internal result classification is exactly 1.
- A dependency invocation with source-like and destination-like arguments and a third argument restricted to 0–65,535.
- Optional result-record writes followed by another dependency invocation.
- Selective output clearing on common errors, with status `0x30A` translated to `0x20`.

“Continuation,” “lookup,” “discovery,” and “metadata” describe inferred roles. The supplied body does not establish the application domain or the actual effects of the apparent transfer dependency. In particular, a bounded quantity argument does not establish a bound on memory modified.

## Interface

The function accepts six arguments and returns a 32-bit integer status. Visible addresses and address-like values use a 32-bit model.

1. **Context reference**

   A nullable address of a 32-bit context value. A null address causes immediate return of −50. A zero value read through a non-null address causes immediate return of a stored status whose numeric value is unavailable.

   A nonzero captured value is subsequently used as the context address. The context-reference address is not reread after dependency calls.

2. **Request address**

   A nonzero address-like value. A zero value causes immediate return of −50.

   Directly relevant request locations are:

   | Byte offset | Use |
   |---|---|
   | 4 | Read as a 32-bit working position on the continuation branch |
   | 28 | Address supplied to several dependencies as embedded request data |

   The complete request layout is unknown.

3. **Requested position**

   A 32-bit integer. The value −1 disables explicit-position comparison and lookup. Other values are used without visible range validation.

4. **Optional data-operation descriptor**

   A nullable address of three consecutive 32-bit fields:

   | Byte offset | Use |
   |---|---|
   | 0 | Destination-like argument to F-70a021fc7c24 |
   | 4 | First capacity-like factor |
   | 8 | Second capacity-like factor |

   The factors participate in a 32-bit product. Their units and the validity requirements for the destination-like value are unknown.

5. **Optional quantity output**

   A nullable address of writable 16-bit storage. For classification 1, it receives the low 16 bits of a dependency-provided 32-bit quantity before the descriptor-based adjustment. On the common error path it receives zero.

6. **Optional result-record output**

   A nullable address of writable storage with these established direct accesses:

   | Byte offset | Width | Accepted-path direct write | Common-error direct write |
   |---|---:|---|---|
   | 0 | 32 bits | Context field at offset 52 | Zero |
   | 4 | 32 bits | Working position | Zero |
   | 8 | 16 bits | First working entry identifier | Zero |
   | 10 | 16 bits | None | Zero |
   | 12 | 32 bits | None | Zero |
   | 16 | 16 bits | None | Zero |
   | 18 | 16 bits | None | Zero |
   | 28 | 16 bits | No direct write; address supplied to a dependency | Zero |

   F-70a021fc7c24 also receives the address at record offset 28 as a destination-like argument. Its access extent is unknown, so this table does not establish the record’s required total size.

**Preconditions.** Every enabled direct access requires valid storage. Each dependency may impose additional requirements not supplied here. The body does not validate alignment, address ranges, object extents, or overlap. Output aliasing is not rejected and can change later reads and dependency-visible contents.

## Behavior

### Entry and initial classification

The context-reference address and request address are checked before reading the context value. If either is zero, the function returns −50 without directly writing optional outputs.

Otherwise, the function reads the context value. If it is zero, the function immediately returns the stored, numerically unknown status. This path also performs no direct optional-output writes.

For a nonzero captured context, an internal result classification starts at zero. F-60be901ce2de receives the captured context, request address, and a four-byte local output area.

- A nonzero return enters common error handling with that status.
- A zero return permits inspection of the first output byte.
- The other three bytes are not directly inspected.

### Continuation branch

A nonzero flag byte selects continuation.

The function reads the 32-bit request field at offset 4 into the working position, then calls F-5f9a16493218 with the captured context, that position, and a five-word working descriptor.

If this lookup returns nonzero, the function does not immediately propagate that status. It calls F-6e3669b482ce on the request and proceeds to position selection and discovery.

If the lookup returns zero, the descriptor’s first word is treated as an address. Eligibility for F-8a92d5d60703 requires both:

- The byte at that address plus 8 compares equal to −1, corresponding to `0xFF` under the displayed signed-byte interpretation.
- The 16-bit field at that address plus 10 is nonzero.

The second field is inspected only if the byte condition succeeds. If eligible, F-8a92d5d60703 receives the context, descriptor-derived address, request address plus 28, and a two-element 16-bit working area. Its return becomes the result classification.

If eligibility fails or that return is zero, the function calls F-b0f922796692 with the context and descriptor:

- A nonzero return enters common error handling.
- A zero return is followed by F-6e3669b482ce on the request, then position selection and discovery.

If F-8a92d5d60703 returns nonzero, discovery is bypassed. The function reads the 32-bit context field at offset 100, increments it, and writes it back. This occurs after the conditional dependency returns; the value need not equal its entry-time value.

Classification 1 then selects additional-data retrieval. Any other nonzero classification skips that retrieval and proceeds to optional record publication.

The displayed signed increment does not establish defined source-language behavior when the field is at its signed maximum. Wrapping must not be promised as an unconditional contract.

### Position selection and discovery

This path is entered when the initial flag is zero or continuation is abandoned.

If the requested position is not −1, it is compared with the working position. When they differ, the requested position becomes the working position and F-5f9a16493218 is called.

On direct entry with a zero flag, the supplied body has not visibly initialized the working position before this comparison. Consequently, a requested position other than −1 exposes an unresolved read; no default working position may be assumed.

If the explicit-position lookup returns zero, the same descriptor eligibility checks apply:

- An eligible F-8a92d5d60703 return of nonzero bypasses discovery.
- Otherwise F-b0f922796692 is called. A nonzero return enters common error handling; zero permits discovery.

A failed explicit-position lookup proceeds to discovery without a visible intervening descriptor-processing call. If the requested position equals the working position, explicit lookup is skipped and discovery is still performed.

Unlike successful continuation, a nonzero classification obtained through explicit-position lookup does not directly increment context offset 100.

F-6e3669b482ce’s return is not used to choose these paths or returned independently. The subsequent discovery result replaces the temporary status.

F-5d3e17c1b8e7 receives, in order:

1. Captured context.
2. Request address plus 28.
3. A 128-byte local scratch area.
4. Working-position output address.
5. Five-word working descriptor.
6. Two-element 16-bit working area.

Its return selects the next action:

- **Zero:** set classification to 1 and perform additional-data retrieval.
- **`0x20`:** proceed directly to optional record publication and final descriptor processing.
- **Any other value:** enter common error handling.

Every visible path reaching discovery has classification zero. It starts at zero, and either conditional operation returning nonzero immediately bypasses discovery. Therefore, discovery returning `0x20` reaches final processing with classification zero. If final processing succeeds, the final return is necessarily `0x20`.

Discovery’s `0x20` does not itself invoke additional-data retrieval or write the quantity output.

### Classification-1 quantity and data-related operation

For classification exactly 1, F-f66a00df5276 receives:

- Captured context.
- The descriptor’s first word.
- The first 16-bit working entry identifier.
- Output addresses for a metadata-source-like value, a data-source-like value, and a 32-bit quantity.

The function does not inspect this dependency’s return value. Subsequent use requires its output cells to have suitable values; that requirement is not independently validated.

If the quantity output is enabled, the low 16 bits of the returned quantity are written there first.

If the data-operation descriptor is enabled:

1. Its fields at offsets 4 and 8 are read and multiplied.
2. The product is compared as an unsigned 32-bit value with the low 16 bits of the returned quantity.
3. If smaller, the factor expression is evaluated again and replaces the working quantity.
4. F-70a021fc7c24 receives the data-source-like value, descriptor field 0 as a destination-like value, and the low 16 bits of the resulting quantity.

For stable factor fields, the third argument is the smaller of the unsigned 32-bit product and the original quantity’s low 16 bits. It is always in 0–65,535. The quantity output reports the original low-16-bit value, not this adjusted argument.

The factor calculation is not visibly widened or saturated. A mathematical product exceeding 32 bits must not be treated as an unlimited capacity. The conditional reevaluation also means the source does not establish one immutable capacity read.

The call is made even when its third argument is zero. No destination-like-value validation is visible. Actual copying, transfer units, termination, and bounds on memory modified remain unknown.

### Result publication and final processing

If a result-record output is enabled, the function performs these operations in order:

1. Read context offset 52 and write that 32-bit value to record offset 0.
2. Write the working position to offset 4.
3. Write the first 16-bit entry identifier to offset 8.
4. Invoke F-70a021fc7c24 with a source-like value and record address plus 28.

For classification 1, the source-like value comes from F-f66a00df5276. Otherwise it is request address plus 28.

This invocation has only two visible arguments. No explicit quantity is supplied. Neither an access extent nor equivalence with the three-argument invocation is established.

After optional publication, F-b0f922796692 is called with the context and descriptor, even when the record output is disabled.

- A nonzero return enters common error handling.
- On zero, classification zero returns `0x20`; any nonzero classification returns zero.

Thus negative as well as positive nonzero classifications can produce final status zero without additional-data retrieval, provided classification is not 1 and final processing succeeds.

### Common error handling

The current error status is retained while enabled outputs are cleared.

First, the 16-bit quantity output is set to zero if present. Then, if the result record is present, zero writes occur in this order:

| Offset | Width |
|---|---:|
| 0 | 32 bits |
| 4 | 32 bits |
| 8 | 16 bits |
| 10 | 16 bits |
| 12 | 32 bits |
| 16 | 16 bits |
| 18 | 16 bits |
| 28 | 16 bits |

This is not a whole-record clear. Bytes 20–27 and bytes after offset 29 receive no direct clearing write.

Finally, status `0x30A` is returned as `0x20`; every other status is returned unchanged. This translation does not make the preceding error path equivalent to an accepted discovery path.

## Side effects

Direct writes comprise the continuation-specific context increment, quantity publication or clearing, and the enumerated result-record writes. The data-operation descriptor is not directly modified.

Dependencies may modify context fields, requests, pointed-to storage, global state, or other resources. Their preservation properties are not established.

Failure is not transactional. Final descriptor processing can fail after record writes, dependency invocations, or the context increment. Common error handling does not restore the counter or undo dependency effects. In particular, if F-70a021fc7c24 modifies memory, this function provides no rollback of that modification.

Aliasing makes ordering significant:

- Quantity publication precedes capacity-factor and destination-like-field reads.
- The three record writes precede the record-related dependency invocation.
- Final descriptor processing follows publication.
- Error clearing follows earlier activity and can overwrite overlapping inputs.

No explicit allocation or deallocation appears. Local working storage is passed to dependencies, but pointer retention, resource acquisition, release, and ownership transfer are unknown. F-b0f922796692 must not be assumed to release resources merely because it occurs at abandonment and completion points.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-60be901ce2de | Supplies initial status and flag output |
| F-5f9a16493218 | Receives a position and five-word descriptor output area |
| F-8a92d5d60703 | Conditionally receives an eligible descriptor-derived address and supplies classification |
| F-6e3669b482ce | Operates on the request when continuation is abandoned |
| F-5d3e17c1b8e7 | Receives request data, scratch storage, and working output areas for discovery |
| F-f66a00df5276 | Supplies two source-like values and a 32-bit quantity for classification 1 |
| F-70a021fc7c24 | Receives source-like and destination-like arguments; one invocation additionally supplies a bounded quantity |
| F-b0f922796692 | Processes the descriptor at selected abandonment points and final completion; nonzero enters common error handling |

These are call-site contracts only. They do not establish dependency internals, purity, ownership, or transfer semantics.

## Acceptance examples

The following are conditional static cases, not runtime observations.

1. **Invalid entry address.** A null context-reference address or zero request address returns −50 without this function directly clearing optional outputs.

2. **Zero captured context.** Valid entry addresses with a zero context value return the stored, numerically unknown status. Neither zero nor `0x20` may be substituted.

3. **Initial dependency error.** If F-60be901ce2de returns `0x30A`, enabled outputs receive the specified error clears and the final status is `0x20`.

4. **Adjusted quantity argument.** With classification 1, returned quantity 100, stable factors 8 and 4, and both relevant outputs enabled, publish quantity 100 and invoke F-70a021fc7c24 with third argument 32. This does not establish that 32 bytes are copied.

5. **Quantity narrowing.** With returned quantity 65,538 and stable product 10, publish 2 and supply third argument 2. The comparison uses the quantity’s low 16 bits, not 65,538.

6. **Zero quantity argument.** With classification 1, an enabled descriptor, and stable product zero, invoke the dependency with third argument zero. Do not infer that the dependency performs no accesses.

7. **Non-unit continuation classification.** An eligible continuation returning classification 2 increments context offset 100, skips F-f66a00df5276, and uses request address plus 28 for optional record publication. Successful final processing returns zero.

8. **Accepted discovery status.** With requested position −1, discovery returning `0x20`, suitable dependency-populated working outputs, and successful final processing, return `0x20`. Classification is zero, additional-data retrieval is skipped, and the quantity output receives no direct write.

9. **Late error.** If final descriptor processing fails after publication, perform the specified clears and status translation. Earlier dependency effects and the counter increment are not rolled back.

10. **Uninitialized comparison exposure.** With an initial zero flag and requested position other than −1, deterministic explicit-position selection cannot be required from this body because the compared working position lacks visible initialization.

## Uncertainties

The following remain **documented_only**:

- Numeric value and provenance of the stored zero-context status.
- Application meaning of positions, classifications, identifiers, and embedded request data.
- Complete object layouts and required storage extents.
- Outputs guaranteed by dependencies, particularly discovery returning `0x20`.
- Actual effects, transfer units, access bounds, and calling convention of F-70a021fc7c24.
- Dependency mutation, ownership, lifetime, resource-release, and pointer-retention contracts.
- Defined source-language behavior for overflowing arithmetic.
- Explanation or external constraints for the uninitialized working-position comparison.
- Endianness, alignment requirements, and safety under aliasing or concurrent modification.

Classification after discovery returns `0x20` is not unresolved: it is zero under the visible control flow.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping. Established claims concern visible control flow, widths, offsets, argument construction, reads, writes, arithmetic expressions, and return handling.

Domain terminology and apparent transfer roles are inferred intent. Dependency effects not established by the supplied body remain explicitly unknown.

No dependency implementations or runtime observations were supplied. No runtime tests have been performed.
