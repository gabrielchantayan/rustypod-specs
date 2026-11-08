<!-- Trusted provenance: F-7a15737ebecd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7a15737ebecd

## Purpose

F-7a15737ebecd selects one of several single-bit updates to a shared word-addressed region. For supported update selectors, a zero control value sets the selected bit; any nonzero control value clears it. The other bits of the selected word are preserved from the value read for that invocation.

The function also recognizes one successful no-operation selector. All remaining selector values return a distinct status without accessing the target region in the supplied representation.

The established behavior is bit manipulation and status reporting. The interpretation of the bits as hardware controls, enable flags, disable flags, or another kind of state is not established. In particular, the inverse relationship between the control argument and the written bit does not by itself establish an active-low hardware interface.

## Interface

### Inputs

The function takes two scalar arguments:

| Argument | Established representation | Behavioral role |
|---|---|---|
| Selector | A four-byte value | Chooses a bit update, a successful no-operation, or an unsupported operation. |
| Control | An integer, represented as 32 bits in the supplied target model | Determines whether the chosen bit is set or cleared. Only equality with zero matters. |

The selector is interpreted by its value, not as an address or pointer. Values zero through sixteen have explicit dispatch entries. Values outside that range take the unsupported-operation outcome.

The control argument is not restricted to Boolean values. Zero and nonzero are the only behavioral classes. Negative values, where representable by the argument type, behave like positive nonzero values.

### Return value

The result is a four-byte status value:

| Status | Meaning established by the function |
|---|---|
| `0x00000000` | A selected update completed normally, or the successful no-operation selector was used. |
| `0x00000013` | The selector does not identify a supported update or the successful no-operation. |

No additional status values appear in the supplied representation. These statuses do not establish a broader error-code convention.

Successful return does not report whether the selected bit changed. An update whose bit already has the requested value still follows the update path.

### Shared state and layout

Update operations access a shared region through an externally supplied base reference. This specification calls that region the **target region**, without assigning it an ownership or hardware interpretation.

The accessed elements are 32-bit words. Under the represented four-byte word layout, their locations are:

| Word index | Byte offset from the target base | Used by this function |
|---|---:|---|
| 0 | `0x00` | Yes |
| 1 | `0x04` | No |
| 2 | `0x08` | Yes |
| 3 | `0x0C` | Yes |
| 4 | `0x10` | Yes |
| 5 | `0x14` | Yes |

The highest addressed word occupies bytes at offsets `0x14` through `0x17`. This does not establish that the underlying object is an array of exactly six words, or that the whole span must be readable for every selector. Each update requires its selected word to support the represented read and write.

For normal completion of an update, the target reference must resolve to a location that is valid for the selected 32-bit access. No local validation of the reference, alignment, mapping, or permissions is shown.

## Behavior

### Selector mapping

The complete mapping is:

| Selector | Outcome | Target word | Selected bit | 32-bit bit mask |
|---|---|---:|---:|---|
| `0x00` | Successful no-operation | — | — | — |
| `0x01` | Unsupported | — | — | — |
| `0x02` | Unsupported | — | — | — |
| `0x03` | Update | 0 | 31 | `0x80000000` |
| `0x04` | Unsupported | — | — | — |
| `0x05` | Update | 4 | 31 | `0x80000000` |
| `0x06` | Update | 3 | 15 | `0x00008000` |
| `0x07` | Unsupported | — | — | — |
| `0x08` | Unsupported | — | — | — |
| `0x09` | Unsupported | — | — | — |
| `0x0A` | Unsupported | — | — | — |
| `0x0B` | Update | 3 | 31 | `0x80000000` |
| `0x0C` | Update | 4 | 15 | `0x00008000` |
| `0x0D` | Unsupported | — | — | — |
| `0x0E` | Update | 2 | 31 | `0x80000000` |
| `0x0F` | Update | 2 | 15 | `0x00008000` |
| `0x10` | Update | 5 | 15 | `0x00008000` |
| Any other value | Unsupported | — | — | — |

Bit numbering is zero-based: bit 31 is the most significant bit of a 32-bit word, and bit 15 has mask `0x00008000`. These are numerical bit positions; the representation does not establish byte order in memory.

### Successful no-operation

Selector `0x00` returns zero without reading or writing the target region. The control value is irrelevant on this path.

This is distinct from an update that leaves a word numerically unchanged: the no-operation path contains no target-word read or write.

### Supported updates

Each update has the following observable logical ordering:

1. Read the selected 32-bit target word.
2. Derive a replacement value using the selected bit and whether control equals zero.
3. Write the replacement value to the selected word.
4. Return zero.

For control equal to zero, the selected bit becomes one. For every nonzero control value, the selected bit becomes zero.

For bit 31, setting uses mask `0x80000000`; clearing preserves only the bits admitted by `0x7FFFFFFF`. For bit 15, setting uses mask `0x00008000`; clearing preserves only the bits admitted by `0xFFFF7FFF`.

These are 32-bit operations. Clearing bit 15 does not clear the upper half of the word. Setting bit 31 does not perform a signed comparison or signal signed overflow.

All bits other than the selected bit are copied from the value read. No other target word is explicitly accessed by that update path.

An update is not skipped when its desired bit value already matches the read value. The supplied representation still performs the read and subsequent write.

### Unsupported selectors

