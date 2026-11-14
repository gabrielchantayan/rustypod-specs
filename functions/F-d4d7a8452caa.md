<!-- Trusted provenance: F-d4d7a8452caa; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d4d7a8452caa

## Purpose

The established static behavior is an ordered sequence of 150 indirect-call occurrences interleaved with byte and word memory accesses. The routine propagates its second and third arguments, computes addresses from returned values and incoming register state, clears selected locations, and stores fragments of returned values.

No application-level purpose is established. Initialization, serialization, object construction, and device control remain possible interpretations, not contractual behavior. Repeated offsets do not establish a structure layout; zero writes do not establish allocation, destruction, or cleanup.

The recovered path ends at an invalid-instruction boundary. A normal return or successful continuation is not established. All behavior below is conditional on execution reaching the relevant event and every preceding call and access completing.

## Interface

### Inputs

The recovered interface contains three 32-bit arguments:

| Input | Established direct use |
|---|---|
| First argument | No explicit use is visible. |
| Second argument | Passed to the first call and stored in a computed word location. |
| Third argument | Stored in two computed word locations. |

These are opaque bit patterns. The first argument’s lack of explicit use does not establish irrelevance to callees: recovered call signatures are incomplete.

Additional inputs include:

- Incoming link-register state, which determines the first call target.
- An incoming unsigned 32-bit register value, designated **R**, used in an address calculation.
- Memory contents, including unsigned address-selection bytes.
- Returned words, one recovered 64-bit result, and an additional register output.
- Any state changes caused by callees.

### Outputs, widths, and preconditions

No ordinary return value is established. Observable effects include calls, memory accesses, and conditional arrival at the invalid-instruction boundary.

Notation:

- **W(a)**: a 32-bit memory location beginning at address *a*.
- **B(a)**: an 8-bit memory location; reads are unsigned.
- **low8(x)**: the least significant eight bits.
- **M(x)**: the result of masking *x* with `0x3fffffff`, then shifting right by one.
- **H(x)**: low8 of the result of masking *x* with `0x3fffffff`, then shifting right by twenty.
- **Q**: the low twenty-two bits of **half**, defined below.

Address, multiplication, and left-shift relationships below use low-32-bit arithmetic. Right shifts of explicitly unsigned words are logical. This is a bit-pattern interpretation of the recovered expressions, not proof of portable signed-overflow behavior. Exact machine overflow and fault semantics remain **documented_only**.

Successful traversal requires valid call destinations and valid accesses of the stated widths. Neither those conditions nor alignment, byte order, memory permissions, or address-space mappings are established.

A byte-selected destination `B(read byte + 0x11)` lies between `0x11` and `0x110`. Such a byte is not a full-width pointer.

## Behavior

### Call-trace contract

The following tables preserve every visible call occurrence. **F-U001** through **F-U150** are opaque occurrence IDs, not resolved callee identities.

A range denotes consecutive calls in increasing ID order. Every call in a four-call range has no explicit recovered arguments; only the last result is retained where identified. Otherwise, “none” means no explicit recovered arguments, not absence of meaningful machine-state inputs. An unmentioned result is not visibly consumed.

For established target relationships, **T** denotes the fixed destination of F-U002. Target entries are hexadecimal byte displacements from T, corresponding left-to-right to the listed occurrences. These are target relationships, not return-value relationships. F-U002 through F-U150 have distinct, statically anchored destinations in the recovered representation. F-U001 instead targets incoming link-register state plus `0xffe`; it could coincide with a fixed destination.

Every intervening call remains observable and potentially state-changing. No table range permits elision, merging, or assumptions of memory preservation.

### Initial word propagation

| Occurrence | Explicit inputs and retained result | Target displacement |
|---|---|---|
| F-U001 | Second argument | Link-derived |
| F-U002–F-U005 | None; last yields **A** | `0, 2, 4, 6` |
| F-U006 | `4A` | `8` |
| F-U007–F-U010 | None; last yields **B** | `14, 16, 18, 1a` |
| F-U011 | **scaled B** | `-1c` |
| F-U012 | None; yields a word | `7c` |
| F-U013 | F-U012’s word | `7e` |
| F-U014–F-U017 | None; last yields **C** | `94, 96, 98, 9a` |
| F-U018 | `4C` | `9c` |
| F-U019–F-U022 | None; last yields **D** | `a8, aa, ac, ae` |
| F-U023 | Word read described below, shifted right by three | `b0` |
| F-U024–F-U027 | None; last yields **E** | `128, 12a, 12c, 12e` |
| F-U028 | `4E` | `130` |
| F-U029–F-U032 | None; last yields **seed** | `13c, 13e, 140, 142` |

