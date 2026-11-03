<!-- Trusted provenance: F-af29aaa9be87; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-af29aaa9be87

## Purpose

F-af29aaa9be87 transfers four 32-bit quantities from corresponding source offsets to destination offsets, then updates an adjacent four-byte region through two byte-sized writes and a final 32-bit write.

The established contract is an ordered set of memory accesses. It is not an atomic structure assignment, an overlap-safe block copy, or a snapshot of the source at entry. Earlier destination writes can alter memory that later source accesses read.

**Inferred intent:** selective copying between similarly laid-out objects, with separate handling of two flag bits.

That intent does not establish object types or field meanings. Whether the final four-byte region becomes a complete copy of the source region depends on byte order, aliasing, and memory stability.

## Interface

### Arguments

The function takes two address-bearing integer arguments:

1. **Destination base:** identifies memory that will be read and modified.
2. **Source base:** identifies memory that will be read.

The supplied representation establishes their use as addresses, but does not independently establish the address-space width, portable pointer conversion rules, or behavior when address arithmetic overflows.

All offsets below are byte offsets.

| Relative location | Width | Source access | Destination access |
|---|---:|---|---|
| +0 through +3 | — | No direct access | No direct access |
| +4 through +7 | 32 bits | Read | Write |
| +8 through +11 | 32 bits | Read | Write |
| +12 through +15 | 32 bits | Read | Write |
| +16 through +19 | 32 bits | Read | Write |
| +20 | 8 bits | Two reads | One read and two writes |
| +20 through +23 | 32 bits | One read | One read and one write |

The byte access at +20 overlaps the 32-bit region beginning at +20. It does not describe a separate storage field.

The four earlier quantities are transferred as uninterpreted 32-bit representations. Their values are not sign-extended, numerically converted, or otherwise transformed.

### Result

There is no explicit return value. Results are observable through memory writes.

There is no status code, copied-size result, success flag, or explicit error result.

### Preconditions for successful completion

All addressed locations must support the listed access widths and directions. In particular:

- The destination must be readable at +20, not merely writable.
- The source must support reads through +23.
- The destination must support writes through +23.
- Address calculations must identify the intended locations.
- Any platform alignment requirements for the 32-bit accesses must be satisfied.

The function contains no visible validation of these conditions. It does not establish that unaligned accesses are supported.

Disjoint regions are not an enforced precondition. Overlap is accepted by the visible control flow, but changes which values later reads observe.

No access to offsets +0 through +3 is required merely because those bytes precede the accessed fields. This statement concerns direct access relative to each base; overlapping bases can give the same physical location a different offset relative to the other base.

## Behavior

### Four ordered field transfers

The quantities at offsets +4, +8, +12, and +16 are transferred in increasing offset order.

Each transfer reads its source quantity and writes that quantity to the corresponding destination location before the next transfer begins. Consequently, the source values are not all captured before the first destination write.

For disjoint, stable memory, these four destination quantities equal the corresponding source quantities. For overlapping memory, each transfer uses the source value present when that particular read occurs. An earlier destination write may already have changed it.

There is no visible equality check, overlap test, alternate copy direction, or staging buffer.

### Two updates to the byte at +20

After the four field transfers, the function reads the destination byte at +20 and retains that value for both subsequent byte updates.

The first source-byte read contributes only bit 0. The first destination-byte write combines:

- Bits 1 through 7 from the retained destination byte.
- Bit 0 from the first source-byte read.

A second, distinct source-byte read contributes bit 1. The second destination-byte write combines:

- Bits 2 through 7 from the retained destination byte.
- Bit 1 from the second source-byte read.
- Bit 0 retained from the first source-byte read.

The destination byte is not reread between these two writes. In particular, the second write does not preserve the upper six bits from whatever value happens to be stored immediately before that write; it uses the initially retained destination byte.

Both writes are present even when their stored values equal the current byte.

The two source-byte reads are separate observations, but **aliasing alone does not make them differ under ordinary stable-memory conditions**:

- If the source and destination byte addresses differ, the intervening destination-byte write does not affect the source byte.
- If those addresses coincide, the retained destination byte and first source-byte read have the same value. Replacing its bit 0 with its own bit 0 leaves that byte unchanged.

