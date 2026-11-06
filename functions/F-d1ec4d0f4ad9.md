<!-- Trusted provenance: F-d1ec4d0f4ad9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d1ec4d0f4ad9

## Purpose

F-d1ec4d0f4ad9 conditionally performs one signed halfword read followed by two word writes. Access to that memory-operation sequence requires an incoming condition flag and six successive comparisons to succeed. A failed gate skips the memory operations.

Every visible path that completes without an earlier access failure reaches a bad-instruction indication. No normal return or successful completion status is established. The supplied representation also reports truncated control flow, so this contract describes its visible behavior without asserting that it represents a valid executable function boundary.

An intended purpose of recognizing a machine-state pattern and writing associated values is plausible but inferred. No application-level meaning for the comparisons, addresses, or stored values is established.

## Interface

### Explicit inputs

The recovered interface has four unsigned word inputs:

| Contract name | Role |
|---|---|
| A | Supplies the original value and low-byte shift count for the first comparison. |
| B | Supplies the other operand of the first comparison and the value of the second store. |
| C | Supplies one operand of the second comparison. |
| D | Supplies the other operand of the second comparison. |

This contract uses the recovered 32-bit word model, with a signed 16-bit halfword for the memory read. These widths do not establish a complete ABI or portable language-level execution model.

There are no explicit pointer, length, ownership, or output-buffer arguments. Nevertheless, implicit machine state determines a source address, and the value read there determines both destination addresses.

### Implicit inputs

The visible behavior also depends on:

- An incoming zero condition flag.
- Four incoming unsigned word pairs, corresponding to architectural registers R4/R5, R6/R7, R8/R9, and R10/R11.
- An incoming link-register value, designated **L** here, used to form the halfword-read address.

These values are not initialized by the visible function. Its explicit argument list therefore does not fully describe the state required to reproduce its behavior.

Whether the implicit values are intentional calling-context inputs, surrounding-code state, or consequences of an inaccurate entry boundary is documented_only.

### Outputs and completion

There is no defined return value and no visible normal return.

The only explicit data-memory outputs are:

1. A 32-bit store containing a sign-extended halfword.
2. A subsequent 32-bit store containing B.

Both are conditional. Neither occurs unless all comparison gates succeed; an access failure may also prevent either or both stores from completing.

The terminal bad-instruction indication has no established runtime contract. An exception, termination, or other transfer of control must not be assumed solely from the supplied representation.

## Behavior

### Initial condition gate

The incoming zero flag determines whether comparison processing begins.

When the flag is clear, the function skips the first comparison, all later comparisons, the halfword read, and both stores. The visible path proceeds directly to the terminal bad-instruction indication.

No shift semantics or memory-validity assumptions are needed to characterize this rejection path.

### First comparison

When the incoming zero flag is set, the first comparison uses the **original** A in two roles:

- The shift count is A’s low eight bits, giving a count from 0 through 255.
- The shifted operand is the unsigned word A itself.

For counts from 0 through 31, the comparison succeeds exactly when B equals A logically right-shifted by that count. Equivalently, the exclusive-or comparison result must be zero.

A count of zero leaves A unchanged. Counts are not reduced to five bits. In particular:

- A equal to 0x00000101 produces count 1.
- A equal to 0x00000021 produces count 33, not count 1.

The supplied representation does not establish a result for a 32-bit shift by 32 through 255. Acceptance or rejection for such inputs is documented_only; this contract does not substitute a processor-specific shift rule.

On first-comparison success, the working comparison result is zero. Later shift counts come from that result, not from the original A.

### Remaining comparisons

Each later comparison is reached only after its predecessor succeeds. Consequently, the working result supplying its shift count is zero on every reached stage. The remaining comparisons therefore have these effective conditions:

| Order | Required condition |
|---|---|
| Second | C equals D. |
| Third | R4 equals R5. |
| Fourth | R6 equals R7. |
| Fifth | R8 equals R9. |
| Sixth | R10 equals R11. |

No oversized-shift rule is needed for these stages: every reached shift has count zero.

A mismatch at any stage prevents all later comparisons and all three explicit data-memory accesses. The sixth comparison is the last visible gate before the halfword read. No address, range, alignment, or permission validation follows it.

