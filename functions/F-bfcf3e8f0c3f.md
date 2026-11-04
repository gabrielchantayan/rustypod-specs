<!-- Trusted provenance: F-bfcf3e8f0c3f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bfcf3e8f0c3f

## Purpose

F-bfcf3e8f0c3f writes a five-word record, samples two halfwords, and updates memory according to whether a supplied address is zero. Successful completion returns an opaque 64-bit input unchanged.

The address input has two roles: it determines the first sampling location and selects the update mode. Zero selects additional updates; it does **not** suppress the first read. Under ordinary numeric address interpretation, zero therefore requires reading a halfword at address 4.

Three additional incoming values affect the result but are not represented among the five explicit inputs. Their origins and delivery mechanism are unresolved. A caller-facing description limited to the explicit inputs cannot fully predict the memory effects.

This contract describes the supplied static evidence. It does not establish a supported hardware service, an exception-handling API, or successful execution on any particular target. No runtime tests have been performed.

## Interface

### Explicit inputs and return value

| Input | Width and interpretation | Observable role |
|---|---|---|
| Selector | Unsigned 32-bit integer | Its upper four bits become the first record word. |
| Source address | Signed 32-bit integer used for addressing | Selects the mode and determines the first halfword-read address. |
| Record location | Pointer to 32-bit unsigned words | Receives five consecutive words. |
| Adjustment | Signed 32-bit integer | Determines the third record word through subtraction of 31. |
| Return payload | Opaque 64-bit value | Returned unchanged on normal completion. |

The zero/nonzero distinction examines the source-address value, not the contents of memory at that address. Negative source-address values select the nonzero mode. Their usability as addresses is a separate target-dependent question.

The return payload has no established numeric or ownership interpretation. There is no separate success flag or error result.

### Additional incoming state

| State value | Width | Observable role |
|---|---:|---|
| Base seed | Signed 32 bits | Determines an address base through multiplication by 268,435,456. |
| Recorded value A | Unsigned 32 bits | Supplies the fourth record word. |
| Recorded value B | Unsigned 32 bits | Supplies the fifth record word and one additional assignment in zero-source mode. |

The provenance, initialization requirements, and calling-convention status of these values are **documented_only**. They must not be assumed to be zero, derived from the explicit inputs, or available through a conventional five-argument interface.

### Record layout

The record occupies 20 consecutive bytes, interpreted as five 32-bit words.

| Byte offset | Initially supplied value |
|---:|---|
| 0 | Selector’s upper four bits, represented as an integer between 0 and 15 |
| 4 | Record location converted to an unsigned 32-bit integer |
| 8 | Adjustment less 31, converted to an unsigned 32-bit integer |
| 12 | Recorded value A |
| 16 | Recorded value B |

These are word values, not an endian-independent serialized format. If the execution environment has pointers wider than 32 bits, the evidence does not establish the pointer-conversion rule.

The layout describes the initial record stores. It does not guarantee that these values remain in the record at return: later destinations may overlap it.

### Other storage destinations

Five symbolically identified destinations are designated **A**, **B**, **C**, **D**, and **E** in this contract. They are storage references, not callable dependencies.

Their declarations, addresses, widths, and complete overlap relationships are unavailable. Consequently, this contract can specify the values supplied to their assignments and their ordering, but not the exact bytes those assignments replace.

## Behavior

### Arithmetic and address interpretation

The selector extraction is unsigned and yields an integer in the inclusive range 0–15.

The adjustment calculation is presented as signed 32-bit subtraction before conversion to the unsigned record-word type. It is representable when the adjustment is at least −2,147,483,617. Thus, adjustment 30 produces the unsigned word `0xFFFFFFFF` without requiring signed overflow. Smaller adjustments than that lower bound require arithmetic semantics not established here.

The base calculation is representable in signed 32-bit arithmetic exactly for base seeds from −8 through 7 inclusive. A valid base product alone does not establish that every subsequent operation is valid.

In particular:

- Adding 4 to the source address remains within the signed 32-bit range only when the source address is no greater than 2,147,483,643.
- The base-relative locations use byte displacements of 40, 48, and, in zero-source mode, 59.
- Zero-source mode also needs the value five less than the base. For base seed −8, that subtraction falls below the signed 32-bit minimum.

For the representable base products, the positive displacements listed above remain within the signed range. Address conversion, accessibility, and alignment remain separate requirements.

Unconditional signed wraparound is not established. A target with independently established 32-bit modular behavior might provide meaningful results beyond these arithmetic boundaries, but those results are not guaranteed by this contract.

### Record construction and sampling

