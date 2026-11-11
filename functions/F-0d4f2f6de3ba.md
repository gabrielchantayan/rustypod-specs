<!-- Trusted provenance: F-0d4f2f6de3ba; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0d4f2f6de3ba

## Purpose

F-0d4f2f6de3ba partially initializes a destination object using configuration memory, an explicit selector byte, and two incoming 32-bit seed values. It performs two validation stages, selects a base value through an independent configuration selector, and calls a dependency whose result determines whether auxiliary fields receive seed values or configuration-derived values.

The visible return values are 32-bit zero on completion and `0xffffffff` on explicit rejection. Rejection is not transactional: earlier destination writes remain observable.

Initialization is an inferred purpose, not an established application-level meaning. The supplied representation does not establish the ordinary calling convention, dependency implementations, or contents of external data references.

## Interface

### Inputs and result

The visible interface supplies:

- An explicit 32-bit argument that is not visibly used.
- A pointer from which one selector byte is read.

Additional incoming machine state supplies:

- **Destination:** the address of writable state memory.
- **Configuration:** the address of configuration memory.
- **Seed A:** a 32-bit value.
- **Seed B:** a 32-bit value.

These additional inputs are operational requirements of the supplied representation. Their presence does not establish a conventional callable interface or explain how a caller supplies them.

A byte-width assignment from either seed uses its low eight bits. Neither seed is established as zero, one, a Boolean value, or an address.

The function returns:

| Result | Meaning |
|---|---|
| `0x00000000` | The completion path reached its final dependency call, and that call returned control |
| `0xffffffff` | An explicit range, mode, or configuration-selector rejection occurred |

The same rejection value covers all explicit rejection causes. Memory faults and nonreturning dependencies are not represented by additional return values.

### Destination layout

All offsets are byte offsets from Destination. A word is 32 bits.

| Offset | Width | Observable use |
|---|---:|---|
| `0x04`, `0x05` | 8 bits each | Seed-derived fields |
| `0x08`, `0x0c` | 32 bits each | Seed-derived fields |
| `0x14` | 32 bits | Configuration-mode copy |
| `0x28` | 8 bits | Auxiliary-branch indicator |
| `0x2c` | 32 bits | Auxiliary payload |
| `0x30`, `0x34`, `0x38` | 32 bits each | Auxiliary quantities, optionally doubled |
| `0x3c` | 32 bits | Existing or seeded count, optionally incremented |
| `0x40`, `0x41` | 8 bits each | Validation-stage fields |
| `0x42`, `0x43` | 8 bits each | First-selector-dependent fields |
| `0x45`, `0x46` | 8 bits each | Seed-derived fields |
| `0x4c` | 32 bits | Seed-derived field |
| `0x50`, `0x54` | 32 bits each | Configuration copies |
| `0x58` | 8 bits | Stored explicit selector |
| `0x59`–`0x5c` | Four individual bytes | First configuration byte group |
| `0x5d`–`0x60` | Four individual bytes | Second configuration byte group |
| `0x64`, `0x74` | 32 bits each | Configuration copies |
| `0x78` | 32 bits | Selected base |
| `0x7c`, `0x80` | 32 bits each | Auxiliary results |
| `0x84`, `0x88` | 32 bits each | Seed A copies |
| `0x8c`, `0x8d` | 8 bits each | Completion-stage and initial seed fields |

No complete object layout or interpretation of intervening bytes is established.

### Configuration layout

| Offset | Width | Observable use |
|---|---:|---|
| `0x00` | 8 bits | Independent configuration selector |
| `0x04` | 32 bits | Copy source for destination `0x50` |
| `0x08` | 32 bits | Unsigned range-check input |
| `0x0c` | 32 bits | Mode comparisons and later copy |
| `0x10`, `0x11` | 8 bits each | Explicit-selector-4 inputs |
| `0x14` | 32 bits | Copy source for destination `0x54` |
| `0x18` | 32 bits | Copy source for destination `0x64` |
| `0x28` | 32 bits | Copy source for destination `0x74` |
| `0x2c`–`0x2f` | Four individual bytes | First byte-group source |
| `0x30`–`0x33` | Four individual bytes | Second byte-group source |
| `0x40` | Address and 32-bit field | Dependency argument address and auxiliary payload |
| `0x44`, `0x48`, `0x50` | 32 bits each | Auxiliary quantities |