An unsupported selector returns `0x00000013`. It does not access the target region in the supplied representation, and the control value has no effect on this outcome.

Explicitly listed unsupported values and values outside the listed selector range have the same result. There is no fallback update, selector wrapping, range clamping, or interpretation of the selector as a word index.

### Arithmetic and concurrency boundaries

There is no numeric scaling, accumulation, or arithmetic on the control argument. Its magnitude is discarded by the zero/nonzero decision.

The update is represented as a read followed by a write, not as an atomic bit operation. If another actor modifies the same word between those accesses, this function can overwrite that modification with bits copied from its earlier read. Preservation of unrelated bits therefore applies to the read snapshot, not necessarily to concurrently changing live state.

No retry, synchronization, or conflict detection is shown.

## Side effects

For a supported update, the explicit memory side effect is one 32-bit write to the selected target word, following its read. The write occurs even if its value equals the value read.

For selector zero and unsupported selectors, there are no explicit target-region accesses or state changes.

The function does not visibly allocate, free, acquire, release, or transfer ownership of memory. It uses the target region as externally provided state. Nothing establishes that the function owns that region or may change its lifetime.

No logging, callback invocation, notification, or explicit modification of the stored target reference appears.

If the region is device-backed, reading or writing it may have effects beyond ordinary memory mutation. The supplied representation does not establish register semantics such as write-one-to-clear behavior, read side effects, or reserved-bit requirements. Consequently, a masked replacement value must not be equated with a guaranteed physical-device state transition without an additional register contract.

## Dependencies

No callee dependency is visible. There is therefore no external function behavior to describe or opaque callee ID to assign.

The function depends on:

- An externally available target-region reference for update paths.
- Valid 32-bit read and write access at the selected word.
- The surrounding execution environment’s memory-access semantics.
- External coordination if callers require protection against concurrent word updates.

The represented target-word read precedes its write. Exact machine-level loading of the shared base reference, including whether it is loaded once or more than once, is not established here. Stability of that reference during an update is an environmental precondition for interpreting the operation as an update to one fixed region.

There is no visible recovery dependency for invalid memory access.

## Acceptance examples

These are conditional acceptance cases derived from static evidence, not executed tests. Update examples assume valid ordinary writable memory, a stable target reference, and no concurrent mutation.

### Successful no-operation

Given selector `0x00`, the function returns `0x00000000` for control zero, positive nonzero control, and negative nonzero control. No target-region access occurs.

### Set the high bit of word 0

Given selector `0x03`, control zero, and word 0 initially `0x12345678`, word 0 is written as `0x92345678`. The function returns zero. Other words have no explicit access.

### Clear the high bit of word 0

Given selector `0x03`, nonzero control, and word 0 initially `0xFFFFFFFF`, word 0 is written as `0x7FFFFFFF`. Control `1` and control `-1` select the same transformation.

### Preserve the upper bits while clearing bit 15

Given selector `0x06`, nonzero control, and word 3 initially `0xFFFFFFFF`, word 3 is written as `0xFFFF7FFF`. Its upper sixteen bits remain set.

### Independently address two bits in one word

Given word 4 initially zero, selector `0x05` with zero control writes `0x80000000`.

If selector `0x0C` with zero control is then invoked without intervening mutation, word 4 becomes `0x80008000`. A subsequent selector `0x05` invocation with nonzero control writes `0x00008000`, preserving bit 15.

### Exercise the highest selected word

Given selector `0x10`, control zero, and word 5 initially `0x00000001`, the function writes `0x00008001` at byte offset `0x14` and returns zero.

### Numerically unchanged update

Given selector `0x0F`, control zero, and word 2 initially `0x00008000`, the function reads that word and writes `0x00008000` back before returning zero. This must not be treated as the no-operation selector’s access behavior.

### Unsupported input

Given selector `0x04` or selector `0x11`, the function returns `0x00000013` without accessing the target region, regardless of control.

## Uncertainties

The following matters are **documented_only**:

- **Target meaning:** The region’s identity and the semantic names of its words and bits are unknown.
- **Memory kind:** Ordinary memory versus memory-mapped device state is not established.
- **Access implementation:** Exact machine instructions, volatile treatment, barriers, and atomicity guarantees are not supplied.
- **Base-reference stability:** Behavior under concurrent replacement of the target reference is not established.
- **Fault handling:** Invalid, inaccessible, or misaligned targets have no visible local recovery path. A particular exception or process outcome cannot be promised.
- **Ownership and lifetime:** Initialization, allocation, release, and lifetime management of the target region are outside the supplied behavior.
- **Platform details:** Byte order and calling-convention details beyond the represented scalar widths are not established.
- **Status interpretation:** Zero and `0x13` are established return values, but their names or meanings in a wider API are unknown.

An inferred intent is that selectors identify individual controls and the second argument requests an inverse-polarity bit state. That inference is not required to reproduce the observable behavior and is not a substitute for a device or subsystem specification.

## Evidence

The supplied static representation establishes the selector dispatch, zero/nonzero control decision, target-word indices, bit masks, read-before-write structure, and two return values.

It also shows no calls, target validation, synchronization, allocation, release, or explicit error recovery. These are observations about the supplied function representation, not guarantees about the wider environment.

No runtime tests have been performed. The acceptance examples are derived expectations, not observed execution results.
