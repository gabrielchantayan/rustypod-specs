<!-- Trusted provenance: F-62b2fbd9cf6a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-62b2fbd9cf6a

## Purpose

F-62b2fbd9cf6a constructs a result in caller-supplied destination storage. A 32-bit mode selects between two behaviors:

- **Mode other than `0x00006D19`:** select a resource token using entry flags, or retain an initially captured default token when the index is invalid; resolve that token through dependencies and construct a resource result.
- **Mode `0x00006D19`:** use an entry’s stored value directly, except when an exact entry discriminator and a zero placeholder-comparison result select resource construction. An invalid index selects a global fallback object directly.

The established contract includes index classification, ordered global reads, dependency calls, destination initialization, and the direct-path identity check. The interpretation of the result as string-like and of resource resolution as localization-like is inferred intent, not an established type contract.

No runtime tests have been performed.

## Interface

### Arguments and result

| Argument | Established role |
|---|---|
| Destination | Address of writable result storage. The visible construction paths access its first two 32-bit words. |
| Container | Address of storage with an entry-related region beginning at byte offset `156` and a 32-bit bound at byte offset `160`. |
| Mode | A 32-bit value compared for equality with `0x00006D19`. All other bit patterns select the non-special behavior. |
| Index | A 32-bit quantity represented in pointer-shaped form in the supplied evidence, but classified through signed comparisons and not dereferenced. |

On ordinary completion, the function returns the destination address. The supplied representation contains recovered register and stack state around shared construction continuations; the limits this imposes on reconstructing their standalone interfaces are retained under **Uncertainties**.

### Relevant storage layouts

Offsets are in bytes. Each listed field is 32 bits wide.

| Object | Offset | Visible use |
|---|---:|---|
| Destination | `0` | Written with a resource-result or direct-result header. |
| Destination | `4` | Cleared before assignment, except on direct-path exact identity. |
| Container | `156` | Address passed to F-017a009221aa. |
| Container | `160` | Read as the signed index bound when the index is nonnegative. |
| Resolved entry | `4` | Value supplied to placeholder comparison or direct assignment. |
| Resolved entry | `8` | Flags in non-special mode; exact discriminator in special mode. |
| Fallback object | `4` | Value supplied to direct assignment when source and destination differ. |

The fallback object’s offset-`8` field is not inspected on the invalid-index special-mode path.

No explicit index argument is visible at the F-017a009221aa call. Ordinary array indexing, an element size, and index-dependent address arithmetic are therefore not established.

### Preconditions and error interface

Every address actually accessed must support its corresponding read or write. This includes global sources whose values are read and subsequently discarded.

For defined execution:

- A nonnegative index requires a readable bound at container offset `160`.
- An in-range index requires the resolver to return an object supporting the selected field accesses.
- Destination storage must support the header write and, where applicable, the second-word clear and assignment dependency.
- Invalid-index special-mode construction requires a usable fallback pointer and any fallback field subsequently read.
- All reached dependencies must satisfy their own, otherwise unspecified, execution requirements.

There is no visible null check, explicit error result, allocation-failure branch, or rollback. Writable destination storage alone does not establish that replacing an already initialized object is safe.

## Behavior

### Initial read and index classification

The function reads a global default resource token on entry, before mode-specific processing. This read occurs even on paths that later replace or never use the captured token.

Index classification uses signed 32-bit comparisons:

- A negative index is invalid without reading the container bound.
- A nonnegative index causes a read of the bound at container offset `160`.
- Such an index is valid only when strictly less than that signed bound.

Index zero is therefore valid when the bound is positive. Equality with the bound is invalid. A zero or negative bound admits no valid nonnegative index; a bound with bit `31` set is negative for this comparison.

No visible index increment, element-size multiplication, or indexed address calculation accompanies this classification.

### Non-special mode: invalid index

For a mode other than `0x00006D19`, an invalid index bypasses entry resolution and flag inspection.

F-9d0ecc5f6adf is called without explicit arguments. Its result and the default token captured at entry are supplied to F-c0b020cd3882. That dependency’s return value is then used for resource-result construction.

The token is captured before either dependency runs. A subsequent change to its global source does not replace the already captured argument.

### Non-special mode: in-range index

F-017a009221aa receives the address of the container region at offset `156`. After it returns, the resolved entry’s offset-`8` word is read as flags.

Only masks `0x01`, `0x02`, `0x04`, `0x10`, and `0x20` affect the visible selection. Other flag bits do not change the described branches.

The following labels name separate global resource-token sources. They do not assign semantic meanings or guarantee distinct runtime values.

| Bit `0x01` | Remaining conditions | Final token |
|---|---|---|
| Clear | `0x04` set | A |
| Clear | `0x04` clear; `0x02` set | B |
| Clear | `0x04` and `0x02` clear; `0x20` set | C |
| Clear | `0x04`, `0x02`, `0x20`, and `0x10` clear | D |
| Clear | `0x04`, `0x02`, and `0x20` clear; `0x10` set | E |
| Set | `0x04` set | D minus `3` |
| Set | `0x04` clear; `0x02` set | F |
| Set | `0x04` and `0x02` clear; `0x10` set | G |
| Set | `0x04`, `0x02`, and `0x10` clear; `0x20` clear | H |
| Set | `0x04`, `0x02`, and `0x10` clear; `0x20` set | J |