Direct destination accesses extend through byte `0x8d`; direct configuration accesses extend through byte `0x53`. These bounds exclude dependency accesses and externally referenced storage.

### Memory preconditions

Every executed access requires accessible memory of the relevant width. A destination count that is incremented without first being seeded must already support a meaningful 32-bit read.

No null, size, alignment, allocation, or overlap checks are visible. No non-aliasing guarantee is supplied. Reads and writes must therefore be understood in their observed order, rather than as operations on an immutable configuration snapshot.

## Behavior

### Initial state and explicit-selector dispatch

The explicit selector is read once before any visible destination write. That retained byte controls the first dispatch.

Before dispatch, destination writes occur in this order: the selector at `0x58`; configuration words `0x04`, `0x18`, `0x14`, and `0x28` at destination `0x50`, `0x64`, `0x54`, and `0x74`, respectively; Seed A at `0x88` and then `0x84`; and the low byte of Seed B at `0x8d`.

Each configuration copy reads its source before writing its destination. The copies are sequential, not a preloaded group.

The first dispatch has these effects:

| Retained explicit selector | Ordered destination effects |
|---|---|
| 0 or 7 | Low byte of Seed A at `0x42`, then `0x43`; configuration bytes `0x30`–`0x33` copied individually, in ascending order, to `0x5d`–`0x60` |
| 1, 2, or 5 | Low byte of Seed A at `0x42`, then `0x43` |
| 3 | Low byte of Seed A at `0x42`, then `0x43`; configuration bytes `0x2c`–`0x2f` copied individually, in ascending order, to `0x59`–`0x5c` |
| 4 | Configuration byte `0x10` copied to `0x42`, followed by configuration byte `0x11` copied to `0x43` |
| 6 or any other byte | No dispatch-specific write |

An unsupported explicit selector does not itself reject. Unselected destination byte groups are not cleared.

### Range validation and count seeding

Configuration word `0x08` is first read as unsigned. The low byte of Seed A is then written to destination `0x40`, before any range rejection.

If the first range read is at most 8, this stage leaves destination count `0x3c` untouched. Otherwise, configuration `0x08` is read again. A second value greater than 16 causes immediate rejection; any second value at most 16 causes Seed B to be written to destination `0x3c`.

For stable configuration memory:

- Values 0 through 8 leave the count unchanged.
- Values 9 through 16 seed the count with Seed B.
- Values 17 through `0xffffffff` reject.

The repeated read is significant. Once the first value exceeds 8, a second value at most 16 is accepted even if it is no longer greater than 8. A signed-negative interpretation does not make a high-bit-set range value acceptable: these comparisons are unsigned.

### Mode validation and count adjustment

Configuration word `0x0c` is compared with 1.

If equal, destination `0x41` receives the low byte of Seed B. Otherwise, destination `0x41` receives the low byte of Seed A before configuration `0x0c` is read again and compared with 2. A second value other than 2 rejects. A second value equal to 2 causes destination count `0x3c` to be read, incremented by one, and written back.

Thus, with stable memory, mode 2 increments either the count seeded by the range stage or the pre-existing destination count. The increment is expressed as a signed 32-bit operation; overflow behavior is not established by the supplied representation.

After mode acceptance, configuration `0x0c` is read again and copied to destination `0x14`. This copied value need not match either comparison value if memory changes.

The subsequent destination writes, in order, are Seed A at `0x08` and `0x0c`; its low byte at `0x05` and `0x04`; Seed A at `0x4c`; the low byte of Seed B at `0x45`; and the low byte of Seed A at `0x46`.

### Independent configuration selector and base selection

The function now reads configuration byte `0x00`. This dispatch is independent of the retained explicit selector.