The associated accesses occur at these boundaries:

- After F-U005 and before F-U006, the second argument is written to W(`4A + 0x14`). Exactly four calls intervene between F-U001 and this write.
- After F-U010, **scaled B = 4B** is written to W(`16B + 0x14`), before F-U011.
- After F-U017, M(scaled B) is written to W(`4C + 0x14`), before F-U018.
- After F-U022, **scaled D = 4D** is written to W(`16D + 0x14`). W(`scaled D + 0x14`) is then read; its unsigned value shifted right by three supplies F-U023.
- After F-U027, scaled D is written to W(`4E + 0x14`), before F-U028.

The D-derived write and read addresses are not generally equal, but can coincide. Equality holds when `12D` is zero modulo `2^32`, including D equal to zero.

### Third-argument and initial byte writes

Define **base = 16 × seed**, **half = M(seed)**, and **Q = half & 0x3fffff**.

| Occurrence | Explicit inputs and retained result | Target displacement |
|---|---|---|
| F-U033 | `(base, seed shifted left by two)` | `144` |
| F-U034–F-U037 | None | `1c0, 1c2, 1c4, 1c6` |
| F-U038 | W(`base + 0x74`) read immediately beforehand | `1c8` |
| F-U039–F-U042 | None; last yields **shift source** | `1d4, 1d6, 1d8, 1da` |
| F-U043 | Shift source shifted left by two | `1a2` |
| F-U044 | None; yields a word | `218` |
| F-U045 | F-U044’s word | `21a` |
| F-U046–F-U049 | None; last yields **K** | `230, 232, 234, 236` |
| F-U050 | K shifted left by two | `238` |
| F-U051–F-U054 | None | `244, 246, 248, 24a` |

After F-U032, before F-U033, the third argument is written to W(`(R shifted right by three) × 0x80000 + 0x24`).

After F-U037, the third argument is written to W(`base + 0x24`). The word at `base + 0x74` is then read for F-U038. The two third-argument stores are separate events; their destinations may or may not coincide.

After F-U042, B(`base + 1`) is read, then H(seed) is written to B(`read byte + 0x11`), before F-U043.

After F-U049, B(`base + 1`) is read again, then zero is written to B(`newly read byte + 0x11`), before F-U050. Intervening calls may change the selector, so this need not clear the earlier destination.

### Byte clearing and independent forwarding pairs

Define **clear base = half × 0x400**.

| Occurrence | Explicit inputs and retained result | Target displacement |
|---|---|---|
| F-U055 | First three inputs: shift source shifted left by twenty-three; fourth: shift source shifted left by twenty-five | `212` |
| F-U056 | None; yields a word | `2ac` |
| F-U057 | F-U056’s word | `2ae` |
| F-U058–F-U061 | None; last yields **L** | `2c4, 2c6, 2c8, 2ca` |
| F-U062 | L shifted left by two | `2cc` |
| F-U063–F-U066 | None | `2d8, 2da, 2dc, 2de` |
| F-U067 | `(0, 0x98, clear base, 0x98)` | `1fe` |
| F-U068 | None; yields a word | `338` |
| F-U069 | F-U068’s word | `328` |
| F-U070 | None; yields a separate word | `344` |
| F-U071 | F-U070’s word | `346` |

After F-U054, the direct accesses are, in order:

1. Zero written to B(`half + 0x0f`).
2. B(`clear base + 1`) read.
3. Zero written to B(`read byte + 0x11`).

F-U055 follows these accesses.

After F-U061, B(`clear base + 1`) is read again and retained as **saved byte**. Zero is written to B(`saved byte + 0x11`), followed by F-U062 and the four calls F-U063–F-U066.

After F-U066, zero is written first to W(`saved byte + 0x44`), then to W(`0x24`), before F-U067. The word destination uses the saved byte, not a fresh read. Its starting address ranges from `0x44` through `0x143`, without an established alignment guarantee.

F-U068/F-U069 and F-U070/F-U071 are two independent producer-consumer pairs. F-U069’s result is not visibly retained or forwarded to F-U070 or F-U071.

