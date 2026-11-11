<!-- Trusted provenance: F-d2aaef3fa8e5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d2aaef3fa8e5

## Purpose

F-d2aaef3fa8e5 repeatedly invokes indirectly addressed dependencies, reads and modifies externally supplied memory, executes a software breakpoint, and updates implicit execution state. The supplied static representation contains an unconditional loop and no normal return path.

No application-level purpose is established. Object initialization, protocol processing, scheduling, and resource management would be inferred intent rather than supported descriptions.

This contract separates visible local operations from unknown dependency behavior. Every operation after a call, memory access, or breakpoint is conditional on execution reaching it.

## Interface

### Declared interface and implicit inputs

The routine has no declared parameters or return value. It exposes no explicit success indicator, error result, cancellation input, iteration limit, or termination condition.

Its behavior depends on these implicit inputs:

| Contract term | Role |
|---|---|
| Context base | Incoming execution-state value used for 32-bit context accesses and subsequently decreased by `0xF4` bytes. |
| Link state | Incoming value used to resolve the first dependency target in an iteration; explicitly replaced near the iteration’s end. |
| Preserved base | Incoming register-associated value used in the final byte-store address. Initialization is not shown. |
| Global slot A | Slot containing a 32-bit early destination base. |
| Global slot B | Slot containing a 32-bit value used both as a destination base and as a truncated stored value. |
| Dependency outputs | Ordinary results, paired results, and secondary register-associated results consumed locally. |
| Addressable storage | Memory reached through globals, the context, dependency results, narrow integer values, and fixed addresses. |

No buffer lengths, object bounds, permissions, or non-aliasing guarantees accompany these inputs.

### Widths and arithmetic

Visible data accesses are 8, 16, or 32 bits. Three dependency occurrences produce values represented as 64 bits and consumed as low and high 32-bit halves. Whether these are genuine 64-bit returns or paired register results is unresolved.

Unsigned halfword loads are zero-extended. Halfword stores retain the low 16 bits, and byte stores retain eight bits. The explicitly unsigned 32-bit right shift is logical; the explicitly unsigned 32-bit left shift discards bits above bit 31.

Other displayed arithmetic includes signed 32-bit additions, multiplications, and left shifts. A portable signed-overflow policy is not established. In particular, the scaling of the breakpoint-adjacent dependency result, the final low half, and the final high half must not be treated as universally defined language-level arithmetic for arbitrary inputs.

The resolved dependency addresses below are the exact displayed target values. No instruction-set tag removal, address alignment, or other target normalization is established.

Byte order, alignment requirements, accessible address ranges, and fault handling are unknown.

## Behavior

### Occurrences versus target identities

**F-opaque-01** through **F-opaque-37** identify call occurrences in one iteration, not distinct functions. The **F-target-001** through **F-target-031** identities in Dependencies identify the fixed targets selected by those occurrences.

Five pairs of occurrences definitely share a target:

- F-opaque-05 and F-opaque-06.
- F-opaque-09 and F-opaque-13.
- F-opaque-17 and F-opaque-24.
- F-opaque-19 and F-opaque-31.
- F-opaque-25 and F-opaque-32.

Unknown callee semantics do not weaken these statically resolved equalities.

### Opening calls and captured early base

Each iteration initializes a local byte-sized value to zero and invokes F-opaque-01 with two explicit zero arguments. Its target is the current link state plus `0xFFE`. F-opaque-02 and F-opaque-03 follow, in that order.

Global slot A is then read as a 32-bit early destination base. This capture precedes F-opaque-04 and F-opaque-05. F-opaque-05 supplies the first paired result.

F-opaque-06 receives two explicit arguments: the first low half, and the displayed 32-bit sum of the first high half, first low half, and a zero byte. Although F-opaque-05 and F-opaque-06 have different occurrence labels and consume different interfaces, both resolve to F-target-004.

A byte from F-opaque-06’s secondary result channel is stored at the captured early base plus `0x11`. F-opaque-07 follows; a zero byte is then stored at the same captured base plus `0x15`.

The global slot is not reloaded for either store. A dependency changing global slot A does not by itself redirect them. Storage validity after intervening calls remains a precondition for successful access.

### Halfword result and context-related writes

F-opaque-08 supplies a 16-bit ordinary result and a secondary result used as a destination base. The ordinary result is stored as a halfword at that base plus `0x0C`.

F-opaque-09 supplies the second paired result. A zero halfword is stored at its high half plus `0x2C`. Its unsigned low half, logically shifted right by 26, is stored as 32 bits at context base plus `0x26C`. This stored value ranges from 0 through 63.

