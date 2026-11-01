<!-- Trusted provenance: F-b5bbbba23d63; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b5bbbba23d63

## Purpose

F-b5bbbba23d63 modifies an indexed unsigned storage element and conditionally decreases a separate signed bookkeeping value. A decision supplied by F-88658862e573 controls the decrease, but does not control whether the indexed assignment is reached after normal dependency completion.

When the integer operations have ordinary, valid single-bit-mask semantics, the indexed assignment clears one selected numerical bit. It neither toggles that bit nor avoids the assignment when the bit is already clear.

**Inferred intent:** removal of a position from a bit-based collection, with an associated accounting adjustment. The available evidence does not establish membership semantics, a population-count invariant, or protection against repeated decreases.

This is a static behavioral contract. No runtime tests have been performed.

## Interface

### Inputs and result

| Item | Contract |
|---|---|
| Locator | Signed integer used to derive storage addresses and supplied unchanged to F-88658862e573. |
| Index | Corresponding-rank unsigned integer selecting a group and a position within that group. |
| Result | No returned value or explicit completion status. Observable behavior consists of the dependency invocation and storage effects. |

Exact integer widths are **documented_only**. The grouping size and storage spacing do not independently establish a 32-bit platform.

The locator is not established to be a pointer, validated handle, allocation identifier, or ownership token. Defined execution requires the environment to support the integer-derived addresses actually used.

### Storage roles

| Role | Starting location | Interpretation |
|---|---|---|
| Bookkeeping | Four numerical address units beyond the locator | Signed integer |
| Stored base | Eight numerical address units beyond the locator | Signed integer used to derive indexed addresses |
| Indexed element | Stored base, advanced four numerical address units for each preceding group | Unsigned integer |

The signed and unsigned storage interpretations correspond to the input integer types.

With byte addressing and four-byte integer objects, bookkeeping occupies relative bytes 4–7, the stored base occupies relative bytes 8–11, and indexed group starts are four bytes apart. These byte ranges are conditional platform consequences, not independently established layout facts.

Starting-location spacing does not prove object size, alignment, or disjointness. There is no dedicated access to the locator’s starting location, although indexed addressing can select that location.

### Preconditions

Defined completion requires all reached operations to be valid, including:

- Representable signed locator-relative additions.
- Supported integer-to-address conversions.
- Accessible, live storage with appropriate alignment and permitted object interpretations.
- Sufficient storage at the selected indexed addresses.
- A representable bookkeeping decrease when required.
- Valid shift, complement, and integer-conversion behavior for the selected position.
- Independently valid overlap or interference, if either is present.

No explicit validation establishes these conditions. A normal dependency return is not evidence that storage remains valid.

## Behavior

### Selection and decision

The index selects consecutive groups of 32 positions. Indices zero through 31 select the first group; indices 32 through 63 select the second. Within each group, positions are numbered zero through 31.

F-88658862e573 is explicitly invoked once when execution reaches the call. It receives the original locator, the unsigned group number, and the unsigned position.

Only the distinction between a zero and nonzero signed result matters:

| Dependency outcome | Required continuation, subject to defined execution |
|---|---|
| Normal return with zero | No dedicated bookkeeping decrease; indexed assignment follows. |
| Normal return with a positive or negative nonzero value | Bookkeeping decreases by one before indexed addressing and assignment. |
| No normal return | No subsequent behavior of this function is established. |

The magnitude or sign of a nonzero result does not change the amount of the decrease.

### Bookkeeping effect

The nonzero-result path decreases the signed bookkeeping value available after dependency completion by exactly one. There is no positivity check, lower-bound check, or consistency check against the selected bit.

A value of one becomes zero. Zero becomes negative one. A negative value becomes one less, provided the result is representable.

The function does not preserve an invocation-entry bookkeeping snapshot for this purpose. If the dependency changes bookkeeping before returning nonzero, the decrease applies to the changed value.

At the minimum representable signed value, no defined result is guaranteed under execution rules that make signed overflow invalid. Neither wrapping nor saturation is established.

A zero result suppresses only the dedicated decrease. It does not guarantee unchanged bookkeeping: the dependency can modify it, and a valid overlapping indexed assignment can modify it later.

### Indexed assignment

After either category of normal dependency return, the function reaches an unsigned indexed assignment.

Under stable addressing and valid ordinary mask semantics, the assigned value equals the indexed source value with the selected numerical bit cleared. Position zero is the least significant numerical bit. Other numerical bits are preserved.

The assignment remains part of the behavior when the selected bit is already clear. An unchanged numerical element value therefore does not imply that the assignment was skipped. It also does not suppress a decrease required by a nonzero dependency result.

The mask is formed using signed-integer operations before combination with the unsigned source value. An unsigned destination does not by itself make those earlier operations valid.

