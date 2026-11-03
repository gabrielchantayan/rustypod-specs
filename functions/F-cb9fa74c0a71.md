<!-- Trusted provenance: F-cb9fa74c0a71; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cb9fa74c0a71

## Purpose

F-cb9fa74c0a71 selects a 32-bit result according to a single 32-bit selector. Depending on that selector, the result comes from one of twelve distinct symbolic storage locations, is the constant 1024, or is the all-ones 32-bit value.

**Established static behavior:** the function is a lookup-style accessor with one fixed-value case and a common fallback. Several selectors deliberately share a storage source.

**Inferred intent:** the selector may identify a property or capability, with the all-ones value indicating an unsupported selection. Neither that interpretation nor the meaning of any stored value is established. The contract therefore uses numeric selectors and neutral storage labels rather than assigning domain semantics.

This is a source-facing contract. It describes behavior supported by the supplied representation, not verified execution behavior.

## Interface

### Input

One argument: a **32-bit selector**.

The supplied representation does not establish whether the public interface treats this word as signed or unsigned. Selection can nevertheless be specified unambiguously by its 32-bit pattern:

- Patterns representing integers 0 through 17 receive the treatment listed below.
- Every other pattern takes the common fallback.
- Within 0 through 17, selectors 9, 11, and 13 also take that fallback.

If a caller interprets the argument as signed, negative values still do not match any listed selector. For example, the all-ones input pattern takes the fallback. Conversion of a wider caller-side value into the 32-bit argument is outside the established interface.

### Output

One **32-bit result**:

- A value supplied by selected symbolic storage;
- The exact bit pattern `0x00000400`; or
- The exact bit pattern `0xFFFFFFFF`.

The all-ones result is 4,294,967,295 under an unsigned interpretation and −1 under a two’s-complement signed interpretation. Those are interpretations of the same returned bits, not separate behaviors.

There is no separate status result. In particular, the result alone need not distinguish a fallback from a selected storage value that also produces all ones.

### Storage labels

This contract assigns neutral labels **S1 through S12** to the twelve distinct symbolic storage references in the supplied representation. These labels identify sources, not values, addresses, ownership classes, or externally visible names.

Two different labels may contain equal values. Conversely, the shared label for two selectors establishes a shared source even when no particular value is known.

### Preconditions

For a storage-backed selection to complete normally, the selected storage access must be valid in the execution environment.

The representation establishes no required initialization sequence, lock, privilege level, or caller-owned buffer. It also provides no explicit validation or recovery if selected storage is inaccessible. Such environmental requirements cannot be invented from the selector logic.

## Behavior

### Selector mapping

The complete mapping is:

| Selector, decimal | Result source |
|---:|---|
| 0 | S1 |
| 1 | S1 |
| 2 | S2 |
| 3 | Constant `0x00000400` |
| 4 | S3 |
| 5 | S4 |
| 6 | S4 |
| 7 | S5 |
| 8 | S6 |
| 9 | Constant `0xFFFFFFFF` |
| 10 | S7 |
| 11 | Constant `0xFFFFFFFF` |
| 12 | S8 |
| 13 | Constant `0xFFFFFFFF` |
| 14 | S9 |
| 15 | S10 |
| 16 | S11 |
| 17 | S12 |
| Any other 32-bit pattern | Constant `0xFFFFFFFF` |

For a storage-backed case, the function returns the 32-bit result contributed by the corresponding symbolic storage reference. The supplied material does not include storage declarations sufficient to establish the physical read width or any conversion from the storage object’s native type.

### Shared selections

Selectors 0 and 1 refer to the same source, S1. Selectors 5 and 6 refer to the same source, S4.

This guarantees source equivalence, not equality between arbitrarily timed calls. If another actor can change the relevant storage, successive calls using either pair may return different values.

No other pair is shown sharing a symbolic source. Equal contents in distinct sources do not establish aliasing.

### Fixed selection

Selector 3 returns exactly 1024. Its result does not depend on any of S1 through S12 in the supplied representation.

The constant is not computed by scaling, shifting, or reading a stored configuration value. There is no selector arithmetic whose overflow must be modeled.

### Fallback

Selectors 9, 11, and 13 reach the same fallback return as unlisted selectors. They do not select storage and do not produce distinct error values.

The fallback is exactly `0xFFFFFFFF`. There is no established wrapping, clamping, masking, retry, or secondary lookup for an unrecognized selector.

### Read and return ordering

At the source-facing level:

1. The selector determines the applicable result source.
2. A storage-backed case obtains its result from that selected source.
3. The function returns the selected result.

The representation contains no explicit inspection of other labeled sources before returning a storage-backed result. Constant and fallback cases contain no explicit access to any labeled source.

These observations do not establish machine-level access counts, access widths, instruction ordering, volatility, or speculative behavior. There is also no sequence of multiple explicit storage reads within a selected case for which a relative ordering must be preserved.

