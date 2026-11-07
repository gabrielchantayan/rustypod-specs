<!-- Trusted provenance: F-5dec2cee97f2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5dec2cee97f2

## Purpose

F-5dec2cee97f2 coordinates three ordered operations using resources referenced by a context object and an input four-word region.

The established behavior is:

1. Query the first resource, construct a destination region anchored at the input’s first two coordinates, and invoke an operation with that resource.
2. Query the second resource, construct a destination region anchored at the input’s first and fourth coordinates, and invoke the same operation with the second resource.
3. Query a third resource and invoke a different operation with the remaining four-word region and an interior address within the context.

The arithmetic is consistent with placing a top piece, placing a bottom piece, and processing the region between them. That interpretation is **inferred intent**, not an established rendering contract. The dependency implementations are unavailable, so drawing, clipping, scaling, blending, and resource ownership are not established.

## Interface

The function has three inputs and no explicit return value.

| Input | Established representation | Use |
|---|---|---|
| Context | A 32-bit address-like value | Supplies resource words at byte offsets 500, 504, and 508; also supplies an interior address at byte offset 196. |
| Forwarded value | One opaque 32-bit word | Passed unchanged to the first two resource-operation calls. Its semantic type is unknown. |
| Input region | Address of four consecutive 32-bit words | All four words are read, in order, before the first dependency call. |

For this contract, the four input words are denoted **L**, **T**, **R**, and **B**, in increasing address order. These are descriptive labels, not proof of a particular rectangle convention.

The input region occupies 16 bytes:

| Byte offset | Word | Established use |
|---|---|---|
| 0 | L | Initial horizontal anchor for both constructed regions; first word of the final working region. |
| 4 | T | Initial vertical anchor; subsequently advanced before the final operation. |
| 8 | R | Copied into the working region but not otherwise used directly by this function. |
| 12 | B | Bottom anchor for the second constructed region; subsequently reduced before the final operation. |

The context’s relevant locations are:

| Byte offset | Access |
|---|---|
| 196 | Its address is passed to the final dependency. No direct read of the contents is shown. |
| 500 | A 32-bit resource word is read separately for the first query and first operation. |
| 504 | A 32-bit resource word is read separately for the final query and final operation. |
| 508 | A 32-bit resource word is read separately for the second query and second operation. |

These byte offsets are distinct: the resource-processing order is **500, 508, 504**, not increasing address order.

Necessary preconditions include readable input words, readable context resource words at each actual access, valid address arithmetic, and satisfaction of each dependency’s own argument requirements. There are no explicit checks for null pointers, invalid resources, malformed regions, or insufficient storage.

## Behavior

### Initial snapshot

Before calling any dependency, the function reads the input region’s four words in order and stores them in private working storage.

Consequently, the ordinary direct behavior does not reread the caller’s input region later. A change to that input memory after the snapshot does not, by itself, alter the working copy. This does not rule out effects through aliases or dependency behavior not visible in the supplied evidence.

Three kinds of temporary storage are used:

- A four-word working region initialized from the input.
- A four-word query-output area, reused for all three resource queries.
- A four-word copied source region and a separate 16-byte destination-description area, reused for the first two operations.

The function does not explicitly initialize the query-output area or destination-description area before handing them to their respective dependencies.

### First resource operation

The function reads the context word at byte offset 500 and calls F-c24767c70ad0 with:

1. The address of the four-word query-output area.
2. The resource word just read.

After that call returns, the function copies the query-output words into the separate source-region area, preserving their order.

Let the query result at this point be **q₀, q₁, q₂, q₃**. The function invokes F-66c32a9130af with the address of the 16-byte destination-description area and four scalar values:

- L
- T
- L + (q₂ − q₀)
- T + (q₃ − q₁)

The first two coordinates therefore come from the input snapshot, while the coordinate differences come from the query output.

Next, the function rereads the context word at offset 500 and calls F-945301cd184f with:

1. That newly read resource word.
2. The forwarded 32-bit input value.
3. The address of the copied source region.
4. The address of the destination-description area.
5. The constant value 255.