For conventional 32-bit signed integers, positions zero through 30 have representable positive single-bit values. Position 31 requires separate justification under the applicable signed-shift rules; two’s-complement representation alone is insufficient. With narrower integers, some selectable positions can exceed the permitted shift count or result range. With wider integers, selection still remains restricted to positions zero through 31; under ordinary mask semantics, higher element bits are preserved.

Signed representation, complement rules, and conversion to unsigned can affect the numerical result. Where those rules are unresolved, the evidence establishes the assignment’s structure, but not an unconditional numerical bit-clear result.

### Observable ordering

The dependency completes before the function’s direct bookkeeping and indexed-storage operations.

On the nonzero-result path, the bookkeeping decrease precedes indexed addressing. The later operations therefore use subsequently available state, not a protected invocation-entry snapshot. Valid overlap can make the decrease affect either the indexed source value or addressing information.

Source and destination addressing each refer to the stored-base location. The evidence does not establish a shared base snapshot or a relative evaluation order between those references.

With stable ordinary storage, both address formations select the same element. If the applicable execution model independently permits different observations without invalid interference, source and destination can differ. In that case, the destination receives the masked source value; preservation of the destination’s own previous unselected bits is not guaranteed.

The indexed source value must be available before the assignment takes effect. These semantic constraints do not establish physical load counts, store counts, instruction ordering, or device-visible transaction ordering.

### Address arithmetic

Successive indexed groups begin four numerical address units apart. Group selection occurs before scaling.

The group displacement is representable in the unsigned index type: dividing the index into groups of 32 before multiplying by four keeps the displacement below that type’s maximum. This arithmetic fact provides no storage-capacity guarantee.

The stored base is interpreted as signed, while the displacement has the corresponding-rank unsigned type. Their combination consequently uses unsigned arithmetic under the represented conversion rules. Address arithmetic can wrap numerically. Negative stored-base values are not explicitly rejected.

The locator-relative additions remain signed. The four-unit addition is required only for the dedicated bookkeeping operation. The eight-unit addition is required for indexed addressing after either normal dependency result.

Thus, a zero result avoids the dedicated bookkeeping access, but does not avoid the stored-base access or its locator-relative calculation. No signed wrapping policy is established.

### Overlap

No disjointness check is present. Where overlap and the associated storage interpretations are independently valid:

- The bookkeeping decrease can change the indexed source before it is read.
- The indexed assignment can change bookkeeping even on the zero-result path.
- The indexed destination can overwrite the stored base.
- An indexed element can begin at the locator itself.

No further use of the stored base is established after the indexed assignment. Overwriting that location does not imply another redirected access during the same invocation.

## Side effects

The direct mutation footprint is a conditional signed bookkeeping decrease followed by an unsigned indexed assignment, subject to defined execution.

Direct read roles are bookkeeping when decreased, the stored base for source and destination addressing, and the indexed source value. These roles need not correspond to distinct objects or distinct physical memory transactions.

No direct allocation, release, ownership transfer, locking, or atomic operation is established. Access to storage does not establish ownership.

F-88658862e573 must not be assumed to preserve globals, bookkeeping, base contents, indexed contents, object lifetimes, or ownership relationships. Conversely, receiving the locator does not establish that the dependency owns or releases any resource.

There is no explicit error status, diagnostic path, retry, rollback, or compensation. If a later operation is invalid, this contract does not turn earlier effects into a specified partial-success result.

No atomic read-modify-write or synchronization guarantee is provided. The function does not authorize data races. Reentrancy and suitability for device memory remain **documented_only**.

## Dependencies

### F-88658862e573

| Aspect | Established call-site contract |
|---|---|
| Invocation count | One explicit invocation when execution reaches the call |
| Inputs | Original signed locator, unsigned group number, unsigned position |
| Position range | Zero through 31 inclusive |
| Result interpretation | Signed result classified only as zero or nonzero |
| Zero-result continuation | Indexed assignment without a dedicated bookkeeping decrease |
| Nonzero-result continuation | Bookkeeping decrease by one, then indexed addressing and assignment |
| State used afterward | State available after dependency completion and any subsequent bookkeeping decrease |

Dependency internals, validation guarantees, preservation guarantees, ownership effects, resource effects, and abnormal-completion mechanisms are **documented_only**.

No other explicit function dependency is established.

## Acceptance examples

These are conditional acceptance requirements, not observed executions.

Unless an example changes an assumption, assume conventional 32-bit two’s-complement integer types, byte addressing, four-byte integer objects, supported integer-derived addresses, appropriate alignment, live accessible storage, disjoint storage roles, stable base observations, no interference after dependency completion, and valid arithmetic for the selected position.

“Available” values are values after dependency completion.

### Ordinary outcomes

