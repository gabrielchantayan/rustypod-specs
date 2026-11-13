<!-- Trusted provenance: F-4c4ffc168ba8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4c4ffc168ba8

## Purpose

F-4c4ffc168ba8 performs a fixed visible sequence of indirect calls, computed-address memory accesses, and bit-field transformations. The supplied representation ends at an invalid-instruction boundary, not a normal return.

No application-level purpose is established. The recurring scaling and offset accesses might belong to an internal record or packed-data protocol, but that interpretation is not an acceptance requirement.

This contract describes the visible path conditionally: each operation is reached only if preceding calls return and preceding accesses complete. The supplied control-flow warning limits confidence in the instruction interpretation and function boundary. No runtime tests have been performed.

## Interface

### Entry and exit

The displayed interface declares no parameters and no return value. Nevertheless, execution depends on two incoming machine-state values:

- A register-derived signed 32-bit value used in a left shift by 10.
- A link-state value used to construct the first indirect-call destination by adding `0xFFE`.

These are hidden inputs. The declaration does not establish a conventional parameterless API, an entry calling convention, or sufficient initialization conditions.

The register-derived value is shifted only after the first two calls. The supplied representation does not establish the actual register-preservation obligations of those dependencies.

There is no displayed normal return, success result, error result, or recovery branch. After the final indirect call, the representation terminates at an invalid-instruction boundary. Its actual runtime consequence is documented_only.

### Widths and arithmetic

The visible operations use:

- Signed and unsigned 32-bit integer calculations.
- 32-bit word loads and stores.
- Unsigned eight-bit loads.
- Eight-bit stores.
- One 64-bit call result, divided into two 32-bit portions.

The following names describe values rather than implementation identifiers:

| Name | Meaning |
|---|---|
| Seed | Result of shifting the register-derived input left by 10 |
| Low | Bits 0–31 of the 64-bit result |
| High | Bits 32–63, subsequently interpreted as signed 32-bit |
| Index A through Index N | Results consumed at designated dependency sites |
| Scaled X | Four times the corresponding index |
| Payload | A captured unsigned 32-bit word |
| Packed | An unsigned 32-bit call result |
| Selector | The second unsigned byte loaded near the end |

Unsigned 32-bit multiplication, addition, and left shift retain the low 32 bits. Unsigned right shifts insert zeroes.

Signed calculations require separate treatment. Seed formation, many index multiplications and address additions, and several later left shifts are displayed with signed operands. The supplied material does not resolve overflowing or otherwise invalid signed operations into a portable wrapping rule. Arithmetic examples involving those operations are conditional on a valid arithmetic domain or an established target-machine interpretation.

In particular, Scaled J is held as unsigned after its formation, but its displayed multiplication originates from a signed index. Its later unsigned use does not independently establish how an overflowing multiplication behaves.

### Memory layout and preconditions

Recurring word addresses are:

- `4 × index + 0x14`.
- `16 × index + 0x14`.

These are absolute computed addresses in the supplied representation. No separate object base, allocation extent, array bound, or structure definition is established. The offset `0x14` is 20 bytes.

The late byte loads use `Index J × 0x10000000 + 1`. This calculation originates from a signed index; its overflowing cases remain unresolved. Each loaded byte is interpreted as an unsigned integer and used to form a store address by adding `0x11`. It is not a pointer-sized load or an established object reference.

Reaching any access requires the computed address to be valid for its access width and permissions. Alignment requirements, byte order, and the actual memory map are documented_only.

## Behavior

### Dependency notation and ordering

F-U001 through F-U088 are opaque identifiers for successive call sites, not recovered dependency identities. Different IDs may reach the same implementation.

A range denotes calls in increasing numeric order. Unless arguments or consumed results are specified, no explicit arguments are displayed and no result is consumed by the visible body. This does not establish purity, absence of hidden inputs, or absence of side effects.

The phases below preserve the visible ordering of calls and explicit accesses. They do not impose an otherwise unspecified evaluation order between argument expressions.

### 1. Obtain initial values and publish Seed

F-U001 is called through the destination formed from the incoming link-state value plus `0xFFE`. F-U002 follows.