### Repeated word clearing

| Occurrence | Explicit inputs and retained result | Target displacement |
|---|---|---|
| F-U072–F-U075 | None; last yields **clear A** | `35c, 35e, 360, 362` |
| F-U076 | `4 × clear A` | `364` |
| F-U077–F-U080 | None; last yields a recovered 64-bit result | `370, 372, 374, 376` |
| F-U081 | Low 32 bits of F-U080’s result, shifted left by two | `17c` |
| F-U082 | `(0, extra register output, that output shifted left by two)` | `36c` |
| F-U083 | None | `3b4` |
| F-U084 | Q shifted left by seven | `3b6` |
| F-U085–F-U088 | None; last yields **clear B** | `3cc, 3ce, 3d0, 3d2` |
| F-U089 | `4 × clear B` | `3d4` |
| F-U090–F-U093 | None; last yields **N** | `3e0, 3e2, 3e4, 3e6` |
| F-U094 | N shifted left by two; yields **clear C** | `1ec` |
| F-U095 | `(clear C, clear C, clear C, Q shifted left by seven)` | `3ee` |

After F-U075, zero is written to W(`4 × clear A + 0x44`), then W(`0x24`), before F-U076.

After F-U081, zero is written to W(`2 × high 32 bits of F-U080’s result + 0x44`), then W(`0x24`), before F-U082. The additional register output used by F-U082 is associated with F-U081; it is not established to equal either half of F-U080’s result.

After F-U088, zero is written to W(`4 × clear B + 0x44`), then W(`0x24`), before F-U089.

After F-U094, zero is written to W(`4 × clear C + 0x14`), before F-U095.

Together with the preceding phase, these are exactly four explicit 32-bit zero writes to absolute address `0x24`. Calls and other writes intervene; no final-zero invariant follows.

### Additional word propagation

| Occurrence | Explicit inputs and retained result | Target displacement |
|---|---|---|
| F-U096–F-U099 | None; last yields **F** | `440, 441, 443, 445` |
| F-U100 | `4F` | `447` |
| F-U101–F-U104 | None; last yields **G** | `454, 456, 458, 45a` |
| F-U105 | **scaled G** | `45c` |
| F-U106–F-U109 | None; last yields **H** | `4d8, 4da, 4dc, 4de` |
| F-U110 | `4H` | `4e0` |
| F-U111–F-U114 | None; last yields **I** | `4ec, 4ee, 4f0, 4f2` |
| F-U115 | `(4I, 16I)` | `4e6` |
| F-U116 | None; yields a word | `558` |
| F-U117 | `(F-U116’s word, Q shifted left by four)` | `55a` |
| F-U118–F-U121 | None; last yields **J** | `570, 572, 574, 575` |
| F-U122 | `4J` | `577` |

The corresponding writes, each before the next listed explicit-input call, are:

- After F-U099: zero to W(`4F + 0x14`).
- After F-U104: **scaled G = 4G** to W(`16G + 0x14`).
- After F-U109: scaled G to W(`4H + 0x14`).
- After F-U114: `4I` to W(`16I + 0x14`).
- After F-U121: `Q × 0x80` to W(`4J + 0x14`).

The Q-derived multiplication and shift by seven produce the same bit pattern. Neither establishes a length or capacity.

### Final byte updates and boundary

Define **fragment base = half × 0x4000000**.

| Occurrence | Explicit inputs and retained result | Target displacement |
|---|---|---|
| F-U123–F-U126 | None; last yields **fragment A** | `584, 586, 588, 58a` |
| F-U127 | Fragment A shifted left by two | `57e` |
| F-U128 | None; yields a word | `5ec` |
| F-U129 | F-U128’s word | `5ee` |
| F-U130–F-U133 | None; last yields **P** | `604, 606, 608, 609` |
| F-U134 | P shifted left by two | `60b` |
| F-U135–F-U138 | None; last yields **fragment B** | `618, 61a, 61c, 61e` |
| F-U139 | Saved selector byte, zero-extended to 32 bits | `612` |
| F-U140 | None; yields a word | `680` |
| F-U141 | F-U140’s word | `682` |
| F-U142–F-U145 | None; last yields **final index** | `697, 69a, 69c, 69e` |
| F-U146 | Final index shifted left by two | `6a0` |
| F-U147–F-U150 | None | `6ac, 6ae, 6b0, 6b2` |