The source-region copy is made **before** F-66c32a9130af is called. The resource word used for F-945301cd184f is loaded **after** that constructor-like call. The two resource loads must not be treated as one cached load: intervening dependencies may change context state.

After F-945301cd184f returns, the function increases the working region’s second word by the difference between the fourth and second words of the copied source region **as they exist at that time**.

This timing matters. If F-945301cd184f changes that source-region storage, the subsequent adjustment reflects its post-call contents, not necessarily the original query result.

### Second resource operation

The function reads the context word at byte offset 508 and queries it through F-c24767c70ad0, reusing the query-output area.

It then copies all four query-output words into the source-region area, replacing the previous source-region contents.

For this second query, let the returned words again be **q₀, q₁, q₂, q₃**. The function invokes F-66c32a9130af with the destination-description area and these four scalar values:

- L
- B − (q₃ − q₁)
- L + (q₂ − q₀)
- B

Here B is still the input snapshot’s fourth word. The first operation has adjusted the working second word, not the working fourth word.

The function then rereads the context word at byte offset 508 and calls F-945301cd184f with the same argument roles as before: resource word, forwarded value, copied source region, destination description, and 255.

After that call returns, the function reduces the working region’s fourth word by the difference between the fourth and second words of the copied source region **at that post-call point**.

There is no direct check that the resulting working second word is less than or equal to the resulting working fourth word.

### Final resource operation

The function reads the context word at byte offset 504 and calls F-c24767c70ad0 with that resource and the reused query-output area.

Unlike the first two phases, this query output is not copied into the separate source-region area. No call to F-66c32a9130af occurs in this phase.

The function rereads the context word at byte offset 504 and calls F-168345203dbe with these arguments, in order:

1. The newly read resource word.
2. The context address plus 196 bytes.
3. The query-output area.
4. The working region.
5. The constant value 1.
6. The constant value 0.

The second argument is an **interior address**, not the 32-bit contents stored at that address.

Subject to ordinary dependency behavior and representable arithmetic, the working region supplied to this final call has:

- First word: L.
- Second word: T plus the first source region’s post-operation vertical difference.
- Third word: R.
- Fourth word: B minus the second source region’s post-operation vertical difference.

The function then returns without an explicit result.

### Arithmetic and boundary behavior

Coordinates and query words are represented as 32-bit integer quantities. Differences and subsequent additions or subtractions are shown in that width; there is no visible widening step.

The evidence does not establish a portable overflow rule. Mathematical examples in this contract therefore assume every intermediate subtraction and addition fits in signed 32-bit range. Behavior outside that range is **documented_only**, not promised to wrap, saturate, or reject the input.

No direct logic:

- Normalizes reversed coordinates.
- Rejects zero or negative dimensions.
- Limits either constructed region to the input’s R coordinate.
- Prevents the first and second regions from overlapping.
- Skips operations for an empty or inverted remaining region.
- Clips, scales, retries, or selects a fallback.

Whether dependencies perform any such actions is unknown.

## Side effects

The function directly writes its own temporary storage. No direct store to the caller’s input region or context contents is shown.

It nevertheless exposes writable-looking addresses to dependencies:

- Query-output storage to F-c24767c70ad0.
- Destination-description storage to F-66c32a9130af.
- Source-region and destination-description storage to F-945301cd184f.
- Query-output storage, working-region storage, and the context interior address to F-168345203dbe.

The dependency signatures do not establish read-only access. In particular, the first two source-region buffers may be modified, and their post-call contents affect later arithmetic.

The direct dependency-call order is fixed:

1. F-c24767c70ad0.
2. F-66c32a9130af.
3. F-945301cd184f.
4. F-c24767c70ad0.
5. F-66c32a9130af.
6. F-945301cd184f.
7. F-c24767c70ad0.
8. F-168345203dbe.

There is no explicit allocation, release, reference-count operation, rollback, or cleanup in this function. This does not establish that dependencies allocate nothing or release nothing.

Temporary addresses refer to storage whose ordinary lifetime ends when this function returns. The evidence shows no transfer of ownership or arrangement extending that lifetime. Whether dependencies retain these addresses is unknown; safe deferred use would require a separate, unavailable lifetime contract.

