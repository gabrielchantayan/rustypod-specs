<!-- Trusted provenance: F-aa093c7bba58; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aa093c7bba58

## Purpose

The established behavior is an ordered sequence of indirect dependency calls and direct memory accesses. The operation extracts bytes from 32-bit values, scales values for arguments and addresses, follows two loaded 32-bit values, and writes to both derived addresses and fixed address `0x0D`.

No application-level purpose is established. Object initialization, device control, serialization, and resource management are not supported interpretations.

The visible sequence ends at an invalid-instruction boundary. It does not establish normal return, a success result, or behavior beyond that boundary. This contract specifies the visible sequence conditionally: each later action requires all preceding calls and accesses to complete.

## Interface

### Entry and exit

The represented interface has no explicit parameters and no explicit return value. Nevertheless, execution depends on implicit entry-machine state:

| Contract role | Required value and use |
|---|---|
| Entry continuation | An address-related value used to form the first call target by adding `0xEE8`. |
| Entry base | An address-related value used for a byte read at offset `0x01` and a byte write at offset `0x0D`. |
| Entry word | An unsigned 32-bit value whose bits supply the first byte written. |

These are descriptive roles, not established public parameter names.

No normal exit is visible. The absence of an explicit return value does not establish a callable, normally returning procedure.

### Data widths and arithmetic

Direct accesses have two widths:

- Byte: 8 bits.
- Word: 32 bits.

Two dependencies produce represented 64-bit results. Each is consumed as separate low and high 32-bit halves. Whether these represent wide scalar returns, paired register outputs, or another calling convention is unresolved.

Unless otherwise qualified, right shifts of unsigned values are logical. Byte writes retain only the low eight bits of the supplied value.

The representation distinguishes unsigned 32-bit values from signed 32-bit intermediates:

- The initial word, linked base, late word, split-result high halves, and secondary register value are unsigned.
- Several dependency results used for scaling, the dimension result, and the later index are represented as signed 32-bit values.
- The first split result’s low half is interpreted as signed 32-bit before its left shift.
- The late word is shifted as unsigned, then retained in a signed 32-bit intermediate.

For unsigned 32-bit expressions, the represented arithmetic retains the low 32 bits. This includes unsigned left shifts and the first split high half’s multiplication by four. Signed left shifts, signed multiplication, signed address addition, and out-of-range unsigned-to-signed conversion require separate caution: their actual machine behavior is **documented_only** without instruction-level confirmation. They must not all be assigned a universal wraparound rule.

“Scaled by four” below denotes a 32-bit left shift by two unless multiplication is explicitly specified. “Scaled by 32” denotes a 32-bit left shift by five.

Byte order, alignment restrictions, and the execution environment’s address-space rules are unknown.

### Preconditions

Reaching the visible endpoint requires executable call targets, normally returning dependencies, and permitted memory accesses at every computed address. Word accesses also require whatever alignment and mapping the environment imposes.

No visible validation enforces these conditions. Low addresses, including `0x0D`, receive no special protection or handling.

## Behavior

### Dependency identities and ordering

`F-001` through `F-065` identify call occurrences in execution order. They are opaque identifiers local to this contract, not evidence of 65 distinct target functions.

A call with “no listed arguments” may still consume implicit registers, memory, or global state. Dependencies are not assumed to preserve globals or leave memory unchanged.

### Initial selector and first writes

The first call, `F-001`, targets the entry continuation plus `0xEE8`. After it returns, `F-002` supplies an unsigned 32-bit **initial word**, which is passed to `F-003`. Calls `F-004` through `F-006` follow with no listed arguments.

`F-007` supplies a signed 32-bit result. Before that result is passed onward, the operation reads one byte from entry base plus `0x01`. This byte is the **selector**. It writes one byte to selector plus `0x11`, taking the byte from the entry word logically shifted right by 22.

The selector is used as a numeric byte, not as an address from which to load another pointer. Consequently, this destination is within `0x11`–`0x110`, inclusive.

`F-008` receives the result of `F-007` scaled by four. Calls `F-009` through `F-012` then occur with no listed arguments.

Define the **selector base** as twice the selector, giving an exact range of 0–510. Define the **shifted initial word** as the initial word logically shifted right by three.

The next direct writes occur in this order:

1. The low byte of the shifted initial word is written at entry base plus `0x0D`.
2. The selector byte is written at fixed address `0x0D`.

`F-013` then receives three listed arguments: selector base, `0x98`, and `0x260`. `F-014` follows with no listed arguments. `F-015` receives an unsigned secondary register value observed after `F-014`, logically shifted right by three. That value is not established as an ordinary return value from `F-014`.

Calls `F-016` through `F-018` have no listed arguments. `F-019` supplies a signed 32-bit result; `F-020` receives that result scaled by four and supplies another signed 32-bit result.

Two further byte writes precede the next call:

1. The retained low byte of the shifted initial word is written at selector base plus `0x0D`.
2. Byte `0xC0` is written at fixed address `0x0D`.