Seed is then formed by shifting the register-derived signed value left by 10. F-U003 receives Seed twice.

F-U004 produces the 64-bit result. Low is its lower 32-bit portion; High is its upper 32-bit portion interpreted as signed 32-bit.

Seed is stored as a 32-bit word at `4 × High + 0x14`. F-U005 then receives High.

### 2. Forward High

F-U006 through F-U009 execute. F-U009 supplies Index A.

High is stored as a 32-bit word at `4 × Index A + 0x14`. F-U010 receives `4 × Index A`.

F-U011 through F-U014 then execute.

### 3. Publish two Low-derived values

Using unsigned 32-bit arithmetic, a word containing `Low × 0x04000000` is stored at `Low × 0x10000000 + 0x14`.

The value `Low AND 0x1FFFFFFF` is retained for the next store. F-U015 receives `Low × 0x04000000`.

F-U016 through F-U019 execute. F-U019 supplies Index B.

The unsigned value `(Low AND 0x1FFFFFFF) >> 9` is stored as a word at `4 × Index B + 0x14`. Its range is zero through `0x000FFFFF`.

F-U020 receives `4 × Index B`.

### 4. Publish Scaled C and consume word-read expressions

F-U021 through F-U024 execute. F-U024 supplies Index C.

Scaled C is formed as `4 × Index C` and stored as a word at `16 × Index C + 0x14`.

F-U025 receives three explicit arguments:

1. An unsigned word read at `Scaled C + 0x14`, shifted right by 12.
2. Scaled C.
3. An unsigned word read at `Scaled C + 0x14`, shifted left by 2 with unsigned 32-bit truncation.

The preceding write and these reads generally use different addresses: `16 × Index C + 0x14` versus `4 × Index C + 0x14`.

Two word-read expressions are present. Their mutual evaluation order and whether they correspond to one or two physical reads are not established. The contract does not guarantee a single coherent snapshot for both arguments.

### 5. Forward Scaled C and publish Scaled E

F-U026 through F-U029 execute. F-U029 supplies Index D.

Scaled C is stored at `4 × Index D + 0x14`. F-U030 receives `4 × Index D`.

F-U031 through F-U034 execute. F-U034 supplies Index E.

Scaled E is formed as `4 × Index E` and stored at `16 × Index E + 0x14`.

F-U035 receives:

1. Scaled E.
2. Scaled E again.
3. The unsigned 32-bit value `Low << 26`.

Only Low’s bottom six bits contribute to the third argument.

### 6. Forward Scaled E and publish Scaled G

F-U036 through F-U039 execute. F-U039 supplies Index F.

Scaled E is stored at `4 × Index F + 0x14`. F-U040 receives `4 × Index F`.

F-U041 through F-U044 execute. F-U044 supplies Index G.

Scaled G is formed as `4 × Index G` and stored at `16 × Index G + 0x14`.

F-U045 receives Scaled G.

### 7. Forward Scaled G and capture Payload

F-U046 through F-U049 execute. F-U049 supplies Index H.

Scaled G is stored at `4 × Index H + 0x14`. F-U050 receives `4 × Index H`.

F-U051 through F-U054 execute. F-U054 supplies Index I.

Scaled I is formed as `4 × Index I` and stored at `16 × Index I + 0x14`.

An unsigned 32-bit word is then read from `Scaled I + 0x14` and retained as Payload. As in phase 4, the preceding store generally does not target the read address.

F-U055 receives Scaled I after Payload has been captured. A later mutation of the source memory does not, by itself, change that captured value.

### 8. Store a portion of Payload

F-U056 through F-U059 execute. F-U059 supplies Index J, which is retained for later phases.

Scaled J is formed as `4 × Index J` and held as an unsigned 32-bit value, subject to the signed-multiplication qualification in the interface.

`Payload >> 12` is stored as a 32-bit word at `Scaled J + 0x14`. F-U060 receives Scaled J.

### 9. Form arguments using retained Index J

F-U061 through F-U064 execute. F-U064 supplies Index K.

Scaled K is formed as `4 × Index K` and stored at `16 × Index K + 0x14`.