Four byte updates occur:

1. After F-U126, B(`fragment base + 1`) is read; low8(fragment A shifted left by two) is stored at B(`read byte + 0x11`), before F-U127.
2. After F-U133, the same selector location is read anew; H(fragment A) is stored at the newly selected destination, before F-U134.
3. After F-U138, that selector location is read anew and saved. Low8(`fragment B × 4`) is stored at B(`saved selector + 0x11`). F-U139 receives the selector itself, not fragment B or the destination address.
4. After F-U145, B(`fragment B × 8 + 1`) is read; H(fragment B) is stored at B(`read byte + 0x11`), before F-U146.

Exactly four no-explicit-argument calls, F-U147–F-U150, follow F-U146 before the invalid-instruction boundary.

## Side effects

Direct effects comprise the specified byte and word reads and writes. Calls may introduce additional memory, register, global-state, device-state, allocation, or lifetime effects. No callee is established to preserve selector bytes or previously written locations.

Aliasing is unrestricted by the evidence. A write may affect a later selector read; accesses may overlap one another or callee-managed state. The stated access order therefore matters independently of intended data layout.

No direct allocation or release is identified. Ownership transfer, borrowing, reference counting, and lifetime obligations remain **documented_only**; their absence cannot be inferred from unidentified callees.

No explicit validation, error result, retry, rollback, or transactional boundary is visible. A fault or non-returning call can leave earlier effects observable. The routine is not established to be all-or-nothing.

## Dependencies

Only F-d4d7a8452caa has a supplied resolved opaque function ID. F-U001–F-U150 distinguish unresolved call occurrences without assigning implementations or contracts.

Recovered call forms include zero through four explicit 32-bit arguments, consumed 32-bit results, one consumed 64-bit result, and an additional register output. Full signatures, calling convention, register preservation, target validity, and return behavior remain **documented_only**.

Fixed destination relationships do not establish distinct semantic services, valid function entries, or state independence.

## Acceptance examples

These are conditional static criteria, not executed tests.

1. **Initial multiplicity.** After F-U001 completes, F-U002, F-U003, F-U004, and F-U005 must occur before the first word write. If A is `3`, the second argument is stored at `0x20`.

2. **Unequal D destinations.** If D is `2`, the routine writes `8` at `0x34`, then reads at `0x1c` for F-U023.

3. **Coincident D destinations.** If D is zero, both accesses start at `0x14`. For ordinary coherent memory, with valid accesses and no external modification, the immediately preceding zero write supplies the read value. D equal to `0x40000000` also produces coincident addresses under the stated wrapping interpretation.

4. **Independent forwarding.** If F-U068 returns `0x11` and F-U070 returns `0x22`, F-U069 receives `0x11` and F-U071 receives `0x22`, regardless of F-U069’s return value.

5. **Mask and truncation.** Seed `0xffffffff` gives half `0x1fffffff`. Seed `0x3ff00000` gives H(seed) equal to `0xff`. Fragment A `0x41` gives an initial byte payload of `0x04`.

6. **Fresh versus saved selectors.** Selector values `5` and later `9` select byte destinations `0x16` and `0x1a`. Conversely, saved byte `7` fixes the later word destination at `0x4b`, even if its original selector location changes.

7. **Completion trace.** Reaching the boundary requires F-U147–F-U150 after F-U146. Their effects cannot be omitted, and cleared locations need not remain zero.

## Uncertainties

The following remain **documented_only**:

- Reachability and executability of the damaged recovered path.
- Exact machine arithmetic, byte order, alignment behavior, and memory mapping.
- Complete interfaces and behavior of every unresolved dependency.
- Validity and meaning of incoming register state.
- Ownership, lifetime, aliasing, and asynchronous memory modification.
- Error, exception, fault, trap, and non-return behavior.
- Any continuation or normal return beyond the invalid-instruction boundary.

An initialization-like intent is an inference only and adds no acceptance requirements.

## Evidence

Established evidence is limited to the supplied static representation: recovered widths, explicit access relationships, ordered call occurrences, captured-result associations, target relationships, and the terminal invalid-instruction boundary.

Warnings about invalid instruction data and overlapping global symbols limit control-flow and object interpretation. Absolute accesses are specified by address and width, not as independently typed global objects.

No runtime execution, tests, dynamic call-target resolution, or memory-map validation has been performed.
