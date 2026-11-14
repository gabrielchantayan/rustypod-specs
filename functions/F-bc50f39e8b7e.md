<!-- Trusted provenance: F-bc50f39e8b7e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bc50f39e8b7e

## Purpose

F-bc50f39e8b7e dispatches a rectangular transfer between two memory-backed surfaces. Depending on a format selector and two representation selectors, it either copies bytes directly, performs per-element transfers through dependency pairs, or delegates the transfer to another function.

The established behavior includes extent checks, dimension limiting, representation-dependent address calculations, traversal order, and dependency selection. The meanings of converted components, packed-bit layouts, alignment policy, and delegated transfer behavior are not established by the supplied material.

“Source,” “destination,” “width,” and “height” below describe their apparent roles. In particular, several height-like arguments are represented as pointer-shaped values, not as conventional integer dimensions. Their numerical use is established; their original API types are not.

## Interface

The function has 17 positional arguments and no return value.

| Position | Contract name | Observed representation and use |
|---|---|---|
| 1 | Source base | 32-bit address-bearing value; shown as signed integer storage. |
| 2 | Source width | Unsigned 32-bit value. |
| 3 | Source height | Pointer-shaped 32-bit value used in comparisons and numerical limiting. |
| 4 | Source horizontal offset | Unsigned 32-bit value. |
| 5 | Source vertical offset | Pointer-shaped 32-bit value used numerically in row-address calculations. |
| 6 | Requested width | Unsigned 32-bit value. |
| 7 | Requested height | Pointer-shaped 32-bit value used as a numerical row count. |
| 8 | Destination base | 32-bit address-bearing value; shown as signed integer storage. |
| 9 | Destination width | Unsigned 32-bit value. |
| 10 | Destination height | Pointer-shaped 32-bit value used in comparisons and numerical limiting. |
| 11 | Destination horizontal offset | Unsigned 32-bit value. |
| 12 | Destination vertical offset | Pointer-shaped 32-bit value used numerically in row-address calculations. |
| 13 | Format selector | 32-bit selector; recognized values are 0 through 7. |
| 14 | Source representation | 32-bit selector. |
| 15 | Destination representation | 32-bit selector. |
| 16 | Source auxiliary argument | Pointer-to-pointer-shaped value forwarded on delegated paths. |
| 17 | Destination auxiliary argument | Pointer-to-pointer-shaped value supplied to destination-stride calls and delegated paths. |

Three representation values are relevant:

- **U:** the literal value `0x1401`.
- **P:** a 32-bit value read from one global configuration location.
- **Q:** a 32-bit value read from another global configuration location.
- A further literal representation value, **R**, is `0x8034`.

P and Q are symbolic names for global values, not established numeric constants. They are not necessarily distinct from each other or from the literals.

There is no success indicator, byte count, or error code. A return may mean that the transfer completed, that a dependency returned, or that the function rejected the request without transferring anything.

For safe inline execution, callers must provide readable and writable storage covering every address actually reached, valid dependency-specific auxiliary state, and address arithmetic that remains meaningful in the target’s 32-bit execution model. The function does not establish these conditions.

## Behavior

### Entry checks and effective dimensions

The function returns before calling dependencies or accessing surface bytes unless all four conditions hold:

1. Source width is greater than source horizontal offset.
2. Source height is greater than source vertical offset.
3. Destination width is greater than destination horizontal offset.
4. Destination height is greater than destination vertical offset.

The width comparisons are unsigned. Height-like comparisons operate on the supplied pointer-shaped values; interpreting them as ordinary dimensions requires the numerical-address interpretation used by the rest of this function.

If the requested width is zero, or the requested height has the null representation, the function also returns immediately.

Otherwise, the effective width is the minimum of:

- Requested width.
- Destination width.
- Source width.

The effective height is the numerically smallest of:

- Requested height.
- Destination height.
- Source height.

**Offsets are not subtracted when limiting either dimension.** The entry checks establish that each starting coordinate is inside its nominal surface; they do not establish that the full transfer fits after that coordinate. The caller must not rely on clipping to the remaining width or height.