F-U065 receives:

1. `(Scaled J AND 0x3F) << 14`.
2. Scaled K.
3. `Index J << 30`.

Where the scaling has the stated fourfold relationship, Scaled J’s bottom two bits are zero. The first argument can therefore have set bits only in positions 16–19.

The third argument is displayed as a signed left shift. Negative inputs and overflowing cases are not assigned an unconditional wrapping interpretation.

### 10. Forward Scaled K and perform the first byte store

F-U066 through F-U069 execute. F-U069 supplies Index L.

Scaled K is stored at `4 × Index L + 0x14`. F-U070 receives `4 × Index L`.

F-U071 through F-U074 execute. F-U074 supplies Packed.

An unsigned byte is read from `Index J × 0x10000000 + 1`. The low eight bits of `Packed << 2` are stored at the address equal to that loaded byte plus `0x11`.

The byte-store destination is therefore between `0x11` and `0x110`, inclusive, if the preceding address calculation and load complete.

F-U075 receives:

1. The bit pattern of unsigned `Packed << 2`, represented in a signed 32-bit intermediate.
2. The same intermediate.
3. Unsigned `Packed << 4`.

Conversion of an out-of-range unsigned result into the signed intermediate is not independently specified by a target-language contract here. The eight-bit store preserves only the low byte.

F-U075 is the exceptional call site whose destination uses an offset of `0xFEC`, rather than the recurring `0xFFE`.

### 11. Forward an intermediate result

F-U076 produces one four-byte result of otherwise unspecified interpretation. F-U077 receives that result.

F-U078 through F-U081 execute. F-U081 supplies Index M.

### 12. Perform the second byte store

A fresh unsigned byte is read from `Index J × 0x10000000 + 1` and retained as Selector.

This read occurs after the first byte store and multiple intervening calls. Its result is not required to equal the first byte read.

The low eight bits of `(Packed AND 0x3FFFFFFF) >> 20` are stored at `Selector + 0x11`. The stored byte contains bits 20–27 of Packed. Bits 28–29 survive the mask and shift but are discarded by the eight-bit store.

F-U082 receives `Index M << 2`, displayed as a signed shift.

### 13. Final calls and terminal boundary

F-U083 through F-U086 execute. F-U086 supplies Index N.

F-U087 receives:

1. `Index N << 2`, displayed as a signed shift.
2. Selector from the second byte read.

F-U088 follows.

The representation then terminates at the invalid-instruction boundary. No normal return, cleanup sequence, rollback, or success indication is displayed.

## Side effects

The visible body contains **fourteen 32-bit stores**, distributed as follows:

| Phase | Word stores |
|---|---:|
| 1 | 1 |
| 2 | 1 |
| 3 | 2 |
| 4 | 1 |
| 5 | 2 |
| 6 | 2 |
| 7 | 2 |
| 8 | 1 |
| 9 | 1 |
| 10 | 1 |
| 11–13 | 0 |
| **Total** | **14** |

It also contains:

- Two eight-bit stores.
- Three word-read expressions: two supplying F-U025 and one capturing Payload.
- Two explicit unsigned-byte reads.
- Eighty-eight indirect-call sites.

These counts exclude instruction fetches, implicit call machinery, and all dependency behavior. The word-read-expression count does not resolve the number of physical reads performed for F-U025.

Stores are interleaved with calls, not presented as an atomic group or transaction. An interrupted path may leave earlier writes and dependency effects in place.

No explicit allocation, deallocation, ownership transfer, reference-count change, synchronization primitive, or memory barrier is established. Passing an address-related integer does not establish ownership transfer.

Dependencies may mutate memory used later. Aliasing may also cause an earlier store to affect a later read without dependency involvement. No assumption is made that dependencies preserve globals, retain arguments, release resources, or avoid aliasing.

## Dependencies

F-U001 through F-U088 identify call sites only. Their implementations and behavioral contracts are unavailable.

Established properties are limited to:

- Relative invocation order.
- Explicit arguments described in the behavior section.
- Results consumed by the visible body.
- Construction of F-U001’s destination from incoming link state.
- The `0xFEC` destination offset at F-U075.
- The recurring `0xFFE` destination offset at the remaining sites.

