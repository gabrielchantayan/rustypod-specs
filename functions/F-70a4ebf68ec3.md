<!-- Trusted provenance: F-70a4ebf68ec3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-70a4ebf68ec3

## Purpose

F-70a4ebf68ec3 exposes a straight-line sequence of unresolved indirect calls, byte reads, byte writes, and 32-bit writes. It clears selected locations, stores scaled dependency results, and extracts byte-sized fields from retained values.

No application-level purpose is established. The available evidence does not identify an object initializer, allocator, resource-release operation, serialization format, or device protocol.

The represented path ends at an invalid-instruction boundary, not a normal return. This contract therefore describes the observable prefix supported by the supplied static representation. It does not promise successful completion.

The representation warns about invalid instruction data and overlapping global symbols. All established behavior below is conditional on that representation accurately describing the executable operations.

## Interface

### Explicit inputs

| Position | Represented width | Established use |
|---|---|---|
| First | Signed 32-bit integer | Supplies bit patterns shifted left by 21 and 23 positions |
| Second | Uninterpreted 32-bit value | Passed to F-D001 |
| Third | Uninterpreted 32-bit value | No explicit use in the represented body |
| Fourth | Unsigned 32-bit integer | Supplies addresses, scaled values, and masked values |

The third input’s lack of explicit use does not establish that dependencies cannot observe its incoming machine state.

### Ambient inputs

The represented behavior also consumes:

- An incoming return-link value, from which F-D001’s target is derived using an offset of `0xfc4`.
- An incoming address-like value, used for a byte read at offset 1.
- An incoming unsigned 32-bit word, whose bits supply the initial byte write.
- An additional 32-bit register result associated with F-D039, subsequently supplied to F-D040.

These are environmental requirements of the represented prefix. Their provenance and calling-convention status are **documented_only**.

### Output and termination

The interface is represented as returning no value. No ordinary return is visible.

The final represented event is an invalid-instruction termination boundary. Whether actual execution traps, invokes a handler, stops, or follows instructions absent from the representation is **documented_only**.

### Widths and arithmetic

The behavioral model uses 32-bit address arithmetic and 32-bit transformed values:

- Address additions, multiplications, and left shifts retain their low 32 bits.
- Byte stores retain the low 8 bits of their supplied value.
- Unsigned right shifts insert zero bits.
- A byte read used in an address calculation is zero-extended.
- A 32-bit store covers four bytes beginning at its calculated address; byte order is unknown.

This is a machine-bit-pattern interpretation, not a claim that every represented signed expression has defined high-level-language overflow behavior. Signed shifts and multiplication require instruction-level confirmation where that distinction matters.

Two recurring quantities are the fourth input’s low 22 bits multiplied by 128 and by 16. Their maximum values are respectively `0x1fffff80` and `0x03fffff0`; neither multiplication overflows 32 bits.

## Behavior

### Dependency and ordering conventions

F-D001 through F-D108 identify unresolved dependency **call occurrences**, in execution order. They do not assert that there are 108 distinct target functions.

A call described as having no explicit arguments has none in the supplied representation. This does not establish an absence of machine-state inputs.

All sequencing below is conditional on earlier calls returning and earlier accesses succeeding. There are no visible branches or loops. A fault or nonreturn can prevent any later event.

### Initial byte update and fourth-input clears

F-D001 receives the second input. F-D002 produces a 32-bit value consumed by F-D003. F-D004 through F-D006 have no explicit arguments; F-D007 then supplies a 32-bit result.

Before F-D008, the routine reads the byte at the ambient address-like input plus 1. The unsigned byte plus `0x11` determines the destination of a byte store. Its value is bits 22 through 29 of the ambient 32-bit word.

F-D008 receives four times F-D007’s result. F-D009 through F-D012 have no explicit arguments.

The next two byte stores are zero clears. The first targets the fourth input plus `0x0f`. The second targets the unsigned byte read at the fourth input multiplied by 1024, plus 1, with `0x11` added to that byte’s value. The direct clear precedes this read.

