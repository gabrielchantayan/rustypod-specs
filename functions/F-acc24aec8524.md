<!-- Trusted provenance: F-acc24aec8524; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-acc24aec8524

## Purpose

The supplied static representation describes an operation that interleaves indirect dependency calls with byte and 32-bit memory accesses. It transforms dependency results and incoming machine state into addresses, packed values, and subsequent call arguments.

No application-level purpose is established. In particular, the evidence does not establish that the operation initializes a valid object, processes a buffer, manages a resource, or implements a particular protocol.

The visible path ends at an undecodable-instruction boundary. A normal return is not established. This contract describes the visible prefix, conditional on execution reaching each operation successfully; it does not certify that the representation is an executable or correctly recovered function.

## Interface

**Explicit input.** One nominally signed 32-bit argument. It is forwarded to the first dependency and later used as an address for a one-byte write at offset 1. Its interpretation as an object pointer, handle, integer, or encoded address is unknown.

**Additional incoming state.**

- An incoming link/control value determines the first indirect call target by addition of `0xF90`.
- One incoming 32-bit register value supplies a logical right shift by 22.
- Another incoming 32-bit register value supplies a logical right shift by 2.
- Two later operations consume secondary register outputs from preceding calls. These are not explicit source-level return values.

These dependencies make the visible interface broader than its single declared argument.

**Output.** No explicit return value is established. Observable outputs include dependency invocations, memory writes, and any effects of the dependencies.

**Widths and notation.**

- A byte is 8 bits; a word is 32 bits.
- A double-width dependency result is represented as 64 bits, with a low and a high 32-bit half.
- “Low byte” means reduction modulo 256.
- Logical right shifts fill with zero.
- Address formulas below use 32-bit machine arithmetic, wrapping modulo \(2^{32}\), as a descriptive model. Several expressions are displayed using signed types; their exact overflow and shift semantics require instruction-level confirmation.
- `word(address)` and `byte(address)` describe access widths, not declared object types.
- Descriptive value names below identify retained results, not source identifiers.

**Preconditions.** The visible path contains no explicit validation. Successful execution requires every reached indirect target to be executable and every reached memory access to be permitted with the required width and alignment. No allocation, size, nullability, aliasing, or lifetime guarantee is supplied.

## Behavior

### Dependency notation and execution order

Dependencies have no resolved identities in the supplied map. The opaque identifiers `F-d01-01`, `F-d01-02`, and so forth designate individual call occurrences within the numbered stages below. Distinct identifiers do not assert distinct runtime functions.

A “five-call group” means five consecutive calls. The first receives the stated arguments; the next four have no explicit arguments in the representation. Unless stated otherwise, only the fifth result is consumed. A four-call or three-call group follows the same convention.

“No explicit arguments” does not establish an empty machine-level interface. Hidden register inputs, secondary outputs, global effects, and memory effects remain unknown.

All stages are ordered. A dependency may change memory between accesses; repeated reads must not be replaced by cached values.

### 1. Entry calls and initial packing

`F-d01-01` receives the explicit input. `F-d01-02` produces a word.

The incoming first register value, logically shifted right by 22, supplies a retained **initial field**. `F-d01-03` receives the preceding dependency word and the other incoming register value logically shifted right by 2.

A four-call group, `F-d01-04` through `F-d01-07`, follows. Its final result supplies an **initial index**.

### 2. Initial read and byte write

The initial index is multiplied by 4 to form an **initial base**. A word is read at initial base plus `0x74`. Its low byte is then written at initial base plus 1.

A five-call group, `F-d02-01` through `F-d02-05`, starts with the initial base. A secondary register output associated with the end of this group supplies a **secondary index**.

The word read before the group is written at four times the secondary index plus `0x14`. The representation retains that word across the calls.

### 3. Initial field propagation

`F-d03-01` receives the secondary index. `F-d03-02` produces a word.

`F-d03-03` receives three explicit arguments:

1. That word.
2. The initial field masked with `0x1FF`, then shifted left by 21.
3. The same packed value again.

A four-call group, `F-d03-04` through `F-d03-07`, produces an index. At four times that index plus `0x14`, the operation writes the initial field shifted left by 23.

The two packings differ: the first explicitly selects nine bits before shifting; the second loses any bits beyond the 32-bit result.

### 4. Masked index construction

A five-call group, `F-d04-01` through `F-d04-05`, starts with four times the preceding index and produces a **raw index**.

The operation writes four times the raw index at sixteen times the raw index plus `0x14`.

It then retains:

- **Masked index:** raw index AND `0x3FFFFFFF`.
- **Quarter index:** masked index logically shifted right by 2.
- **Saved scale:** four times the original raw index.

### 5. Two related word slots

A five-call group, `F-d05-01` through `F-d05-05`, starts with the masked index and produces a **slot index**.

The masked index is written at four times the slot index plus `0x14`.

A second five-call group, `F-d05-06` through `F-d05-10`, starts with four times the slot index and produces a **payload word**.

The operation then:

1. Writes the low byte of eight times the slot index at explicit input plus 1.
2. Writes the payload word shifted left by 2 at eight times the slot index plus `0x34`.

### 6. Replacement through a double-width result

`F-d06-01` receives the payload word shifted left by 2. `F-d06-02` produces a word. The payload word masked with `0x3FFFFFFF` becomes the retained **payload base**.

A five-call group, `F-d06-03` through `F-d06-07`, starts with the word from `F-d06-02` and produces a double-width result.

The operation writes:

1. The low byte of eight times the slot index at payload base plus 1.
2. The high half of the double-width result at eight times the slot index plus `0x34`.

Thus, absent intervening changes to address interpretation, the second write targets the same computed slot as the preceding payload write. The dependencies between them may observe or modify that slot.

### 7. Additional byte fields

A five-call group, `F-d07-01` through `F-d07-05`, starts with the low half of the preceding double-width result shifted left by 2. Its final result is retained for the next stage.

Two byte writes follow:

- At `(payload base >> 20) + 0x11`, write the low byte of sixteen times the slot index.
- At sixteen times the slot index plus `0x0D`, write the low byte of the saved scale.

### 8. Repeated scaled arguments

`F-d08-01` receives the preceding retained result shifted left by 2 twice. `F-d08-02` produces a word.

`F-d08-03` receives that word and its logical right shift by 3. A four-call group, `F-d08-04` through `F-d08-07`, produces another result.

The operation writes the low byte of sixteen times the slot index at absolute address `0x11`, then writes the low byte of the saved scale again at sixteen times the slot index plus `0x0D`.

The absolute write is not conditional in the visible path.

### 9. Payload-base fields

A five-call group, `F-d09-01` through `F-d09-05`, starts with the preceding result shifted left by 2 and produces a **record seed**.

Before using that seed, the operation writes:

1. The saved scale as a word at payload base plus `0x14`.
2. The low byte of twice the payload base at payload base plus `0x0D`.

The record seed determines:

- **Record base:** record seed multiplied by 32.
- **Record value:** record seed shifted left by 7.
- **Small selector:** `(record seed AND 0x07FFFFFF) >> 17`.

The small selector ranges from 0 through 1023.

### 10. Record fields and byte-directed selection

A three-call group, `F-d10-01` through `F-d10-03`, has no explicit arguments and produces a **shift source**.

The operation writes the record value at record base plus `0x14`, then the low byte of shift source shifted left by 2 at record base plus `0x0D`.

A five-call group, `F-d10-04` through `F-d10-08`, starts with shift source shifted left by 2. Its results are not explicitly consumed.

Next:

1. At small selector plus 7, write the low byte of quarter index shifted left by 1.
2. Read a byte at record base plus 1.
3. At that byte’s unsigned value plus `0x11`, write the low byte of the small selector.

This destination lies between `0x11` and `0x110`.

### 11. Clearing a freshly selected byte

`F-d11-01` receives shift source shifted left by 21 three times. `F-d11-02` produces a retained **transfer word**.

A five-call group, `F-d11-03` through `F-d11-07`, starts with the transfer word and produces a result.

The operation freshly reads the byte at record base plus 1, retains it as the **selector byte**, and writes zero at selector byte plus `0x11`.

This read occurs after the calls. It need not equal the earlier read.

### 12. Selector-based transfer

A five-call group, `F-d12-01` through `F-d12-05`, starts with the preceding result shifted left by 2.

The operation then:

- Retains twice the selector byte as the **selector base**, ranging from 0 through 510.
- Writes the low byte of transfer word shifted right by 3 at record base plus `0x0D`.
- Writes the selector byte at absolute address `0x0D`.

`F-d12-06` receives selector base, `0x98`, and `0x260`, in that order.

### 13. Secondary output and absolute replacement

`F-d13-01` has no explicit arguments. A secondary register output associated with it is logically shifted right by 3 and passed to `F-d13-02`.

A four-call group, `F-d13-03` through `F-d13-06`, produces a result. `F-d13-07` receives that result shifted left by 2 and itself produces a result.

The operation writes the low byte of transfer word shifted right by 3 at selector base plus `0x0D`, then writes `0xC0` at absolute address `0x0D`.

If selector base is zero, the former write also targets `0x0D`; the subsequent `0xC0` write still follows it.

### 14. Loaded-address propagation

A five-call group, `F-d14-01` through `F-d14-05`, starts with the result from `F-d13-07` shifted left by 2 and produces a **link value**.

The operation writes link value shifted left by 2 at selector base plus `0x24`, then reads a word at selector base plus `0x74` as the **loaded base**.

`F-d14-06` receives:

1. Loaded base.
2. Loaded base shifted left by 1.
3. Link value shifted left by 5.
4. `(loaded base AND 0x7FFFFFFF) >> 2`.

Four calls without explicit arguments, `F-d14-07` through `F-d14-10`, follow.

The operation writes link value shifted left by 5 at loaded base plus `0x24`.

### 15. Further double-width exchanges

A word is read at loaded base plus `0x74` and passed to `F-d15-01`. Four calls without explicit arguments follow; the last, `F-d15-05`, produces a double-width result.