A local unsigned quantity is assigned `0x21C48960` before F-opaque-10. After that call, a 32-bit destination base is read from context base plus `0x320`. F-opaque-11 and F-opaque-12 execute before a 32-bit store at the captured destination base plus `0x54`. The stored value is the second low half shifted left by two using unsigned 32-bit arithmetic.

Neither that paired result nor the captured destination base is visibly refreshed across the intervening calls.

### Fixed-address read and breakpoint boundary

F-opaque-13 through F-opaque-17 execute in order. F-opaque-13 revisits the target used by F-opaque-09; it does not visibly replace the earlier paired result.

An unsigned halfword is then read at fixed address `0x21C48994`, derived from `0x21C48960 + 0x34`. Its zero-extended value is the **first narrow base**.

After F-opaque-18, a 32-bit value is read at context base plus `0x280` and passed to F-opaque-19. The routine then executes a software breakpoint with immediate `0x92`.

The representation contains subsequent operations, but continuation after the breakpoint is environment-dependent. If execution does not resume, none of those later operations is required to occur.

### Post-breakpoint addressing and halfword stores

On continuation, the result of F-opaque-19 is shifted left by two to form an indexed displacement. Separately, unsigned arithmetic on the retained constant produces:

- `0x21C48960` shifted right by 18: `0x871`.
- That result multiplied by four: `0x21C4`.

A 32-bit **object base** is read at first narrow base plus `0x44`. F-opaque-20 executes, followed by F-opaque-21. A 32-bit read at first narrow base plus `0x04` supplies the second argument to F-opaque-22; the first argument is F-opaque-21’s result.

Global slot B is read after F-opaque-22 and before F-opaque-23. Following F-opaque-23, a halfword is stored at the captured slot-B value plus the indexed displacement. The stored value is the low 16 bits of that same captured slot-B value—not the displacement or a separate dependency result.

F-opaque-24 then executes. A 32-bit **intermediate base** is read at fixed address `0x2218`, the sum of `0x21C4` and `0x54`. F-opaque-25 supplies a destination base, and the halfword `0x21C4` is stored at that base plus `0x04`.

After F-opaque-26 and F-opaque-27, an unsigned halfword is read at intermediate base plus `0x34`. Its zero-extended value is the **narrow index**.

### Final paired result and context movement

F-opaque-28 supplies the final paired result. Its low half is first stored as 32 bits at context base plus `0x258`. An unsigned halfword is then read at final high half plus `0x14`; its zero-extended value is the **narrow field**.

The routine prepares the final low half shifted left by six and the final high half multiplied by four. These are displayed as signed 32-bit operations, with the arithmetic limitations stated in Interface.

The context base is explicitly decreased by `0xF4`, or 244 bytes. After this adjustment, the final low half is stored as 32 bits at four times the narrow index plus `0x74`.

The context store therefore uses the context before its explicit adjustment. The narrow-index-derived store is independent of the context base. Since the narrow index is an unsigned halfword, its four-times value is between 0 and `0x3FFFC`, without overflow.

The representation establishes a context-state update, not that this is necessarily a valid stack adjustment.

### Final object updates and repetition

F-opaque-29 precedes a 32-bit store of the captured narrow field at object base plus `0x54`. F-opaque-30 precedes another 32-bit store of the same value at object base plus `0x64`. The halfword has been zero-extended for both stores.

F-opaque-31 then executes. The low 16 bits of the captured object base are stored at the sum of the final low half shifted left by six and F-opaque-31’s secondary result. F-opaque-31 selects the same fixed target as F-opaque-19, but its ordinary result is not consumed here.

After F-opaque-32 and F-opaque-33, one byte is read at five times the narrow index. F-opaque-34 executes before that captured byte is stored at narrow field plus preserved base plus `0x19`. This is not a fresh byte read after F-opaque-34.

F-opaque-35 precedes a 32-bit store of the scaled final high half at object base plus `0x44`. F-opaque-36 follows.

The link state is then assigned `0x08712031`. A 32-bit value is read at scaled final high half plus `0x44` and passed to F-opaque-37. The loop repeats with no local exit test.

Under the displayed carried-state assignment, the next F-opaque-01 target resolves to `0x0871302F`. The explicit context adjustment likewise carries into the next iteration. Effects of dependencies on actual execution state remain separately unresolved.

## Side effects

### Direct writes and their order