`F-021` receives the result of `F-020` scaled by four. Calls `F-022` through `F-024` follow with no listed arguments.

The two fixed-address writes are distinct observable actions separated by calls and another direct write. Neither may be eliminated merely because the later write targets the same address.

### Selector-derived and linked word state

`F-025` supplies a signed 32-bit **dimension result**. The operation writes that result scaled by four as a 32-bit word at selector base plus `0x24`.

It then reads an unsigned 32-bit word from selector base plus `0x74`. This retained value is the **linked base**. The dimension result scaled by 32 is separately retained as the **scaled dimension**.

`F-026` receives, in order:

1. Linked base.
2. Linked base shifted left by one.
3. Scaled dimension.
4. Linked base masked with `0x7FFFFFFF`, then logically shifted right by two.

The second argument uses an unsigned 32-bit shift. The fourth argument removes the original top bit before shifting; it is not generally interchangeable with an unmasked right shift.

After calls `F-027` through `F-030`, all with no listed arguments, the scaled dimension is written as a 32-bit word at linked base plus `0x24`.

Next, a 32-bit word is read from linked base plus `0x74` and passed to `F-031`. Calls `F-032` through `F-034` follow with no listed arguments.

These two word reads use different retained bases. The offsets suggest related storage, but no structure type, pointer ownership, or field meaning is established.

### Split results and indexed word writes

`F-035` supplies the first represented 64-bit result. Its low half, interpreted as signed 32-bit and shifted left by two, becomes the **retained scaled value**. Its high half is unsigned.

Before the next call, the full shifted initial word is written as a 32-bit word to four times that high half plus `0x14`. This address expression uses unsigned 32-bit arithmetic in the representation. The stored value is not truncated to the byte used earlier.

`F-036` receives four listed arguments: the retained scaled value three times, followed by the first high half logically shifted right by three.

`F-037` supplies the second represented 64-bit result. `F-038` receives:

1. The second low half, interpreted as signed 32-bit.
2. The second high half, unsigned.
3. The same second low half again.
4. The second high half logically shifted right by three.

Calls `F-039` through `F-041` have no listed arguments. `F-042` supplies a signed 32-bit **index**.

The retained scaled value is written as a 32-bit word at four times the index plus `0x14`. `F-043` then receives four times the index. These index calculations are represented as signed multiplication, unlike the preceding unsigned high-half address calculation.

Calls `F-044` through `F-046` follow with no listed arguments.

### Two late selector-based byte writes

`F-047` supplies an unsigned 32-bit **late word**. Its unsigned left shift by two is retained in a signed 32-bit intermediate, the **late scaled value**.

The operation reads one byte from linked base plus `0x01` and writes the low byte of the late scaled value to that selector byte plus `0x11`.

Calls `F-048` and `F-049` follow with no listed arguments. `F-050` supplies a signed 32-bit result.

A fresh byte is then read from linked base plus `0x01`. At that newly read byte plus `0x11`, the operation writes the low byte of the late word masked with `0x3FFFFFFF` and logically shifted right by 20.

Thus:

- The first write obtains its byte from the late word shifted left by two.
- The second obtains its byte from original late-word bits 20–27.
- The second destination uses a new memory read, not the earlier selector.

`F-051` receives the result of `F-050` scaled by four. Calls `F-052` through `F-054` follow with no listed arguments.

### Repeated zero writes and endpoint

`F-055` supplies a signed 32-bit result.

The operation reads a byte at late scaled value multiplied by `0x400`, plus `0x01`. It writes a zero byte at the resulting selector byte plus `0x11`.

`F-056` receives, in order:

1. Late scaled value shifted left by three.
2. The same shifted value again.
3. Late scaled value.
4. The result of `F-055` scaled by four.

The multiplication for the selector-read address and the shifts of the retained signed value have the signed-arithmetic uncertainties described above.

Calls `F-057` through `F-059` follow with no listed arguments. `F-060` supplies a signed 32-bit result.

The same selector-read address expression is evaluated again, and a byte is read again. A zero byte is written to that newly selected byte plus `0x11`. Intervening calls may cause this destination to differ from the preceding zero-write destination.

`F-061` receives the result of `F-060` scaled by four. Calls `F-062` through `F-065` follow with no listed arguments.

The visible sequence then reaches the invalid-instruction boundary. No normal return is established.

## Side effects

### Direct access inventory

The complete visible direct-access inventory, excluding all dependency effects, is:

| Access | Count | Distribution |
|---|---:|---|
| Byte writes | **9** | Five initial writes, two late selector-based writes, two zero writes. |
| 32-bit word writes | **4** | Two offset-based writes and two indexed writes. |
| Byte reads | **5** | One initial selector read, two linked-base selector reads, two final selector reads. |
| 32-bit word reads | **2** | One at selector base plus `0x74`; one at linked base plus `0x74`. |

The five initial byte writes comprise the first selector-derived write, the entry-base-relative write, the selector-base-relative write, and both fixed-address writes. The fixed-address writes are already included in the total of nine.