| Index | Dependency result | Available bookkeeping | Available selected element | Required outcome |
|---:|---|---:|---:|---|
| 1 | Positive nonzero | 7 | 15 | Bookkeeping becomes 6; element becomes 13. |
| 4 | Zero | 9 | 16 | Element becomes zero; bookkeeping remains 9 under the isolation assumptions. |
| 2 | Nonzero | 8 | 8 | Bookkeeping becomes 7; the indexed assignment leaves the element numerically 8. |
| 0 | Negative nonzero | 0 | 1 | Bookkeeping becomes −1; element becomes zero. |
| 30 | Zero | 6 | 1,073,741,824 | Element becomes zero; bookkeeping remains 6 under the isolation assumptions. |

Index 32 selects position zero in the second group. Under the stated byte-layout assumptions, that element begins four bytes beyond the stored base. An available value of 7 becomes 6. No direct assignment to a disjoint first-group element is required.

### State supplied by the dependency

If the dependency leaves bookkeeping at 12 and returns nonzero, the dedicated decrease produces 11 regardless of bookkeeping’s invocation-entry value.

If the dependency changes bookkeeping and returns zero, this function does not restore the earlier value.

If the dependency installs a valid replacement base that remains stable, indexed addressing uses the replacement. If it invalidates required storage, normal dependency return does not establish a defined completed update.

These examples describe permitted dependency effects, not evidence that the dependency actually produces them.

### Overlapping storage

The following examples replace disjointness only where overlap and compatible storage interpretations are independently valid:

| Condition | Required consequence |
|---|---|
| Bookkeeping and the selected element share value 4; the dependency returns nonzero; position zero is selected | The decrease produces 3, then the indexed assignment produces 2. |
| Those roles share value 5; the dependency returns zero; position zero is selected | No dedicated decrease occurs; the indexed assignment produces 4. |
| The selected element begins at the locator, contains 15, and position zero is selected with a zero result | The assignment produces 14 at that location. |
| The indexed destination overlaps the stored base | The assignment can change the base; no later access through the changed base is required. |

If independently valid differing base observations select separate source and destination elements, a source value of 15 with position zero selected requires a destination value of 14. The destination’s previous value does not determine the preserved bits.

### Arithmetic boundaries

- Index 31 selects position 31 in the first group. A numerical clear result remains conditional on valid signed-mask formation.
- Minimum signed bookkeeping with a nonzero result has no guaranteed defined completion where the decrease overflows.
- Minimum signed bookkeeping with a zero result does not itself invalidate an otherwise valid, disjoint position-zero update.
- For a 32-bit signed locator, 2,147,483,639 permits the eight-unit addition numerically. A locator of 2,147,483,643 permits the four-unit addition but not the eight-unit addition. Neither establishes accessible storage.
- With 32-bit unsigned address arithmetic, a stored base of −4 and index 32 yield numerical address zero. Usable storage there is not established.
- The maximum 32-bit unsigned index, 4,294,967,295, selects group 134,217,727 and position 31. Its displacement is 536,870,908 address units. Neither adequate capacity nor valid mask formation follows.

## Uncertainties

The following remain **documented_only**:

- Application meanings of the locator, bookkeeping, indexed storage, and dependency result.
- Membership rules, idempotence expectations, and accounting invariants.
- Exact integer widths, object sizes, signed representation, address-unit size, pointer width, byte order, and calling convention.
- Applicable shift, complement, overflow, conversion, and integer-derived-address rules.
- Storage capacity, allocation boundaries, accessibility, alignment, lifetime, ownership, and permissible overlap.
- Dependency validation, preservation, resource effects, and abnormal completion.
- Whether the relevant execution model permits differing base observations without invalid interference.
- Physical memory transactions, synchronization, concurrency safety, reentrancy, and device-memory suitability.

These gaps cannot be resolved by assuming the dependency validates inputs, preserves globals, retains storage, or guarantees disjointness.

## Evidence

**Established static evidence:** a signed locator and corresponding-rank unsigned index; no returned value; 32-position grouping; four-unit spacing between indexed group starts; signed storage roles starting four and eight address units beyond the locator; one explicit dependency invocation; a decrease controlled solely by a nonzero dependency result; and an indexed unsigned assignment after either normal result category.

**Established ordering:** dependency completion precedes direct storage operations. The conditional bookkeeping decrease precedes indexed addressing. Source and destination address formation separately refer to the stored base, without an established relative order between those observations. The source value is obtained before the assignment takes effect.

**Conditional conclusions:** numerical bit clearing, concrete byte layouts, overlap outcomes, address-boundary calculations, and acceptance outcomes require their stated arithmetic, representation, storage, and stability assumptions.

**Inferred intent:** removing an indexed membership bit while adjusting associated accounting.

**Unavailable evidence:** dependency internals, independent platform definitions, complete storage and ownership guarantees, and runtime observations.

No runtime tests have been performed.