Its low half shifted left by 2 becomes the **exchange value**. At four times its high half plus `0x14`, the operation writes the full word transfer word shifted right by 3—not merely its low byte.

`F-d15-06` receives the exchange value three times and the high half logically shifted right by 3.

`F-d15-07` produces another double-width result. `F-d15-08` receives its low half, its high half, its low half again, and its high half logically shifted right by 3.

### 16. Final indexed updates

A four-call group, `F-d16-01` through `F-d16-04`, produces an index. The exchange value is written at four times that index plus `0x14`.

A five-call group, `F-d16-05` through `F-d16-09`, starts with four times that index and produces a **final seed**.

The operation reads a byte at loaded base plus 1 and writes the low byte of final seed shifted left by 2 at the selected byte value plus `0x11`.

A three-call group, `F-d16-10` through `F-d16-12`, produces a result. The byte at loaded base plus 1 is read again. At the newly selected address plus `0x11`, the operation writes the low byte of `(final seed AND 0x3FFFFFFF) >> 20`.

### 17. Final clears and unresolved termination

A five-call group, `F-d17-01` through `F-d17-05`, starts with the preceding result shifted left by 2.

Define the **final scale** as final seed shifted left by 2. The operation reads a byte at final scale multiplied by `0x400` plus 1 and writes zero at that byte value plus `0x11`.

`F-d17-06` receives final scale shifted left by 3 twice, final scale itself, and the preceding group’s result shifted left by 2.

A four-call group, `F-d17-07` through `F-d17-10`, produces a result. The same computed byte-source address is read again, and zero is written to its newly selected destination.

A final five-call group, `F-d17-11` through `F-d17-15`, starts with the preceding result shifted left by 2. The visible representation then reaches the undecodable boundary.

## Side effects

Direct effects include byte and word writes to calculated addresses, explicit writes to absolute addresses `0x0D` and `0x11`, and byte-directed writes within `0x11`–`0x110`.

No direct allocation, release, ownership transfer, synchronization, rollback, or error reporting is established. Dependencies may perform any of these; their absence from the visible operations is not a guarantee of absence overall.

Writes may alias one another, dependency-owned storage, or subsequent read locations. The contract preserves access order rather than assuming separate records. A later failure may leave earlier writes visible.

## Dependencies

All `F-d…` dependencies are unresolved indirect calls. Their argument descriptions record only explicitly visible arguments. Return classifications record only consumed result bits.

The first target depends on incoming control state. Later targets are expressed through fixed address arithmetic in the supplied representation; this does not establish callable signatures or semantic identities.

No dependency is assumed to preserve global state, leave memory unchanged, allocate memory, release memory, or return normally. Secondary register outputs and double-width results require ABI confirmation.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests.

- **Initial byte narrowing:** If the initial word read is `0x123456AB`, the next direct byte write stores `0xAB`, not the full word.
- **Distinct field packings:** If the initial field is `0x200`, both visible packed values become zero under the 32-bit model: the nine-bit mask removes the bit in one case, and the left shift discards it in the other.
- **Word versus byte retention:** If the transfer word is `0x12345678`, byte transfers derived from its right shift store `0xCF`; the later word transfer stores `0x02468ACF`.
- **Fresh selector reads:** If a dependency changes the byte at record base plus 1 from 2 to 7 between stages 10 and 11, the later zero write targets `0x18`, not `0x13`.
- **Narrow destination range:** A selector byte of `0xFF` selects address `0x110`; it is not sign-extended.
- **Absolute-write ordering:** If the selector byte is zero, stage 13 writes its saved transfer byte at `0x0D` before overwriting that location with `0xC0`.
- **Repeated clears:** If dependencies change the final selector byte between the two final reads, both selected destinations are cleared in order.
- **Invalid access:** If a reached address is inaccessible, successful continuation is not promised. No visible recovery or rollback path exists.

## Uncertainties

**documented_only:**

- Intended application behavior and valid caller population.
- Correctness of recovered control flow, especially given the bad-instruction warning.
- Whether the final boundary represents a trap, malformed decoding, or missing continuation.
- Exact calling convention, hidden inputs, register preservation, and secondary-output provenance.
- Whether double-width results are genuine scalar returns or reconstructed register pairs.
- Signed overflow, signed left shifts, alignment rules, byte order, and exceptional memory behavior.
- Memory ownership, object extents, alias restrictions, concurrency, and dependency side effects.
- Any normal return, error status, or completion guarantee.

The 32-bit arithmetic model makes the visible transformations explicit; it does not resolve those uncertainties.

## Evidence

Established static evidence consists of the supplied single-function representation: one explicit argument, unresolved indirect calls, byte and word accesses, shifts and masks, secondary register outputs, double-width result extraction, and an undecodable terminal boundary.

Packed fields and recurring offsets suggest structured state manipulation, but that is inferred intent only. No object schema or domain meaning is established.

No implementation access, runtime execution, or runtime tests were available or performed. Acceptance examples are conditional consequences of the static representation, not observed results.