No dependency return value is inspected. There is no explicit error result or recovery branch. Faults, exceptions, failure signaling through external state, and partial effects are governed by unavailable dependency and runtime behavior.

## Dependencies

| Opaque ID | Established role in this function | Unknown contract |
|---|---|---|
| F-c24767c70ad0 | Called three times with a four-word output area and a resource word. The output is subsequently consumed. | Whether all four words are always written; coordinate meaning; error behavior; external side effects. |
| F-66c32a9130af | Called twice with a 16-byte destination area and four computed scalar arguments. | Output encoding, validation, full initialization, and external side effects. |
| F-945301cd184f | Called twice with a resource, forwarded word, two temporary-region addresses, and 255. | Rendering semantics, buffer mutation, ownership, clipping, interpretation of 255, and errors. |
| F-168345203dbe | Called once with the third resource, a context interior address, query output, working region, 1, and 0. | Processing semantics, mutation, ownership, interpretation of the constants, and errors. |

The value 255 may suggest maximum intensity or opacity, but that meaning is not established. The constants 1 and 0 likewise have no established semantic names.

No dependency is assumed to preserve globals, preserve resource fields, or own or release passed memory.

## Acceptance examples

These are conditional acceptance examples derived from static evidence. They are not runtime test results.

### Ordinary non-overflowing geometry

Assume:

- Input words are (10, 20, 110, 100).
- The first query produces (2, 3, 32, 11).
- The second query produces (5, 7, 45, 19).
- Dependencies do not mutate the copied source regions.
- No other dependency effect changes relevant temporary state.

The first F-66c32a9130af invocation receives coordinates:

- (10, 20, 40, 28).

The second receives:

- (10, 88, 50, 100).

The final working region is:

- (10, 28, 110, 88).

The query at offset 504 still occurs, and its actual output is passed separately to the final dependency.

### Input right boundary does not constrain the first two constructions

With input (10, 20, 15, 100) and the same query results, the constructed coordinates remain:

- (10, 20, 40, 28).
- (10, 88, 50, 100).

The final working region retains 15 as its third word. No direct clipping to that boundary occurs in this function.

### Post-operation source mutation changes the remaining region

Assume the first query’s vertical difference is 8, but F-945301cd184f changes its source-region words so that the post-call vertical difference is 12.

The first constructed destination still uses the earlier query difference of 8. The working second word advances by 12.

Any contract that always advances it by the original query height would omit observable behavior supported by the call and read order.

### Resource changes between query and operation

If an intervening dependency changes the context word at offset 500, the first F-945301cd184f call receives the updated word loaded after destination construction.

The queried resource and the operated-on resource are therefore not guaranteed to be identical. The same reload distinction applies at offsets 508 and 504.

### Insufficient remaining height

Assume input (0, 0, 100, 10), first post-operation vertical difference 8, and second post-operation vertical difference 7.

The final working region is (0, 8, 100, 3). The final call still occurs; no direct empty-region or inversion check intervenes.

## Uncertainties

The following are **documented_only**:

- Exact semantic types of the context resource words and forwarded value.
- Coordinate convention, signedness at external interfaces, and inclusive versus exclusive endpoints.
- Overflow behavior outside representable signed 32-bit arithmetic.
- Whether query and construction dependencies fully initialize their output areas.
- Dependency mutation, global-state effects, ownership, retention, and failure behavior.
- Whether the operations render anything and, if so, how clipping, scaling, compositing, and invalid geometry are handled.
- The extent and structure of the context subobject addressed at byte offset 196.
- Valid aliasing relationships among caller-provided storage, context storage, and resources.

These unknowns do not justify treating dependencies as pure, ignoring resource reloads, or replacing post-call source-region reads with earlier query values.

## Evidence

The established claims come from the supplied static function representation: four initial input reads, explicit context offsets, copied four-word query results, coordinate arithmetic, dependency arguments, post-operation adjustments, and fixed call order.

The top-piece/bottom-piece/remaining-region interpretation is inferred from those calculations, not independently verified.

No dependency implementation, runtime trace, or runtime test result was supplied. No runtime tests have been performed.