#### Ordered reads, including discarded values

Final token selection does not describe all observable global accesses. Selection includes intermediate reads even when a later source replaces their values.

When `0x01` is clear:

- A is read first, whether or not `0x04` is set.
- If `0x04` is clear, B is read after A.
- If `0x02` is also clear, C is read after B.
- If `0x20` is also clear, D is read after C.
- If `0x10` is then set, E is read after D.

Thus selecting E requires the ordered reads A, B, C, D, E. Selecting D requires A, B, C, D. These are not alternatives in which only the final source is accessed.

When `0x01` is set and `0x04` is clear:

- F is read first.
- If `0x02` is clear, G is read after F.
- If `0x10` is also clear, H is read after G.
- If `0x20` is then set, J is read after H.

Thus selecting J requires F, G, H, J; selecting H requires F, G, H.

When both `0x01` and `0x04` are set, D is read for the subtractive case. The A–E and F–J selection chains are not traversed.

All these selection reads occur after entry resolution and the flags read, and before F-9d0ecc5f6adf. The default-token read at function entry remains an additional earlier read on every path.

“D minus `3`” is a word-sized subtraction from the loaded D value, not a separate global resource source. The 32-bit result is the subtraction’s low 32 bits; no saturation or validity check is visible. Whether that resulting token is meaningful to the resolver remains unspecified.

After selection, F-9d0ecc5f6adf is called. Its result and the selected token are passed to F-c0b020cd3882. No global token reload is visible between these calls.

### Resource-result construction

The resource-result path has this observable ordering:

1. Write the resource-result header to destination offset `0`.
2. Clear destination offset `4` to zero.
3. Call F-151ab4bfe553 with the destination and the value returned by F-c0b020cd3882.
4. Return the destination.

The header comes from a fixed source in the supplied body; its numeric value is not supplied. No source/destination identity check or cleanup of previous destination contents precedes this initialization.

### Special mode: invalid index

For mode `0x00006D19`, an invalid index selects the object referenced by a global fallback pointer.

This path does not call the entry resolver, inspect a type discriminator, or compare placeholder text. It proceeds directly to direct-result construction.

### Special mode: in-range index

F-017a009221aa receives the address of the container region at offset `156`. The returned entry’s offset-`8` word is treated as an exact discriminator:

| Exact value | Placeholder literal | Resource token selected on comparison result zero |
|---:|---|---|
| `2` | `<<<cell_phone>>>` | Cell-placeholder token |
| `1` | `<<<home_phone>>>` | Home-placeholder token |
| `8` | `<<<work_phone>>>` | Work-placeholder token |

These are equality tests, not bit tests. Discriminator `3`, for example, matches neither `1` nor `2`.

For a recognized discriminator, the corresponding resource token is read before F-634a31ef9eb5 is called. That dependency receives the entry’s offset-`4` value and the corresponding literal.

- **Zero result:** call F-9d0ecc5f6adf, then F-c0b020cd3882 with its result and the previously captured placeholder token; perform resource-result construction.
- **Nonzero result:** perform direct-result construction from the entry.

The placeholder token read occurs even when the comparison returns nonzero and the token is discarded.

For every unrecognized discriminator, no placeholder comparison occurs and the entry is used directly.

The visible zero test establishes the branch condition. Interpreting zero as textual equality is inferred intent; the full comparison contract is unavailable.

### Direct-result construction

The selected source object is either the resolved entry or the fallback object.

First, the function writes the direct-result header to destination offset `0`. This header has a different source from the resource-result header; their runtime bit patterns are not established to differ.

The function then compares the destination and source addresses:

- **Exact identity:** no explicit offset-`4` clear, source-value read, or F-151ab4bfe553 call occurs. The header write has already occurred.
- **Different addresses:** clear destination offset `4`, then read source offset `4`, then call F-151ab4bfe553 with the destination and that value.

The destination is returned on ordinary completion.

After a nonzero placeholder comparison, direct construction reads the source value again. It does not necessarily reuse the value supplied to F-634a31ef9eb5. Changes made by that dependency can therefore affect the later assignment.

## Side effects

Established direct effects include:

- The initial default-token read, including paths where its value is unused.
- Conditional container-bound and entry-field reads.
- Every reached intermediate global token read in its specified order, including discarded reads.
- Conditional placeholder-token and fallback-pointer reads.
- Destination header writes and conditional second-word clears.
- Dependency calls in the branch-specific order described above.

No explicit allocation, retention, release, or deallocation is visible in this function. Dependencies may perform any of these operations; their ownership rules are not established.

Previous destination contents are overwritten without a visible cleanup call. The function does not establish safe replacement of an initialized result or recovery after a dependency exits abnormally.

No direct write to a separate source object is visible. Aliasing can change this:

- Direct-path exact identity suppresses the clear and assignment, but not the header write.
- Distinct, partially overlapping objects receive no overlap protection.
- Because the destination writes precede the direct source-value read, overlap can change the value subsequently read.