The function also forms destination-address intermediates equivalent to twice the destination horizontal offset plus destination base, and twice the destination width. These calculations occur before format dispatch, although some routes never use their results.

### Dispatch groups

| Format selector | Route |
|---|---|
| 0 or 1 | Direct one-byte transfer, ignoring both representation selectors. |
| 2 | Delegate to F-2ed72e31a28f, ignoring both representation selectors. |
| 3 or 5 | Three-byte or two-byte route selected by U and P. |
| 4, 6, or 7 | Four-byte or two-byte route selected by U, Q, and R. |
| Any other value | Return without a transfer dependency or direct surface access. |

Unsupported representation combinations also return without a transfer. Some such paths have already performed address arithmetic, but they do not call the stride dependency or enter a transfer loop.

### Stride and address conventions

Every inline transfer path obtains source stride first, then destination stride, by calling F-082b668da333.

The source call supplies the nominal source row size in bytes. The destination call supplies the nominal destination row size in bytes and the destination auxiliary argument. The supplied evidence does not resolve the dependency’s apparent argument-count inconsistency or establish whether it uses either argument as alignment or storage state.

For a side using \(b\) bytes per element, the nominal row size is:

\[
b \times \text{surface width}
\]

The first processed address on that side is:

\[
\text{base} + \text{stride} \times \text{vertical offset}
+ b \times \text{horizontal offset}
\]

These equations describe the observed address construction, not checked mathematical arithmetic. The implementation uses 32-bit operations and address-bearing conversions.

The source-stride call precedes the destination-stride call. The function provides no evidence that either dependency preserves globals or memory.

### Direct one-byte transfer

Formats 0 and 1 use one byte per element on both sides.

After obtaining both strides and constructing the initial addresses, the function processes effective-height rows, each containing effective-width bytes. For every byte:

1. Read the current source byte.
2. Advance the source cursor by one byte.
3. Write that byte to the current destination address.
4. Advance the destination cursor by one byte.

At each row boundary, it adjusts each cursor by its stride minus effective width. Thus, when arithmetic is ordinary and non-wrapping, successive row starts differ by their respective strides.

This is a forward, immediately interleaved read/write transfer. It does not snapshot a row or the complete rectangle. Overlapping regions can therefore cause earlier writes to change later source reads.

Both representation selectors and the source auxiliary argument are unused on this route.

### Three-byte/two-byte routes

Formats 3 and 5 have the following dispatch, assuming the relevant comparisons observe the stated value of P:

| Source representation | Destination representation | Source bytes/element | Destination bytes/element | Per-element dependencies |
|---|---|---:|---:|---|
| U | U | 3 | 3 | F-4ee07aa0986e, then F-e2db7eabd42d |
| U | P | 3 | 2 | F-4ee07aa0986e, then F-65573ddce6ca |
| P | U | 2 | 3 | F-a0d7716b0f77, then F-e2db7eabd42d |
| P | P | Delegated | Delegated | F-2ed72e31a28f |
| Other | Any unsupported combination | — | — | None |

Literal U comparisons take precedence over P comparisons. For example, if P equals U, a U-to-U request selects the three-byte inline route, not delegation.

On each inline route, the function obtains strides and constructs initial addresses using the byte counts in the table. It then invokes the listed pair once per element, in row-major order:

- The first dependency receives references to temporary state and the source cursor.
- The second receives references to temporary state, the destination cursor, and the first dependency’s intermediate result.

The function does not itself contain the element-level surface loads or stores on these routes. Cursor advancement and conversion semantics belong to the dependencies.

After every row, including the final row, it adjusts each cursor by:

\[
\text{stride} - \text{effective width} \times
\text{that side's bytes per element}
\]

Consequently, the familiar row-start spacing follows only if the dependencies advance their cursors by the expected element sizes.

### Four-byte/two-byte routes

Formats 4, 6, and 7 share this dispatch:

| Source representation | Destination representation | Route |
|---|---|---|
| U | U | Inline 4-byte to 4-byte route |
| U | Q | Inline 4-byte to 2-byte route |
| U | R | Inline 4-byte to 2-byte route |
| Q | U | F-dc9e9ffc7266 |
| Q | Q | F-2ed72e31a28f |
| Q | R | F-5bbebe570c33 |
| R | U | F-01d5ec758062 |
| R | Q | F-26417011e1c9 |
| R | R | F-2ed72e31a28f |
| Unsupported combination | — | Return |

The table is subject to comparison precedence:

- Source U is checked first.
- For source U, destination U is checked before Q, and Q before R.
- Otherwise, source Q is checked before source R.
- Within source Q, destination U precedes Q, which precedes R.
- Within source R, destination U precedes Q, which precedes R.

If Q equals a literal selector, the earlier matching branch wins.

The inline per-element dependency pairs are:

| Route | First dependency | Second dependency |
|---|---|---|
| 4-byte to 4-byte | F-789728274fbc | F-94989df6c6d2 |
| 4-byte to 2-byte selected by Q | F-789728274fbc | F-d212779ab59f |
| 4-byte to 2-byte selected by R | F-789728274fbc | F-2db8dd7f4e5a |

Stride acquisition, address construction, pair ordering, and row-end adjustments follow the same rules as the three-byte/two-byte inline routes. In the four-byte-to-four-byte route, the source stride is retained separately for row-end adjustment.

No component order, byte order, alpha interpretation, rounding rule, or packed-field layout is established.

### Delegated routes

Before delegating, the function retains the original bases, surface dimensions, and offsets, but uses the limited transfer width and height.

The visible call supplies the first nine arguments. Additional destination geometry and auxiliary references are placed in local call-adjacent storage. Specialized delegated routes also prepare references to zero-initialized temporary words.

This supports an intended transfer interface containing both surfaces and auxiliary state. It does not establish the complete calling convention. In particular, the evidence does not justify replacing these calls with a guessed nine-argument API or claiming that every prepared value is consumed.

After the selected delegate returns, this function returns. There is no local fallback or second transfer attempt.

## Side effects

Established direct effects are:

- Byte reads and writes on formats 0 and 1.
- Calls to stride, element-processing, or transfer dependencies on supported routes.
- Local cursor and temporary-state changes.
- Conditional reads of the global representation values P and Q.

The representation globals are consulted by conditional comparisons rather than captured as one entry snapshot. Their reads occur before transfer dependencies on the selected route. No subsequent redispatch occurs if a dependency changes them.

No explicit heap allocation, deallocation, reference-count operation, or ownership transfer appears in the function. This does **not** establish that dependencies avoid such actions. Their ownership behavior and their treatment of supplied auxiliary pointers remain unknown.

The function has no rollback. Direct writes occur incrementally. If a dependency fails, faults, or exits abnormally, the supplied evidence provides no atomicity guarantee or recovery behavior.

There are no explicit bounds checks on computed addresses, overlap checks, overflow checks, null-base checks, or null checks on auxiliary arguments.

## Dependencies

All dependency descriptions below are limited to their observed role at this call site.

| Opaque ID | Observed role |
|---|---|
| F-082b668da333 | Supplies row-stride values for inline routes. |
| F-4ee07aa0986e | First per-element operation for three-byte source routes. |
| F-a0d7716b0f77 | First per-element operation for the two-byte-to-three-byte route. |
| F-e2db7eabd42d | Second per-element operation for three-byte destinations. |
| F-65573ddce6ca | Second per-element operation for the three-byte-to-two-byte route. |
| F-789728274fbc | First per-element operation for four-byte source routes. |
| F-94989df6c6d2 | Second per-element operation for four-byte destinations. |
| F-d212779ab59f | Second operation for four-byte-to-Q transfers. |
| F-2db8dd7f4e5a | Second operation for four-byte-to-R transfers. |
| F-2ed72e31a28f | Common delegated route, including format 2. |
| F-dc9e9ffc7266 | Delegated Q-to-U route. |
| F-5bbebe570c33 | Delegated Q-to-R route. |
| F-01d5ec758062 | Delegated R-to-U route. |
| F-26417011e1c9 | Delegated R-to-Q route. |

“First operation” and “second operation” do not establish pure reader/writer behavior. Dependencies receive writable references to cursors and temporary state; their actual memory effects require separate contracts.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests.

1. **Rejected starting coordinate.** If source width equals source horizontal offset, the function returns without a dependency call or surface-byte access. The same rejection applies at either vertical boundary or the destination horizontal boundary.

2. **Zero requested extent.** With valid starting coordinates, requested width zero or null-represented requested height causes an immediate return.

3. **Limiting does not account for offsets.** Given source width 10, destination width 12, requested width 8, and source horizontal offset 9, the effective width is 8, not 1. On a direct-byte route with sufficiently backed storage, eight bytes are processed per row. Nominal surface width alone does not make this request safe.

4. **Observable direct-copy order.** Suppose both strides are 4, both vertical offsets are zero, both horizontal offsets are 1, and the effective rectangle is 2 by 2. Formats 0 and 1 transfer relative offsets 1, 2, 5, and 6, in that order, reading each source byte immediately before its corresponding destination write.

5. **Forward overlap.** Suppose a direct-byte transfer starts writing one byte after its source, has effective width 3 and height 1, and the original source bytes begin `A, B, C`. Immediate forward read/write behavior produces three copied `A` bytes, because each write changes the next source byte. Snapshot-copy behavior would not satisfy this contract.

6. **Representation-independent direct route.** Changing either representation selector does not change format 0 or 1 dispatch. The destination auxiliary argument is still supplied to the destination-stride call.

7. **Three-to-two dependency order.** For format 3 or 5, source U, destination P, and P distinct from U, each element invokes F-4ee07aa0986e before F-65573ddce6ca. Exact output bytes remain conditional on those dependencies.

8. **Global-value collision.** If Q equals R, source U with destination R selects F-d212779ab59f as the second inline operation, because the Q comparison wins before the R comparison.

9. **Unsupported format.** A selector outside 0 through 7 produces no transfer dependency call and no direct surface-byte access, although preceding dimension and arithmetic processing may occur.

10. **Delegated same-representation route.** For format 4, Q distinct from U and R, and both representations equal to Q, the function invokes F-2ed72e31a28f once and returns after it returns.

## Uncertainties

The following are **documented_only**:

- The original types and intended domains of pointer-shaped dimensions and offsets.
- The values, stability, and external meanings of P and Q.
- Stride alignment, stride sign, failure behavior, and the complete interface of F-082b668da333.
- Exact component and packed-bit layouts.
- Whether element dependencies advance cursors by the apparent element widths.
- Whether temporary state passed to dependencies is input, output, or shared state. Several reference targets are not explicitly initialized before their first call; unconditional reads from them cannot be assumed valid.
- The complete delegated calling convention and use of call-adjacent storage.
- Dependency effects on globals, auxiliary objects, ownership, and allocation.
- Behavior under concurrent memory modification or overlapping converted transfers.
- Overflow semantics where the representation mixes signed arithmetic, unsigned arithmetic, pointer arithmetic, and integer-to-address conversions.

Widths, selector storage, and address-bearing arithmetic are visibly 32-bit in the supplied representation. Multiplication by 2, 3, or 4, stride multiplication, address addition, and row-end adjustment are not widened or checked. A portable mathematical interpretation for overflowing signed expressions is not established; callers requiring defined addresses must exclude those cases.

## Evidence

The established claims come from static inspection of the supplied function body: entry predicates, minimum-selection operations, dispatch precedence, literal selectors, conditional global reads, dependency calls, direct byte accesses, and cursor adjustments.

The surface-transfer interpretation is inferred intent supported by two-dimensional addressing and repeated per-element operations. Dependency semantics beyond the call site remain unknown.

No dependency implementation was supplied. No runtime tests, smoke runs, memory traces, or output comparisons have been performed.