## Side effects

**Established static evidence:** the body contains no explicit writes to the labeled storage, no other explicit state updates, and no function calls.

Accordingly, the represented logic:

- Does not update a selected value;
- Does not record the last selector;
- Does not maintain a call count or cache;
- Does not allocate, release, copy, or transfer an explicitly identified memory object;
- Does not explicitly set an auxiliary error status.

These are statements about the supplied body, not guarantees about unspecified hardware effects or the broader execution environment. If a symbolic storage reference represents a special resource, its read-side effects are not established here.

The argument is used as a selector, not dereferenced as a pointer. The return is an untyped 32-bit word. If an external consumer interprets that word as a pointer or handle, this function supplies no established ownership, lifetime, validity, or release rule for that interpretation.

No callee-related preservation or ownership assumptions are needed: no dependency calls are present.

## Dependencies

There are **no explicit function dependencies** in the supplied body, so there are no dependency function IDs to enumerate.

The data dependencies are the twelve symbolic storage sources:

| Storage source | Selecting input or inputs |
|---|---|
| S1 | 0, 1 |
| S2 | 2 |
| S3 | 4 |
| S4 | 5, 6 |
| S5 | 7 |
| S6 | 8 |
| S7 | 10 |
| S8 | 12 |
| S9 | 14 |
| S10 | 15 |
| S11 | 16 |
| S12 | 17 |

Their producers, initialization, mutability, synchronization requirements, and external meanings are unknown. Nothing establishes that the twelve sources form an array, occupy contiguous storage, or are initialized together. Numeric proximity among selectors does not imply a physical layout.

## Acceptance examples

The following are **conditional acceptance examples, not executed tests**.

1. **Shared first source.** If S1 supplies `0x12345678` and remains unchanged across both calls, selectors 0 and 1 must each return `0x12345678`.

2. **Shared second pair.** If S4 supplies `0x0000002A` and remains unchanged, selectors 5 and 6 must each return 42. A change between calls may legitimately change the result.

3. **Independent source selection.** If S2 supplies 100 and S3 supplies 200, selector 2 must return 100 and selector 4 must return 200. Selector adjacency must not redirect either selection.

4. **Fixed-value case.** Selector 3 must return `0x00000400` regardless of the contents of the twelve labeled sources.

5. **Explicit fallback selectors.** Each of selectors 9, 11, and 13 must return `0xFFFFFFFF`; none selects a labeled storage source.

6. **Upper boundary.** Selector 17 must return the result supplied by S12. Selector 18 must return `0xFFFFFFFF`.

7. **Large unmatched patterns.** Inputs `0x80000000` and `0xFFFFFFFF` must both return `0xFFFFFFFF`. Their possible signed interpretations do not create additional matching cases.

8. **Sentinel collision.** If S7 supplies `0xFFFFFFFF`, selector 10 must return that pattern. A consumer cannot infer from the result alone whether selector 10 selected such a value or an unsupported selector took the fallback.

9. **Zero stored value.** If S9 supplies zero, selector 14 must return zero. The represented logic does not substitute the fallback merely because a stored result is zero.

10. **Later selectors.** If S10, S11, and S12 supply distinct results, selectors 15, 16, and 17 must return their respective results without exchanging or combining them.

## Uncertainties

The following remain **documented_only**:

- **Backing storage widths and types.** The function interface is represented as 32-bit input and output, but declarations of the storage objects are absent. Exact load widths and conversion rules are not independently established.
- **Signedness and ABI.** Register assignment, calling convention, and caller-facing signedness are not supplied.
- **Data layout.** Physical addresses, alignment, contiguity, sections, and binary representation of backing objects are not part of this contract.
- **Initialization and meaning.** No initial values, units, valid ranges, or semantic names are established.
- **Concurrency.** Atomicity, synchronization, visibility, and stability across calls are unknown. A shared source does not imply an immutable result.
- **Read-side effects.** The storage may be ordinary memory or something requiring additional access semantics; the supplied body does not resolve this.
- **Fault behavior.** No explicit fault handling is shown. Behavior after an invalid or faulting storage access is not established.
- **Ownership interpretation.** A returned word might be interpreted externally as data, a pointer, or a handle. No such interpretation or associated lifetime rule is established.
- **Fallback meaning.** “Unsupported selector” is an inferred interpretation, not a proven domain-level error definition.

## Evidence

The supplied static representation establishes a single selector-based dispatch, twelve distinct symbolic storage references, two selector pairs sharing sources, one fixed 1024 result, and a common all-ones fallback.

The complete selector table above follows that representation. The absence of explicit calls, writes, allocation, and release likewise describes only the supplied body.

No runtime tests, binary-level inspection, storage inspection, or concurrency experiments have been performed. Acceptance examples specify conditional obligations; they are not execution evidence.