| Configuration selector | Effect |
|---|---|
| 0, 3, or 5 | Store Seed A at destination `0x78`; continue |
| 1 | Read the 32-bit value from the external reference designated **Base-1**, store it at `0x78`, and continue |
| 7 | Read the 32-bit value from the separate external reference designated **Base-7**, store it at `0x78`, and continue |
| 2, 4, 6, or any other byte | Reject |

Base-1 and Base-7 designate separate external data references. Their numerical values are unknown and may be equal. Neither numerical inequality nor immutability of the referenced contents is established.

The two selectors are not compared. In particular, explicit selector 4 can proceed when the configuration selector is accepted.

### Auxiliary branch and derived values

For an accepted configuration selector, F-6d8c85350b66 is invoked with the address of configuration `0x40`. Its 32-bit result is tested only for zero versus nonzero.

**Zero result.** Destination `0x28` receives the low byte of Seed A. Seed A is then written, in order, to `0x2c`, `0x30`, `0x34`, `0x38`, `0x7c`, and `0x80`. These are seed assignments, not established clearing operations.

**Nonzero result.** Destination `0x28` receives the low byte of Seed B. Configuration word `0x40` is then copied to destination `0x2c`. Configuration words `0x44`, `0x48`, and `0x50` are read in that order; each is retained and immediately copied to destination `0x30`, `0x34`, and `0x38`, respectively.

After those copies, destination byte `0x58` is read afresh. If it equals 7, destination `0x30`, `0x34`, and `0x38` are overwritten, in that order, with their corresponding retained configuration quantities shifted left by one bit. The decision does not use the original retained explicit selector. The shifts are signed-looking 32-bit operations; negative operands and signed-overflow boundaries remain unresolved.

The derived results use a fresh unsigned 32-bit read of destination `0x78` as the retained base:

- Destination `0x30` is read and added to that base; the 32-bit result is stored at `0x7c`. The unsigned base makes this addition unsigned, with reduction modulo \(2^{32}\).
- Destination `0x34` is read for an unsigned comparison with the base. If greater, `0x34` is read again and the base is subtracted using unsigned arithmetic. Otherwise, the selected result is zero. That result is stored at `0x80`.

With stable memory and ordinary representable values, the second result is the nonnegative excess of `0x34` over the base. With mutation between its comparison and subtraction reads, it is not necessarily a clamped difference: the subtraction can wrap modulo \(2^{32}\).

Destination `0x38` does not participate in either derived result. Dependency effects and aliasing can alter the values observed by these fresh reads.

### Completion

After either auxiliary branch, the function obtains a counter address from an external reference. It then writes the low byte of Seed A to destination `0x8c`, reads the addressed 32-bit counter, increments it by one, and writes it back.

The counter increment is represented as signed 32-bit arithmetic. Overflow behavior is not established.

F-266a29fc8dc5 is subsequently called with an externally referenced first argument and a second argument of zero. Its return value is unused. If it returns control, this function returns zero.

The counter address is obtained before the destination `0x8c` write; the counter contents are read afterward. That distinction can matter if storage overlaps.

## Side effects

Direct effects are destination mutation and, on completion, an external counter increment. No allocation, release, ownership transfer, or returned storage is visible in this function.

Explicit rejection preserves preceding effects:

- **Range rejection:** initial writes, first-dispatch effects, and destination `0x40` have occurred.
- **Mode rejection:** those effects plus destination `0x41` and any range-stage count assignment have occurred.
- **Configuration-selector rejection:** mode acceptance, any count increment, the mode copy, and the subsequent seed-derived writes have occurred.

No explicit rejection path calls either dependency or increments the completion counter.

Dependencies may have additional effects. Their call sites establish no purity, ownership, or global-preservation guarantees. Completion does not guarantee that every earlier destination write still has its assigned value after the final dependency returns.

Locations not directly written retain their previous contents only if aliasing, dependencies, or other actors do not change them. No atomicity, synchronization, or rollback mechanism is visible.

## Dependencies

| Opaque function ID | Established call-site behavior | Unresolved behavior |
|---|---|---|
| F-6d8c85350b66 | Receives configuration address plus `0x40`; zero selects seed assignments, nonzero selects auxiliary copying and derivation | Validation meaning, accessed extent, mutations, ownership, global effects, exceptional behavior, whether it returns |
| F-266a29fc8dc5 | Called after the counter increment with an externally referenced argument and zero; result unused | Argument meaning, mutations, ownership, global effects, exceptional behavior, whether it returns |