B remains the original second input throughout the visible sequence. The zero comparison result does not replace the value used by the second store.

### Signed halfword read

After all gates succeed, the function reads a signed 16-bit quantity from the address represented by **L + 13 bytes**.

Let **S** denote the resulting value after promotion to the recovered signed 32-bit word width. Its range is −32768 through 32767, inclusive. A halfword whose sign bit is set produces a negative S.

The source address is not explicitly checked. Its validity requires environmental facts absent from the supplied representation, including:

- The meaning and valid range of L.
- Whether forming L + 13 is valid in the applicable arithmetic model.
- Whether the resulting address is readable for two bytes.
- Whether the target permits any resulting unaligned access.
- The byte order used to decode the halfword.

The addition of 13 does not by itself establish either alignment or misalignment; that depends on L.

### First word store

After the read, the function stores S as a signed 32-bit word at the address represented by:

**S − 0x08574200**

The stored value is the sign extension of the halfword. For example, S equal to −1 yields the stored word 0xFFFFFFFF.

S determines both the stored value and the destination address. The destination is not relative to L or to the source address.

Under the recovered signed 32-bit model, this subtraction remains within the signed 32-bit range for every possible S. Signed overflow in this particular subtraction is therefore not an unresolved issue. What remains unresolved is whether converting the resulting negative integer to an address yields a usable destination.

If addresses are interpreted as corresponding 32-bit bit patterns, S equal to zero produces destination 0xF7A8BE00. This is a conditional address interpretation, not a guarantee that the location is mapped or writable.

### Second word store

After the first store, the function stores the original B as an unsigned 32-bit word at the address represented by:

**S − 0x085741FC**

This subtraction also remains within signed 32-bit range for all possible S.

The second destination is exactly four bytes greater than the first in the represented arithmetic. Under a compatible byte-addressed memory model, two valid four-byte stores therefore occupy adjacent, non-overlapping regions.

The second stored value depends on B, not on S or the final comparison result. Its destination depends on S.

### Terminal behavior

When the memory block completes, or when any comparison gate fails, the visible control flow reaches the bad-instruction indication.

This is not a returned validation error, a success result, or an established recovery path. The representation explicitly ends there with truncated control flow. Any continuation beyond that boundary is documented_only.

## Side effects

On a fully matching path, the explicit data-memory access order is:

1. Read two bytes as a signed halfword at the L-derived address.
2. Store the sign-extended value as a four-byte word at the first destination.
3. Store B as a four-byte word at the second destination.
4. Reach the terminal indication, if execution has not already been interrupted.

On a rejected path, none of those three explicit data-memory accesses occurs. This statement does not characterize instruction fetches, exception machinery, or effects of the terminal indication.

The sequence is not transactional. No rollback or recovery mechanism is visible. If the first store completes and a later operation fails, the representation provides no mechanism to undo that completed store.

The source region may overlap a destination region; no non-aliasing condition is shown. The halfword is read before either store, so S has already been obtained before a destination write can modify overlapping source bytes.

No allocation, deallocation, ownership transfer, resource release, synchronization, or explicit global-state update is shown apart from the memory stores. The function does not establish ownership of the source or destination memory.

The comparison sequence updates a working value in the recovered representation. That does not establish a complete outgoing register or condition-flag state. Unmentioned machine state must not be assumed preserved.

The terminal indication has uncharacterized effects. It must not be assumed to preserve globals, release memory, or permit execution to resume.

## Dependencies

The supplied dependency mapping identifies only F-d1ec4d0f4ad9. No separately identified ordinary callee appears in the visible behavior.

The bad-instruction indication has no supplied opaque dependency ID or characterized callable interface. It is treated as an unresolved terminal operation, not as a service with assumed preservation or ownership guarantees.

Environmental dependencies include:

- Interpretation of incoming register and condition-flag state.
- First-stage shift semantics for counts from 32 through 255.
- Halfword byte order and signed decoding.
- Validity of source-address formation.
- Integer-to-address conversion and the address-space model.
- Alignment, accessibility, and permissions for each memory access.
- Handling of the terminal bad-instruction indication.

These dependencies do not establish ownership or preservation of memory or global state.