The five record words are supplied in increasing byte-offset order: 0, 4, 8, 12, then 16. All five stores precede the two samples in the supplied representation.

The base is derived after record construction. Sampling then occurs in this order:

1. An unsigned 16-bit value is obtained from the location four bytes beyond the source address.
2. An unsigned 16-bit value is obtained from the location 40 bytes beyond the derived base.

Both reads precede every mode-dependent update. Neither read is skipped merely because its value is unused by the selected mode.

The sampled values are captured values. Later uses do not reread the original locations. Where used as 32-bit values, both samples are zero-extended rather than sign-extended.

This ordering matters when storage overlaps. For example, a sample that intersects the record can observe bytes written during record construction rather than the bytes present on entry.

### Nonzero-source mode

A nonzero source address selects the shorter update path.

After both samples, the function performs a four-byte store 48 bytes beyond the base. The supplied word is the zero-extended second sample, so its upper 16 bits are zero.

There are no branch-specific assignments to destinations A through E in this mode. This does not imply that their contents remain unchanged: they might overlap the record or the final store.

The first sample has no subsequent visible use in this mode, but its read remains part of the behavior and can prevent completion if it fails.

Normal completion returns the unchanged 64-bit payload after the final store.

### Zero-source mode

A zero source address selects additional updates after both samples have completed.

The first additional assignment supplies the value five less than the base to destination A. A four-byte store then supplies the zero-extended second sample at the location 59 bytes beyond the base.

Destination B next receives the value `0xFFFFFB00`. After that assignment, destination C receives the zero-extended first sample, destination D receives the base, and destination E receives recorded value B, in that order.

The concluding store supplies the four-byte value `0x0000008D` at **absolute address 48**. It is not a store 48 bytes beyond the original base.

The distinction between absolute and base-relative addressing remains observable whenever the base is nonzero. With a zero base, the final addresses coincide, but the supplied values and preceding updates can still differ between modes.

The store 59 bytes beyond the base is explicitly four bytes wide. Under ordinary byte addressing, a representable base product is four-byte aligned, making this store begin three bytes beyond a four-byte boundary. Whether the target supports this unaligned access, faults on it, or handles it another way is **documented_only**.

Normal completion returns the unchanged payload only after the concluding absolute-address store.

## Side effects

### Access footprint

On normal completion, either mode includes:

- Five explicit 32-bit record stores.
- Two explicit unsigned 16-bit reads.
- One concluding explicit 32-bit store.

Zero-source mode additionally includes a four-byte store at base plus 59 and five assignments to symbolically identified storage. The widths of those five assignments are unresolved; they must not be counted as five independently established four-byte stores.

These counts describe accesses in the supplied representation. They do not establish bus transaction counts, atomic machine instructions, or hardware-visible ordering.

### Aliasing and overlap

No general non-overlap requirement is established among the record, sampled locations, explicit store locations, or destinations A through E.

An accompanying warning reports that some named storage overlaps smaller symbols at the same address. It does not identify sufficient ranges to reconstruct the storage layout. Treating all named destinations as independent objects would therefore be unjustified.

Observable consequences include:

- Record construction can alter bytes subsequently sampled.
- Later updates can overwrite part or all of the record.
- One named assignment can affect storage also associated with another destination.
- The concluding store can overwrite an earlier update.
- Updating a sampled location after sampling does not change the already captured value.

Even if every access succeeds, the values supplied to individual assignments are not necessarily the values still present at every destination on return.

### Preconditions, errors, and partial effects

Normal completion requires defined arithmetic and address conversion, accessible storage with suitable permissions, and target support for every reached access.

Zero-source mode specifically requires the read at address 4, the concluding write at address 48, and the potentially unaligned store at base plus 59 to complete under the applicable address interpretation.

No explicit validation, recoverable error result, rollback, or recovery procedure is visible. A failed access may occur after earlier writes have modified memory. The operation is not established as transactional.

Fault delivery, partial modification by a failed store, resumption, and changes made by any fault handler are **documented_only**. The normal access sequence does not specify an exception-recovery protocol. The unchanged-payload return is promised only for normal completion.

### Ownership and external interference

No allocation, release, ownership transfer, or lifetime extension is visible. Storing the record address as an integer does not transfer ownership of the referenced storage.

There are no visible calls. That does not establish exclusion of interrupts, concurrent mutation, or device behavior. Volatility, compiler barriers, atomicity, and hardware ordering are not independently established.

## Dependencies

No direct or indirect function calls are visible. No dependency function IDs are therefore identified.

Execution depends on:

- The three additional incoming state values.
- The contents and accessibility of both sampled locations.
- Writable storage for the record and every reached destination.
- Target arithmetic, pointer-conversion, alignment, and byte-order rules.
- The missing declarations and placement of destinations A through E.

No callee’s preservation of globals or ownership behavior is assumed. Symbolic storage labels do not establish callable dependencies or a public memory map.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Unless an example states otherwise, assume defined arithmetic and address conversion, successful accesses, no external interference, and non-overlapping storage.

### Record values and nonzero-source result

Suppose the selector is `0xD1234567`, the converted record location is `0x20001000`, the adjustment is 100, and recorded values A and B are `0x11223344` and `0x55667788`.

The initial record words must be, in increasing offset order:

| Offset | Word |
|---:|---|
| 0 | `0x0000000D` |
| 4 | `0x20001000` |
| 8 | `0x00000045` |
| 12 | `0x11223344` |
| 16 | `0x55667788` |

With a nonzero source address and base seed 3, a second sample of `0xABCD` requires a concluding four-byte store of `0x0000ABCD` at `0x30000030`.

The first read must still precede the second read. No branch-specific assignments to destinations A through E occur. The returned payload must match the input payload.

### Zero source and a nonzero base

Suppose the source address is zero, the base seed is 2, address 4 supplies `0x1234`, and base plus 40 supplies `0xBEEF`.

Destination A receives `0x1FFFFFFB` before the four-byte store of `0x0000BEEF` beginning at `0x2000003B`. Destination B then receives `0xFFFFFB00`; destinations C, D, and E subsequently receive `0x00001234`, `0x20000000`, and recorded value B, respectively.

The concluding store must supply `0x0000008D` at `0x00000030`, not `0x20000030`.

Acceptance is conditional on support for the low-address accesses and unaligned store. The named assignments remain subject to unresolved destination widths and conversions.

### Failure during the first sample

Suppose record construction completes, but the first halfword read faults and execution does not resume.

The record may already be modified. The normal sequence has not reached the second read, any branch-specific update, the concluding store, or the return. No rollback is specified.

This applies to nonzero-source mode as well as zero-source mode: an unused first sample is not an optional read.

### Sampling overlapping the record

Suppose a nonzero source address equals the record location.

The first sample addresses the first two bytes of the record word at byte offset 4. Since that word has already been stored, the sample follows the newly written address representation and target byte order, not entry-time memory contents.

Its value still has no later visible use in nonzero-source mode.

### Coincident concluding addresses

With base seed zero, both modes conclude with a four-byte store at address 48.

Nonzero-source mode supplies the zero-extended sample from address 40. Zero-source mode supplies `0x0000008D`. Equal addresses do not make the modes equivalent because the zero-source mode also performs its additional updates.

### Arithmetic boundaries

Adjustment 31 produces a third record word of zero. Adjustment 30 produces `0xFFFFFFFF`.

The minimum signed 32-bit adjustment cannot have 31 subtracted within that signed range. No unconditional record-word value is specified for that case.

Base seed −8 produces a representable base. Nevertheless, the value required for destination A in zero-source mode is outside the signed 32-bit range. That path requires additional target arithmetic evidence.

## Uncertainties

The following remain **documented_only**:

- Provenance and delivery of the additional incoming state.
- Completeness of the explicit interface.
- Pointer width and conversion to an unsigned 32-bit value.
- Behavior outside the representable signed-arithmetic ranges.
- Numeric-address conversion, mapping, and permissions.
- Endianness and unaligned-access behavior.
- Addresses, declarations, widths, conversions, and overlap of destinations A through E.
- Whether accessed storage is ordinary memory, device state, or privileged storage.
- Atomicity, hardware-visible ordering, and external interference.
- Fault delivery, partial access effects, and resumption.
- Operational meaning of the symbolic destination labels.

These unknowns do not justify omitting either sample, treating zero as a no-access request, assuming independent destination storage, or guaranteeing successful return.

## Evidence

**Established static evidence:** five ordered 32-bit record stores; a signed base calculation; two ordered unsigned 16-bit reads; zero/nonzero mode selection; the described mode-specific assignments and explicit stores; and return of the unchanged 64-bit payload after normal completion. No calls, allocation, release, explicit validation, or recovery are visible. A supplied warning identifies overlapping named storage without resolving its layout.

**Inferred intent:** the record may preserve execution state, and the symbolic destinations may relate to exception handling. Neither interpretation is required to explain the established accesses or defines an additional interface guarantee.

**Evidence limits:** independent calling-convention documentation, destination declarations, a target memory map, instruction-level confirmation, and runtime observations were not supplied. All acceptance examples are conditional static criteria. No runtime tests have been performed.