External data also supplies Base-1, Base-7, the counter address, and the final dependency’s first argument. Their contents are unavailable. Separate base references do not imply distinct values.

## Acceptance examples

These are conditional static consequences, not observed executions. Unless stated otherwise, assume accessible, stable, non-overlapping memory, returning dependencies that preserve relevant state, and arithmetic within the established domain.

1. **Lower boundary preserves the count.** With range value 8, mode 1, configuration selector 0, and a zero classification result, destination `0x3c` remains unchanged. The auxiliary branch assigns Seed A, the counter increments, and completion returns zero.

2. **Seeded count is incremented.** With range value 9, mode 2, and Seed B equal to 10, destination `0x3c` becomes 11. Completion still requires an accepted configuration selector and returning dependencies.

3. **Upper boundary and rejection differ.** Stable range value 16 seeds the count and proceeds to mode validation. Stable value 17 rejects after destination `0x40` is written, before mode-stage writes or either dependency call.

4. **Invalid mode leaves partial initialization.** With range value 12 and stable mode 3, destination `0x3c` receives Seed B and `0x41` receives Seed A’s low byte. Rejection occurs before the mode copy to `0x14`.

5. **Selectors need not agree.** Explicit selector 4 copies configuration bytes `0x10` and `0x11`. Configuration selector 0 can subsequently proceed. Conversely, configuration selector 4 rejects even when the explicit selector is accepted by its own dispatch.

6. **Separate references may produce equal bases.** If Base-1 and Base-7 both supply 10, configuration selectors 1 and 7 each assign 10 to `0x78`. Different reference identities do not require different stored values.

7. **Doubling affects all three auxiliary quantities.** If classification is nonzero, destination `0x58` is still 7, the retained base is 10, and configuration quantities are 3, 8, and 4, destination `0x30`, `0x34`, and `0x38` become 6, 16, and 8. Derived destinations `0x7c` and `0x80` become 16 and 6.

8. **Difference and unsigned addition boundaries.** A stable destination `0x34` equal to or below base 10 produces zero at `0x80`. Separately, an unsigned base of `0xffffffff` added to destination `0x30` equal to 1 produces zero at `0x7c`.

9. **Repeated reads are independent observations.** If the first range read is 9 and the second is 0, the count is seeded rather than rejected. If the first mode read is 3 and the second is 2, the count increments; a later mode-copy read can observe another value.

## Uncertainties

- **documented_only — Calling boundary:** the additional incoming state may reflect a nonstandard entry point, an internal fragment, or incomplete interface information.
- **documented_only — Intent:** “count,” “base,” and “auxiliary” describe local use, not verified domain semantics.
- **documented_only — External data:** referenced values are unknown. Base-1 and Base-7 may be equal; no immutability guarantee is supplied.
- **documented_only — Signed arithmetic:** count and counter overflow, signed left shifts, and signed interpretation of out-of-range unsigned subtraction results require additional platform or instruction evidence. No explicit overflow rejection or saturation is visible.
- **documented_only — Platform:** endianness, alignment tolerance, memory-mapped behavior, and atomicity are unspecified.
- **documented_only — Aliasing and mutation:** no disjointness or stability guarantee is supplied. Ordered accesses document local observations, not a concurrency protocol.
- **documented_only — Dependencies:** preservation of globals, object contents, ownership, and normal return cannot be assumed.

## Evidence

Established static evidence is the supplied function representation: access widths and offsets, visible access ordering, repeated reads, dispatch membership, comparisons, arithmetic operand types, external references, dependency calls, and explicit results.

The representation establishes two separate base-data references, not unequal numerical values. It also distinguishes unsigned derived addition and subtraction from the unresolved signed increments and shifts.

Initialization intent and field-role names are interpretive conveniences. They do not justify treating the seeds as constants or the dependencies as pure.

No runtime tests, implementation inspection beyond the supplied representation, dependency inspection, or instruction-level validation have been performed.