The four word-write destinations, in order, are:

1. Selector base plus `0x24`.
2. Linked base plus `0x24`.
3. Four times the first split result’s unsigned high half plus `0x14`.
4. Four times the later signed index plus `0x14`.

### Aliasing and persistence

No non-aliasing guarantees are established. Direct accesses can overlap one another or memory modified by dependencies. Selector-based byte destinations always lie within `0x11`–`0x110`, but their selecting reads may occur elsewhere.

Ordering is part of the contract. An earlier write may affect a later read, including through overlap. Repeated reads cannot be replaced with cached values across calls.

The listed writes establish actions, not final memory contents. Later direct writes or dependency effects may overwrite any location.

### Ownership, failures, and partial progress

No direct allocation, deallocation, reference-count change, or explicit ownership transfer is visible. Passing an address-like value does not establish borrowing, retention, release, or even interpretation as an address by a dependency.

No validation branch, error return, retry, cleanup, or rollback is visible. If a dependency fails to return or an access faults, subsequent actions are not established as reached. Earlier effects have no visible compensating reversal.

## Dependencies

There are 65 visible indirect call occurrences.

Only `F-001` has a target based on implicit entry continuation. Subsequent call targets are formed from embedded address-related values and offsets. Their private locations are unnecessary to this behavioral contract.

For every dependency, the following remain **documented_only**:

- Actual target identity and whether multiple occurrences share a target.
- Calling convention beyond the listed arguments and consumed outputs.
- Implicit inputs and preservation of registers or globals.
- Memory, resource, and ownership effects.
- Exceptions, traps, non-return, and other failure behavior.

In particular, the secondary value used by `F-015` has an established unsigned shift but an unresolved calling-convention origin.

## Acceptance examples

These are conditional arithmetic examples, not runtime observations. Each assumes execution reaches the described actions and required accesses are permitted. Examples about an early action do not assert that later signed arithmetic is valid.

1. **Selector interpretation.** If the initial selector byte is `0x20`, the first byte destination is `0x31`, and selector base is `0x40`. No additional pointer load is involved.

2. **Byte truncation versus word preservation.** If the initial word is `0x00000918`, the shifted initial word is `0x00000123`. Its two byte writes supply `0x23`; its later full-word write supplies `0x00000123`.

3. **Entry-word extraction.** If the entry word is `0xFFC00000`, its logical right shift by 22 is `0x3FF`; the first byte write supplies `0xFF`.

4. **Dimension scaling without overflow.** If `F-025` supplies 3, the selector-base word write supplies 12, and the linked-base word write supplies 96.

5. **Unsigned mask and shift.** If the linked base is `0x80000004`, the fourth argument to `F-026` is 1. Its unsigned second argument is `0x00000008`. This does not establish accessibility of addresses based on that linked base.

6. **Fresh selectors.** If the two linked-base selector reads yield `0x10` and `0x30`, their respective destinations are `0x21` and `0x41`.

7. **Different late-word portions.** If the late word is `0x01200003`, its scaled value is `0x0480000C`. The first associated byte write supplies `0x0C`; the second supplies `0x12`.

8. **Distinct fixed writes.** If the initial selector is 7, the two direct writes to `0x0D` supply 7 and then `0xC0`, with the specified intervening actions. Final contents are not guaranteed.

9. **Final selector reread without overflow.** If the late word is 1, the late scaled value is 4 and both final selector reads use address `0x1001`. If those reads yield 2 and 9, the zero writes target `0x13` and `0x1A`.

10. **Non-returning dependency.** If `F-001` does not return, none of the subsequent direct accesses is reached through this visible sequence. No alternative continuation is shown.

## Uncertainties

The following remain **documented_only**:

- Application-level intent, public naming, and intended caller.
- Reliability and completeness of the represented control flow.
- Exact machine behavior for signed overflow, problematic signed shifts, and out-of-range signed conversion.
- Address-width enforcement, endianness, alignment, mapping, and privileges.
- Dependency contracts and implicit-state preservation.
- The semantic interpretation of both split results.
- Whether the endpoint represents an executed invalid instruction, unreachable data, or incomplete reconstruction.
- Any behavior beyond that endpoint.

**Inferred intent:** repeated offsets, scaling, and split outputs suggest manipulation of structured machine state. This inference does not justify assigning object layouts, units, ownership, or domain-specific meaning.

## Evidence

Established static evidence consists of the supplied representation’s explicit call ordering, argument expressions, consumed result widths, direct-access widths, masks, shifts, offsets, and endpoint warnings.

The direct-access counts follow the visible accesses: nine byte writes, four word writes, five byte reads, and two word reads. Dependency-internal accesses are unknown and excluded.

Warnings about invalid instruction data and truncated control flow limit conclusions about completeness and termination. They do not establish normal completion.

No runtime tests have been performed. Acceptance examples are conditional consequences of the visible arithmetic and ordering, not observed executions.