F-D013 receives four arguments: the first input shifted left by 21 three times, followed by the first input shifted left by 23. F-D014 produces a value passed to F-D015. F-D016 through F-D018 have no explicit arguments; F-D019 supplies another 32-bit result.

The routine now rereads the byte at the fourth-input-times-1024 address plus 1. It captures that byte and clears the byte at its unsigned value plus `0x11`. This is a new observation, not reuse of the earlier read.

F-D020 receives four times F-D019’s result. F-D021 through F-D024 have no explicit arguments. After these calls, the captured byte determines a 32-bit clear at its value plus `0x44`, followed by a 32-bit clear at absolute address `0x24`.

The captured byte is not refreshed for the word clear, even if the intervening dependencies changed its source location.

### First result-indexed clears and split result

F-D025 receives four arguments: zero, `0x98`, the fourth input multiplied by 1024, and `0x98`.

F-D026 produces a value passed to F-D027. F-D028 produces a value passed to F-D029. F-D030 through F-D032 have no explicit arguments. F-D033 then supplies a 32-bit result.

Before F-D034, the routine clears the word at four times F-D033’s result plus `0x44`, then clears the word at absolute address `0x24`. F-D034 receives four times that same result.

F-D035 through F-D037 have no explicit arguments. F-D038 supplies a represented 64-bit result. F-D039 receives its low 32-bit half shifted left by two.

After F-D039, the high 32-bit half of F-D038’s result determines a word clear at twice that half plus `0x44`. Another word clear at `0x24` follows.

F-D040 receives zero, the additional 32-bit register result associated with F-D039, and that additional result shifted left by two. The meaning of this additional result is not established.

### Masked-input calls and zero-valued word stores

F-D041 has no explicit arguments. F-D042 receives the fourth input’s low 22 bits multiplied by 128.

F-D043 through F-D045 have no explicit arguments. F-D046 supplies a 32-bit result. The routine clears the word at four times that result plus `0x44`, then clears the word at `0x24`. F-D047 receives four times F-D046’s result.

F-D048 through F-D050 have no explicit arguments. F-D051 supplies a value whose left shift by two is passed to F-D052. F-D052 itself supplies a 32-bit result. Before F-D053, the routine clears the word at four times F-D052’s result plus `0x14`.

F-D053 receives F-D052’s result in its first three positions and the fourth input’s low 22 bits multiplied by 128 in its fourth position.

F-D054 through F-D056 have no explicit arguments. F-D057 supplies a result used to clear the word at four times that result plus `0x14`. F-D058 then receives four times F-D057’s result.

### Retained and scaled word values

F-D059 through F-D061 have no explicit arguments. F-D062 supplies a 32-bit result.

The routine retains four times F-D062’s result and stores that value at sixteen times F-D062’s result plus `0x14`. F-D063 receives the retained value.

F-D064 through F-D066 have no explicit arguments. F-D067 supplies another 32-bit result. The routine stores the retained value from F-D062 at four times F-D067’s result plus `0x14`. This store does not substitute four times F-D067’s result as its value.

F-D068 receives four times F-D067’s result. F-D069 through F-D071 have no explicit arguments. F-D072 supplies a result used for both the next word’s address and its value: the address is sixteen times the result plus `0x14`, and the stored value is four times the result.

F-D073 receives four times F-D072’s result and sixteen times that result.

F-D074 produces a value passed as F-D075’s first argument. F-D075’s second argument is the fourth input’s low 22 bits multiplied by 16.

F-D076 through F-D078 have no explicit arguments. F-D079 supplies a result determining the final explicit word-store address: four times that result plus `0x14`. The stored value is the fourth input’s low 22 bits multiplied by 128.

F-D080 receives four times F-D079’s result. F-D081 through F-D083 have no explicit arguments. F-D084 supplies an unsigned 32-bit result.

### Final byte-field writes

The next three byte reads use the address formed by multiplying the fourth input by `0x04000000` and adding 1. Only the fourth input’s low six bits affect that multiplication. Each read is a separate observation.