Different observations therefore require an additional cause, such as external mutation or device-specific access behavior. Earlier field transfers can change the source byte before these observations, but do not themselves explain a difference between the two observations.

### Final 32-bit update at +20

After the second byte write, the function obtains a 32-bit destination quantity and a 32-bit source quantity, both beginning at +20.

The final destination value contains:

- Numeric bits 0 and 1 from that destination-word read.
- Numeric bits 2 through 31 from that source-word read.

The destination contribution is selected by the mask `0x00000003`; the source contribution is selected by `0xFFFFFFFC`. These masks partition the 32-bit result without overlap.

The final store covers all four bytes at destination offsets +20 through +23.

The supplied expression does not establish which of the two final word reads occurs first. Both occur after the second byte write and supply the final store. No intervening write by this function is shown between those final reads.

### Byte-order-dependent result

The byte operations address the lowest-addressed byte of the final region. The final word operation preserves the lowest two **numeric** bits of its destination input. These refer to the same two bits on a conventional little-endian target, but not on a conventional big-endian target.

For disjoint, stable memory on a conventional little-endian target:

- The byte updates establish the source byte’s low two bits in the destination word.
- The final word update supplies the remaining 30 bits from the source.
- The completed destination word equals the source word.

For disjoint, stable memory on a conventional big-endian target:

- The byte updates affect the highest-address-order significance byte: numeric bits 24 through 31.
- The final word update replaces those bits with the corresponding source-word bits.
- The final word retains numeric bits 0 and 1 from the destination’s entry-state word.

Thus a complete fifth-word copy is a conditional result, not a byte-order-independent guarantee.

For overlapping regions, even the final source-word read may observe earlier destination changes. Results must be derived from the actual access sequence rather than from an entry-state source snapshot.

### Exact aliasing

If both arguments identify the same base, ordinary stable memory retains its original final contents on either conventional byte order.

Each field transfer writes back the value it just read. The byte updates preserve the byte’s existing bits. The final word update combines complementary portions of the same word.

The accesses are nevertheless present. Equal addresses do not cause an early return or suppression of stores.

## Side effects

### Memory writes and intermediate states

A completing invocation has seven writes in the supplied representation:

- Four 32-bit writes at offsets +4, +8, +12, and +16.
- Two 8-bit writes at +20.
- One 32-bit write at +20.

The final region can pass through two intermediate byte values before its final word value is stored. The operation is not presented as an indivisible update.

No direct destination write lies outside offsets +4 through +23. In particular, destination offsets +0 through +3 are not initialized or copied by this function.

### Memory reads

The representation contains nine reads:

- Four 32-bit source reads for the initial transfers.
- One destination-byte read.
- Two source-byte reads.
- One destination-word read for the final update.
- One source-word read for the final update.

These are source-level semantic accesses. They do not establish an exact machine-instruction count, bus-transaction count, or externally observable access trace for an optimized implementation.

### Ownership and lifetime

There are no visible allocations, releases, reference-count changes, or ownership transfers.

If a transferred quantity represents an address or handle, only its 32-bit representation is transferred. No operation on the represented resource is established. There is no evidence of deep copying, resource retention, or object-lifetime management.

No accesses to named global state or calls to external functions are shown. The supplied addresses may nevertheless identify shared or globally reachable memory.

### Failure and interruption

There are no visible checks, error branches, retries, or rollback actions.

If a required access faults or execution is interrupted, earlier completed writes may remain. The source does not establish the fault mechanism, whether a failing store can be partially visible, or whether control can return to the caller after such a failure.

No locking, synchronization, atomicity, or supported concurrent-access model is established.

## Dependencies

No calls to other functions are visible. There are no opaque callee IDs to enumerate.

The behavior depends on platform properties not supplied by a visible callee:

- Integer-address interpretation and address arithmetic.
- Eight-bit byte access and 32-bit word access.
- Byte order.
- Alignment restrictions.
- Memory-access permissions and faults.
- Compiler lowering and memory-model rules.
- Any special behavior of shared or device-backed memory.

The absence of helper calls does not establish guarantees about those facilities.

## Acceptance examples

These examples are conditional consequences of static evidence, not results of runtime tests.

### Disjoint, stable, little-endian memory

Assume valid, disjoint memory on a conventional little-endian target.