## Acceptance examples

These are conditional expectations derived from static evidence, not observed executions. Examples using memory accesses assume the recovered widths, valid address interpretation, and successful preceding accesses unless a failure is explicitly discussed.

### Incoming flag rejects the state

For any explicit arguments and implicit register values, an incoming zero flag that is clear skips all comparisons and the three explicit data-memory accesses. The visible path reaches the terminal indication.

### Defined-count first-comparison failure

With the incoming zero flag set, A equal to zero, and B equal to one:

- The shift count is zero.
- The shifted value is zero.
- The first comparison fails.
- The later comparisons, halfword read, and stores are skipped.

### Oversized counts remain unresolved

With the incoming zero flag set and A equal to 0x00000021, the shift count is 33. B equal to 0x00000010 is **not** established as a matching value.

Likewise, A equal to 0x00000020 gives count 32. Neither example establishes whether later comparisons or memory accesses occur. Both remain documented_only rather than being evaluated using an assumed count mask.

### A later mismatch rejects the state

With the incoming zero flag set, A and B both zero, and C unequal to D, the first comparison succeeds and the second fails. No register-pair comparison or conditional data-memory access follows.

A mismatch in any later pair has the same memory-access consequence when all preceding gates succeed.

### Full match with S equal to zero

Assume:

- The incoming zero flag is set.
- A and B are both zero.
- C equals D and all four register pairs are equal.
- The halfword read succeeds and produces S equal to zero.
- Both destination accesses are valid under the corresponding 32-bit address-bit-pattern interpretation.

The first store writes zero to 0xF7A8BE00. The second writes B, also zero, to 0xF7A8BE04. The visible path then reaches the terminal indication, not a normal successful return.

### Full match with a nonzero second argument

With the incoming zero flag set, A equal to **0x00000101**, and B equal to **0x00000080**:

- A’s low byte is 0x01, so the shift count is 1.
- The unsigned right-shifted value is 0x00000080.
- The first comparison succeeds.

If C equals D, every register pair is equal, and the memory accesses succeed, the second store writes **0x00000080**. S determines its destination but does not change that stored value.

### Negative halfword

On a fully matching path, a halfword decoded as −1 produces:

- First-store data equal to 0xFFFFFFFF.
- A first destination one byte below the destination for S equal to zero.
- A second destination four bytes after the first.

Under the corresponding 32-bit address-bit-pattern interpretation, the destinations are 0xF7A8BDFF and 0xF7A8BE03. Whether these unaligned word accesses are supported and writable remains conditional.

### Failure after partial completion

If the read and first store complete but the second store faults, no visible rollback is provided for the first store. Reaching the terminal indication after that fault is not established.

## Uncertainties

The following remain documented_only:

- Whether the recovered entry point is a valid callable boundary.
- Whether the represented comparison chain corresponds to executable instructions.
- First-stage shift behavior for counts of 32 or greater.
- The complete ABI, underlying type definitions, byte order, and address-space model.
- Validity of L and of forming L + 13.
- Conversion of the negative destination integers into usable addresses.
- Access permissions, alignment support, and fault behavior.
- Intended meaning of the implicit register values and the signed halfword.
- Ownership of every touched memory region.
- Concrete effects of the bad-instruction indication.
- Any behavior beyond the truncated control-flow boundary.

No safe fallback, error code, successful return, preservation guarantee, or memory-safety guarantee is established.

## Evidence

Established static evidence is the supplied recovered representation: four explicit unsigned inputs, an incoming zero-flag gate, eight additional incoming register values, six gated comparisons, a link-register-relative signed halfword read, two ordered word stores, and a terminal bad-instruction indication.

The later comparisons reduce to equality because each reached shift count comes from a preceding zero result. The corrected nonzero-input example follows from the low-eight-bit count: 0x00000101 supplies count 1 and shifts to 0x00000080; 0x00000021 supplies count 33 and has no established shift result here.

The signed range and stored sign extension follow from the halfword interpretation and promotion. The destination subtractions are within signed 32-bit range; address conversion and memory validity remain separate uncertainties.

Application-level intent is inferred only. The invalid-instruction and truncation warnings limit conclusions about executable behavior.

No runtime tests have been performed.