The first read selects a byte-store destination by adding `0x11` to the byte’s unsigned value. The stored value is the low byte of four times F-D084’s result.

F-D085 receives F-D084’s result shifted left by two. F-D086 produces a value passed to F-D087. F-D088 through F-D090 have no explicit arguments. F-D091 supplies a 32-bit result.

The second read at the fourth-input-derived address selects another byte-store destination in the same manner. The stored value is bits 20 through 27 of the earlier F-D084 result. Although F-D091 has just returned, its result is not the source of this byte.

F-D092 receives F-D091’s result shifted left by two. F-D093 through F-D095 have no explicit arguments. F-D096 supplies another unsigned 32-bit result.

The third read at the fourth-input-derived address is captured. The routine writes the low byte of four times F-D096’s result at the captured byte’s unsigned value plus `0x11`. F-D097 receives the captured byte, zero-extended to 32 bits—not F-D096’s result or the destination address.

F-D098 produces a value passed to F-D099. F-D100 through F-D102 have no explicit arguments. F-D103 supplies a 32-bit result.

The final byte read uses a different source address: eight times F-D096’s result plus 1. Its unsigned byte plus `0x11` selects the destination of a store containing bits 20 through 27 of F-D096’s result.

F-D104 receives F-D103’s result shifted left by two. F-D105 through F-D108 have no explicit arguments. The represented path then reaches the invalid-instruction boundary.

## Side effects

The complete represented prefix contains:

- **Seven explicit byte reads.**
- **Eight explicit byte writes.**
- **Fourteen explicit 32-bit writes: ten through computed addresses and four at absolute address `0x24`.**

These counts exclude dependency effects. They describe operations, not necessarily distinct locations.

The following ledger establishes the order and contents of all fourteen word writes. Every address and stored value is interpreted at 32-bit width.

| Order | Destination | Stored value |
|---|---|---|
| 1 | Captured byte from the second times-1024 read, plus `0x44` | Zero |
| 2 | Absolute `0x24` | Zero |
| 3 | Four times F-D033’s result, plus `0x44` | Zero |
| 4 | Absolute `0x24` | Zero |
| 5 | Twice F-D038’s high 32-bit half, plus `0x44` | Zero |
| 6 | Absolute `0x24` | Zero |
| 7 | Four times F-D046’s result, plus `0x44` | Zero |
| 8 | Absolute `0x24` | Zero |
| 9 | Four times F-D052’s result, plus `0x14` | Zero |
| 10 | Four times F-D057’s result, plus `0x14` | Zero |
| 11 | Sixteen times F-D062’s result, plus `0x14` | Four times F-D062’s result |
| 12 | Four times F-D067’s result, plus `0x14` | Retained four-times-F-D062 value |
| 13 | Sixteen times F-D072’s result, plus `0x14` | Four times F-D072’s result |
| 14 | Four times F-D079’s result, plus `0x14` | Fourth input’s low 22 bits multiplied by 128 |

Byte-derived byte-store destinations range from `0x11` through `0x110`, inclusive. The captured-byte-derived word-store starting address ranges from `0x44` through `0x143`.

These are literal address calculations, not established structure-field accesses. Word stores need not be aligned. Computed accesses may overlap one another or other referenced locations; the representation supplies no non-aliasing guarantee.

Dependencies may modify memory or globals between explicit operations. Consequently, repeated reads and repeated clears remain separate observable events. Captured bytes and retained results are reused only where the represented dataflow establishes that reuse.

No explicit allocation, release, ownership transfer, synchronization, rollback, or error-reporting operation is established. There is no visible recovery path for a failed access or dependency call.

## Dependencies

F-D001 through F-D108 are opaque identifiers for indirect-call occurrences. Their actual identities, target equality, argument conventions beyond the represented uses, and implementation behavior are **documented_only**.

F-D001’s target depends on incoming return-link state. Later targets are represented using fixed address arithmetic. That arithmetic does not establish reliable public function identities or a portable dispatch interface.

Consumed results have the following represented widths:

- Ordinary consumed results: 32 bits.
- F-D038: 64 bits, with both halves used independently.
- Additional register result associated with F-D039: 32 bits.

An ignored result does not prove that a dependency returns nothing.

No dependency is assumed to preserve globals, leave memory unchanged, return successfully, or own or release any memory. Represented retention of a particular value is a caller-dataflow observation, not a general preservation guarantee for unknown callees.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Each assumes the relevant prefix is reached, dependencies return, and all required accesses succeed.

1. **Word-write accounting.** Reaching the end of the represented prefix requires fourteen explicit word stores in the ledger’s order: ten computed-address stores and four absolute-address stores. Coincident addresses do not reduce the operation count.

2. **Zero fourth input.** The direct byte clear targets `0x0f`. The times-1024 and times-`0x04000000` source reads both address 1, but remain separate observations and may yield different bytes.

3. **Unsigned byte addressing.** A source byte of `0xff` selects byte-store address `0x110`. If captured for the early word clear, it selects starting address `0x143`. Neither access loads or follows a full-width pointer from the byte’s source.

4. **Masking boundary.** Fourth inputs zero and `0x00400000` produce identical low-22-bit-scaled arguments and stored masked values. Their direct input-plus-`0x0f` clear addresses differ, so complete behavioral equivalence is not required.

5. **Shift widths.** A first-input bit pattern of `0x00000100` yields `0x20000000` for F-D013’s first three arguments and `0x80000000` for its fourth, under the 32-bit model.

6. **Cross-call retention.** If F-D062 returns 3 and F-D067 returns 5, the corresponding stores write 12 at `0x44`, then 12 at `0x28`. The second value is not 20.

7. **Wide-result halves.** If F-D038’s low half is 3 and high half is 5, F-D039 receives 12, and the subsequent high-half-derived word clear starts at `0x4e`. Success requires support for that unaligned access.

8. **Byte-field extraction.** If F-D084 returns `0x12345678`, its first byte write stores `0xe0`, and its later field write stores `0x23`. Intervening dependencies may cause different destination bytes to be selected.

9. **Address truncation.** With fourth input `0xffffffff`, the direct byte clear targets `0x0000000e`, the times-1024 source read targets `0xfffffc01`, and the times-`0x04000000` source read targets `0xfc000001`.

10. **Repeated absolute clears.** If dependencies change the word at `0x24` between explicit clears, each subsequent explicit clear writes zero again. The four operations cannot be treated as a single initialization event.

## Uncertainties

### Established static evidence

The representation supports the described call order, explicit argument transformations, result widths, byte-derived addressing, retained-value reuse, and memory-operation ledger.

### Inferred intent

The repeated clears and scaled stores suggest state manipulation. They do not identify a particular data structure or application role. Constants such as `0x14`, `0x44`, and `0x98` do not independently establish field offsets, lengths, or allocation sizes.

### Documented-only unknowns

The following remain unresolved:

- Whether invalid instruction data affected operation reconstruction or function boundaries.
- The actual calling convention and provenance of ambient inputs.
- Whether F-D038 supplies one logical 64-bit result or paired machine results.
- The meaning of the additional result associated with F-D039.
- Exact machine semantics underlying represented signed arithmetic.
- Endianness, alignment requirements, memory mappings, and permissions.
- Whether accesses reach ordinary memory, device registers, or faulting regions.
- Dependency identities, side effects, ownership rules, and failure behavior.
- Caller-enforced input constraints and valid address ranges.
- Actual termination, recoverability, and error signaling.

No bounds checking or validation is visible. Reachable call targets and valid memory accesses are prerequisites for progressing through the prefix, not conditions that the routine is shown to enforce.

## Evidence

This contract uses only the supplied static function representation, including its invalid-instruction and overlapping-global warnings. The word-write ledger accounts independently for ten computed-address writes and four writes to absolute address `0x24`.

No dependency implementations, independent instruction listing, ABI specification, memory map, or runtime trace were supplied.

No runtime tests have been performed. Acceptance examples express arithmetic and ordering consequences of the conditional static model, not observed executions.