Call results used as indices are not visibly validated. Their acceptable ranges depend on the actual memory map, permissions, alignment rules, and arithmetic interpretation.

Calls without displayed arguments are not established as parameterless, pure, deterministic, or removable.

## Acceptance examples

These are conditional consequences of the static representation, not executed tests. Each assumes all preceding calls return, the relevant arithmetic is defined or otherwise established, and every referenced access is valid.

### Zero Low

If Low is zero:

- Phase 3 stores zero at `0x14`.
- F-U015 receives zero.
- The word stored through Index B is zero.
- F-U035 receives zero as its third argument.

This does not constrain dependency effects or establish successful termination.

### Unsigned truncation

If Low is `0xFFFFFFFF`, unsigned 32-bit arithmetic yields:

| Observation | Value |
|---|---|
| Phase 3 first store address | `0xF0000014` |
| Phase 3 first stored word | `0xFC000000` |
| F-U015 argument | `0xFC000000` |
| Word stored through Index B | `0x000FFFFF` |
| F-U035 third argument | `0xFC000000` |

Unbounded multiplication would not satisfy these observations.

### Distinct store and read addresses

If Index C is 1, phase 4 stores the word 4 at `0x24`. The read expressions supplying F-U025 address `0x18`.

The store at `0x24` does not establish the contents read at `0x18`.

### Captured Payload

If the phase 7 read yields `0xABCDEF01`, phase 8 stores `0x000ABCDE` through Index J.

A mutation of the source word during F-U055 does not by itself alter the captured Payload.

### Two independently observed selectors

If Packed is `0x12345678`, the first byte store writes `0xE0`, and the second writes `0x23`.

If the first byte read yields `0x20` and the second yields `0x30`, the stores target `0x31` and `0x41`, respectively. F-U087 receives `0x30` as its second argument.

Replacing the second read with reuse of the first selector would not preserve the visible behavior.

### Complete-path store count

If execution reaches the terminal boundary through all displayed operations, the body has performed fourteen word-store operations and two byte-store operations, excluding dependency effects.

Coincident addresses do not reduce the number of store operations. An interrupted execution need not reach these totals.

### Interrupted execution

If F-U060 does not return, phase 8’s word store has already occurred. Phases 9–13 are not reached through the displayed sequence.

No visible rollback obligation restores the earlier memory state.

## Uncertainties

### Established static evidence

The supplied representation establishes the displayed ordering, explicit access widths, arithmetic expressions, call-result dependencies, fourteen word stores, two byte stores, and terminal invalid-instruction boundary.

These are properties of the supplied static representation, not a verified executable trace.

### Inferred intent

The repeated offset accesses may suggest an internal record protocol. The masks and shifts may suggest extraction or placement of packed fields.

Neither inference establishes an object model, application format, allocation contract, or ownership policy.

### Documented_only unknowns

The following remain unresolved:

- Correctness of the instruction interpretation and function boundary.
- Actual entry convention and preservation of hidden machine-state inputs.
- Dependency identities, hidden arguments, global effects, and ownership behavior.
- Valid memory ranges, permissions, alignment, and byte order.
- Signed overflow, signed left-shift behavior, and unsigned-to-signed conversion details.
- Whether F-U025’s two read expressions produce distinct physical reads.
- Concurrency, interrupts, and memory-mapped-device behavior.
- The runtime consequence of the terminal invalid-instruction boundary.

There is no displayed validation, error return, or recovery branch for invalid indices, inaccessible memory, nonreturning dependencies, or arithmetic-domain violations. Absence of such handling does not establish which runtime failure would occur.

## Evidence

This contract is based solely on the supplied static representation and its control-flow warning. It does not rely on dependency implementations, surrounding source, external documentation, or runtime observations.

The word-store total is established by fourteen explicit word-store sites: two each in phases 3, 5, 6, and 7, and one each in phases 1, 2, 4, 8, 9, and 10.

No runtime tests have been performed. Acceptance examples are arithmetic and ordering consequences under their stated preconditions; they are not execution results.