A fully reached iteration performs three byte stores, five halfword stores, and seven 32-bit stores. Their local order is:

| Order | Width | Destination | Supplied value |
|---:|---:|---|---|
| 1 | 8 | Early base + `0x11` | Secondary byte from F-opaque-06 |
| 2 | 8 | Early base + `0x15` | Zero |
| 3 | 16 | Secondary base from F-opaque-08 + `0x0C` | F-opaque-08 result |
| 4 | 16 | Second high half + `0x2C` | Zero |
| 5 | 32 | Context + `0x26C` | Second low half logically shifted right 26 |
| 6 | 32 | Captured context-loaded base + `0x54` | Second low half shifted left two |
| 7 | 16 | Captured slot-B value + displacement | Low 16 bits of captured slot-B value |
| 8 | 16 | F-opaque-25 result + `0x04` | `0x21C4` |
| 9 | 32 | Context + `0x258`, before adjustment | Final low half |
| 10 | 32 | Four times narrow index + `0x74` | Final low half |
| 11 | 32 | Object base + `0x54` | Zero-extended narrow field |
| 12 | 32 | Object base + `0x64` | Zero-extended narrow field |
| 13 | 16 | Shifted final low half + secondary result | Low 16 bits of object base |
| 14 | 8 | Narrow field + preserved base + `0x19` | Captured byte |
| 15 | 32 | Object base + `0x44` | Final high half multiplied by four |

These counts exclude dependency effects. Aliasing can cause stores to affect later reads, other destinations, or externally visible state. No concurrency or atomicity guarantees are established.

### Ownership, validity, and failure

No explicit local allocation, release, retain, or ownership transfer appears. This does not establish that dependencies perform none of those operations.

A captured address is not a lifetime guarantee. Every later access requires the addressed storage still to be accessible with the relevant width and permissions. Dependencies are not assumed to preserve globals, allocations, memory contents, or implicit execution state.

There are no visible pointer, bounds, alignment, overflow, or dependency-success checks. No local rollback or error translation is shown. A fault, nonreturning dependency, or nonresuming breakpoint may leave a partially completed iteration’s effects visible.

## Dependencies

F-opaque-01 uses the link-relative resolver described above. The remaining occurrences have these fixed bindings, calculated from the supplied constant target arithmetic:

| Stable opaque target | Resolved target value | Occurrences |
|---|---|---|
| F-target-001 | `0x08712E01` | F-opaque-02 |
| F-target-002 | `0x08712E07` | F-opaque-03 |
| F-target-003 | `0x08712E0B` | F-opaque-04 |
| F-target-004 | `0x08712DF9` | F-opaque-05, F-opaque-06 |
| F-target-005 | `0x08712E59` | F-opaque-07 |
| F-target-006 | `0x08712E5B` | F-opaque-08 |
| F-target-007 | `0x08712E73` | F-opaque-09, F-opaque-13 |
| F-target-008 | `0x08712E5F` | F-opaque-10 |
| F-target-009 | `0x08712E79` | F-opaque-11 |
| F-target-010 | `0x08712E7B` | F-opaque-12 |
| F-target-011 | `0x08712E63` | F-opaque-14 |
| F-target-012 | `0x08712E3F` | F-opaque-15 |
| F-target-013 | `0x08712E65` | F-opaque-16 |
| F-target-014 | `0x08712E89` | F-opaque-17, F-opaque-24 |
| F-target-015 | `0x08712EC9` | F-opaque-18 |
| F-target-016 | `0x08712EDD` | F-opaque-19, F-opaque-31 |
| F-target-017 | `0x08712E9D` | F-opaque-20 |
| F-target-018 | `0x08712EAD` | F-opaque-21 |
| F-target-019 | `0x08712EAF` | F-opaque-22 |
| F-target-020 | `0x08712E7D` | F-opaque-23 |
| F-target-021 | `0x08712EBB` | F-opaque-25, F-opaque-32 |
| F-target-022 | `0x08712EE1` | F-opaque-26 |
| F-target-023 | `0x08712EF9` | F-opaque-27 |
| F-target-024 | `0x08712F05` | F-opaque-28 |
| F-target-025 | `0x08712EC7` | F-opaque-29 |
| F-target-026 | `0x08712ED9` | F-opaque-30 |
| F-target-027 | `0x08712E9F` | F-opaque-33 |
| F-target-028 | `0x08712EC5` | F-opaque-34 |
| F-target-029 | `0x08712F03` | F-opaque-35 |
| F-target-030 | `0x08712EED` | F-opaque-36 |
| F-target-031 | `0x08712EFB` | F-opaque-37 |