Dependencies are not assumed to preserve globals or source objects. The specified read order does not imply an atomic snapshot or synchronization. Captured values remain inputs to later visible operations; unread globals and reread fields are not assumed unchanged.

## Dependencies

| Opaque ID | Established relationship |
|---|---|
| F-017a009221aa | Receives the container-region address at offset `156`; returns an entry-like address. Selection mechanism, failure behavior, and ownership are unknown. |
| F-9d0ecc5f6adf | Called without explicit arguments; its result is passed to F-c0b020cd3882. |
| F-c0b020cd3882 | Receives the preceding result and a captured resource token; returns the value used for resource assignment. |
| F-634a31ef9eb5 | Receives an entry value and one placeholder literal. Zero selects resource construction; nonzero selects direct construction. |
| F-151ab4bfe553 | Receives initialized destination storage and a value. Copying, sharing, allocation, retention, release, and failure behavior are unspecified. |
| F-7afe554bcf62 | Identifies the shared resource-construction continuation. No separate call boundary is established here. |
| F-b9acea4b06ad | Identifies the shared direct-construction continuation. No separate call boundary is established here. |

No dependency is assumed to preserve globals, preserve source storage, transfer ownership, or release supplied memory.

## Acceptance examples

These are conditional criteria derived from static evidence, not executed results.

1. **Zero index and exclusive bound.** With index `0` and bound `1`, entry resolution is reached. With index `1` and bound `1`, it is not. The latter uses the default resource path in non-special mode and the fallback object in special mode.

2. **Negative index avoids the bound read.** With index `-1`, classification does not read container offset `160`. The initial default-token read still occurs.

3. **Nonpositive bound admits no index.** With index `0` and a bound of zero or signed `-1`, entry resolution is skipped.

4. **High-precedence flag avoids later selection reads.** In non-special mode, flags `0x26` select A. A is read after the entry flags; B, C, D, and E are not read by selection. The initial default-token read occurs earlier.

5. **Discarded reads remain required.** With non-special mode and flags `0x00`, the selection reads are A, B, C, D, in that order, and D supplies the token. Reading only D would not match the established access sequence.

6. **Late clear-bit-chain override.** With flags `0x10`, the selection reads A, B, C, D, E, in order, and uses E.

7. **Set-bit chain has a different precedence.** With flags `0x11`, selection reads F then G and uses G; H and J are not read. With flags `0x21`, selection reads F, G, H, J and uses J.

8. **Subtractive case bypasses both chains.** With flags `0x05`, selection reads D and uses its 32-bit value minus `3`. If D contains `2`, the resulting word is `0xFFFFFFFF`; acceptance by F-c0b020cd3882 is not established.

9. **Exact discriminator, not combined flags.** In special mode, discriminator `3` causes no placeholder comparison and uses direct construction.

10. **Placeholder token precedes comparison.** With discriminator `2`, the cell-placeholder token is captured before comparison with `<<<cell_phone>>>`. If comparison returns zero, that captured token is used even if the comparison changed its global source.

11. **Nonzero comparison permits a later value to matter.** With discriminator `1`, a nonzero comparison followed by distinct source and destination addresses causes a later read of source offset `4`. If the comparison changed that field, the later value is supplied to assignment, subject to any intervening alias effects.

12. **Direct identity still writes the header.** If the selected direct source equals the destination, the direct-result header is written, but the explicit second-word clear and assignment call are omitted.

13. **Fallback type is irrelevant.** Invalid special-mode input selects fallback direct construction without inspecting fallback offset `8`, regardless of the value stored there.

## Uncertainties

The following remain **documented_only**:

- Concrete object types, complete sizes, alignment requirements, and byte order.
- Numeric header and resource-token values and their semantic meanings.
- Whether the two header sources contain different runtime values.
- How F-017a009221aa selects an entry without a visible explicit index argument.
- Dependency error behavior, nonlocal exits, allocation behavior, and ownership obligations.
- Semantic validity of the subtractive token, including underflow results.
- Concurrency guarantees, memory-access atomicity, and changes between ordered reads.
- Supported aliasing beyond the explicit direct-path identity check.
- The complete standalone interfaces of the shared continuations, given recovered register and stack artifacts.

The string-like result, localization-like resource resolution, and text-equality interpretation of a zero comparison result are inferred intent. They are not prerequisites for the established branch and access behavior.

## Evidence

This contract is based on the supplied static function body and opaque dependency mapping.

Established evidence includes 32-bit field accesses, signed index comparisons, the exact mode value, flag precedence, ordered intermediate global reads, word-sized subtraction, exact placeholder discriminators and literals, captured-token timing, dependency order, destination initialization, and the direct-path identity check.

The final-selection table and the ordered-read requirements describe different aspects of the same behavior: a token may be selected only after earlier global values have been read and discarded. Both are part of this contract.

Notices about removed unreachable regions and recovered execution state do not establish additional reachable behavior. No missing operation or dependency guarantee is inferred from those notices.

No runtime tests have been performed. The acceptance examples are static conditional requirements, not observed executions.