The source words at +4, +8, +12, and +16 are:

- `0x11223344`
- `0x55667788`
- `0x99AABBCC`
- `0xDDEEFF00`

The source word at +20 is `0x76543253`. The destination word initially at +20 is `0xA1B2C3AC`.

Required results:

- The first four destination words become the four corresponding source words.
- The first byte write at +20 stores `0xAD`.
- The second byte write at +20 stores `0xAF`.
- The final word write at +20 stores `0x76543253`.

The intermediate byte values retain the original destination byte’s upper bits. The final word store replaces those upper bits using the source word.

### Disjoint, stable, big-endian memory

Assume conventional big-endian ordering, disjoint regions, and stable ordinary memory.

With source word `0x76543253` and entry-state destination word `0xA1B2C3AC` at +20, the final destination word is `0x76543250`.

The source supplies numeric bits 2 through 31. The destination retains its entry-state low two numeric bits, both zero. The earlier byte writes do not modify those low two numeric bits on this byte order.

### Exact aliasing

Assume both arguments identify the same valid base and memory is ordinary and stable.

Required result: all accessed contents retain their entry-state final values. The four field writes, two byte writes, and final word write remain part of the represented operation.

This example does not establish harmlessness for memory whose reads or writes have device side effects.

### Destination shifted forward by four bytes

Assume the destination base is four bytes greater than the source base, all required accesses are valid, and memory is ordinary and stable. Let the entry-state source word at +4 be `0x11223344`.

The first four transfers propagate that value through source-relative offsets +8, +12, +16, and +20. Thus the first four destination words all become `0x11223344`.

The fifth destination word is at source-relative +24. Its result depends on byte order:

- **Conventional little-endian:** it becomes `0x11223344`.
- **Conventional big-endian:** it becomes `0x11223344` with its low two numeric bits replaced by the low two numeric bits of that fifth destination word’s entry-state value.

For example, on the big-endian target, if that entry-state destination word ends in low bits `11`, the final word is `0x11223347`.

The preceding byte writes do not replace the fifth destination word’s low two numeric bits on the big-endian target.

### Distinct byte observations require an additional cause

Assume ordinary stable memory during the byte-update phase.

Whether the source-byte address and destination-byte address differ or coincide, the first destination-byte write alone cannot make the second source-byte observation differ from the first.

If an independently supported memory environment instead supplies different source-byte values at the two reads, the second destination-byte write uses bit 0 from the first observation and bit 1 from the second. This is a conditional description of the data selection, not a concurrency guarantee.

### A later access fails

Assume one or more early destination writes complete, but a later required access cannot complete.

No rollback or successful return is established. Earlier modifications may remain. The exact failure signal and visibility of the failing access are documented_only.

## Uncertainties

- **documented_only — Byte order:** not independently established; it changes the final-region interpretation.
- **documented_only — Address width and arithmetic:** integer arguments are used as addresses, but exact widths, conversion rules, valid ranges, and overflow behavior are unresolved.
- **documented_only — Alignment:** minimum alignment and treatment of unaligned word accesses are unknown.
- **documented_only — Physical access realization:** instruction boundaries, store granularity, bus transactions, and atomicity are not established.
- **documented_only — Concurrent mutation:** no supported synchronization or inter-thread visibility contract is supplied.
- **documented_only — Device memory:** read-triggered changes and write side effects are outside the established ordinary-memory behavior.
- **documented_only — Fault behavior:** exception handling, partial stores, and recovery are unresolved.
- **documented_only — Object semantics:** enclosing type, total object size, field meanings, and resource ownership are unknown.
- **Inferred intent:** selective field copying with flag handling. This does not strengthen the established access-level contract.

## Evidence

Static evidence establishes four ordered 32-bit transfers, a retained destination byte, two source-byte reads, two byte writes selecting bits 0 and 1, and a final 32-bit write selecting destination low bits and source remaining bits.

The overlap and byte-order conclusions follow from those access widths, locations, and masks. In particular, the intervening first byte write cannot by itself change the next source-byte observation in ordinary stable memory, and forward-shifted overlap does not establish an unconditional fifth-word value across byte orders.

No visible helper calls, validation, allocation, explicit error reporting, or return value are present.

No runtime tests have been performed.