These are 31 distinct fixed target values across 36 occurrences. The incoming link-relative target may coincide with one of them; its initial value is not established.

The explicitly consumed interfaces are:

- F-opaque-01: two zero arguments.
- F-opaque-05, F-opaque-09, F-opaque-28: paired 32-bit results.
- F-opaque-06: two arithmetic arguments and a consumed secondary byte.
- F-opaque-08: a 16-bit ordinary result and secondary destination base.
- F-opaque-19: one context-loaded 32-bit argument and a result used for displacement.
- F-opaque-21: a 32-bit result forwarded to F-opaque-22.
- F-opaque-22: two explicit 32-bit arguments.
- F-opaque-25: a result used as a destination base.
- F-opaque-31: a secondary result used in addressing.
- F-opaque-37: one 32-bit argument read through the scaled final high half.

Other occurrences show no explicit arguments or locally consumed ordinary results. That absence does not establish an argument-free machine interface. Shared target identity does not prove equal outputs, purity, preservation, or identical effective inputs.

## Acceptance examples

These are conditional deductions, not runtime observations.

1. **Definite target sharing.** F-opaque-05 and F-opaque-06 both select `0x08712DF9`. Assigning different targets to those occurrences would contradict the fixed bindings, even though their consumed interfaces differ.

2. **Repeated target with different result consumption.** F-opaque-09 and F-opaque-13 both select `0x08712E73`. The later call does not visibly refresh the paired result consumed from the earlier occurrence.

3. **Next-iteration resolution.** If execution reaches the next iteration with the displayed link assignment carried forward, F-opaque-01 selects `0x0871302F` and receives two zero arguments.

4. **Unsigned extraction.** If the second low half is `0xFC000003`, its context store at offset `0x26C` receives 63. Its later unsigned left-shifted store receives `0xF000000C`.

5. **Captured base.** If slot A supplies address A and an intervening dependency replaces that slot with address B, the two early byte stores still address A, assuming A remains valid.

6. **Truncation versus addressing.** If slot B supplies `0x12345678`, its indexed halfword store supplies `0x5678`; the destination calculation uses the full captured base.

7. **Breakpoint stops execution.** If breakpoint `0x92` does not resume, post-breakpoint accesses and calls do not occur. Earlier completed effects have no visible rollback.

8. **Unsigned widening.** If the narrow field is `0xFFFF`, both object-field stores supply `0x0000FFFF`, not a sign-extended value.

9. **Index-derived addresses.** If the narrow index is 3, the final-low-half store addresses `0x80`, and the later byte read addresses `0x0F`, provided those locations are accessible.

10. **Read timing and aliasing.** If a dependency changes storage before a later explicit read, that read uses memory at its own execution point. Conversely, changing the byte source during F-opaque-34 does not replace the byte already captured before that call.

11. **No normal completion.** If every call returns, all accesses succeed, and the breakpoint resumes, completion of F-opaque-37 leads to another iteration rather than a routine return.

## Uncertainties

The following remain **documented_only**:

- Application-level purpose and object schemas.
- Dependency implementations and complete calling conventions.
- Whether paired outputs are language-level 64-bit returns.
- Reliability and meaning of secondary register-associated outputs.
- Meaning of the context base and validity of its repeated decrement.
- Initialization and preservation of the incoming preserved base.
- Validity of low numeric addresses derived from halfwords.
- Signed-overflow and signed-shift behavior outside established representable cases.
- Endianness, alignment, memory-mapped-device effects, and concurrency.
- Breakpoint handling and resumability.
- Dependency effects on globals, memory, ownership, and execution state.

Target arithmetic for F-opaque-02 through F-opaque-37 is established, not an uncertainty. Whether those resolved addresses identify useful executable routines in a particular runtime environment remains unknown.

Repeated offsets suggest structured state manipulation, but assigning a particular lifecycle or data model would be inferred intent.

## Evidence

Established static evidence consists of the supplied representation’s unconditional loop, explicit access widths and ordering, fixed constants, target-resolution arithmetic, consumed results, context adjustment, and breakpoint operation.

The fixed target table records the resolved arithmetic rather than treating all indirect calls as unresolved. It preserves all five definite shared-target pairs while keeping call occurrences distinct from target identities.

No dependency bodies, caller guarantees, architecture contract, memory map, or execution observations were supplied. No runtime tests have been performed. All acceptance examples are conditional static deductions.
