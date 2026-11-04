<!-- Trusted provenance: F-c1ffc1efb8aa; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c1ffc1efb8aa

## Purpose

F-c1ffc1efb8aa conditionally clears a 32-bit payload-state value and performs a four-byte zero store. It then performs a separate four-byte store containing the upper nine bits of the current payload.

The two destination addresses are formed from **separate reads of recovered address-base state**. The supplied representation does not establish that the first store cannot change that state. Consequently, a common base, an eight-byte distance between destinations, and an untouched intervening gap are conditional properties, not unconditional guarantees.

No ordinary return is represented. After the second store, execution either reaches an invalid-instruction boundary directly or enters an overflow-dependent loop whose exit also reaches an invalid-instruction boundary.

This contract describes the supplied static representation. That representation reports bad instruction data and removal of unreachable material. It does not establish executable validity, a public calling convention, or application-level intent.

## Interface

### Incoming state

The displayed interface has no explicit arguments and no declared result. Its represented behavior nevertheless consumes the following execution state:

| Logical input | Interpretation |
|---|---|
| Control word | Unsigned 32-bit quantity used to decide whether clearing occurs. |
| Payload word | Unsigned 32-bit quantity, conditionally replaced with zero and subsequently shifted. |
| Address-base state | Recovered address-like state read separately when forming each store destination. Its declaration and architectural storage model are unavailable. |
| Overflow indication | Character-typed incoming condition, compared with zero. Its architectural origin and stability are not established. |
| Stack-relative anchor | Recovered stack reference used to construct a replacement address base on one branch. |

These descriptions do not establish argument registers, stack arguments, or a callable ABI.

Bit numbering starts at zero at the least significant bit. Shifts of the two unsigned words are logical shifts. The relevant shift distances are 21, 22, and 23 bits.

### Results and observable outcomes

No returned value is represented. The visible operations are:

- Conditional replacement of the payload-state value with zero.
- Conditional replacement of address-base state with a recovered stack-relative address.
- An optional four-byte zero store using an address-base read taken for that store.
- A subsequent four-byte extracted-value store using a fresh address-base read.
- A payload-field test followed by either an invalid-instruction boundary or an overflow-dependent loop.

The visibility of execution-state replacements to a real caller is unknown. No architectural register-preservation contract is established.

### Preconditions

The representation performs no destination validation. Successful completion of a selected store requires that its actual destination be writable and satisfy any target-specific alignment or addressing requirements.

For the second store, this requirement applies to the address formed from the **second address-base read**, not necessarily to an offset from the first read.

Other prerequisites include:

- Valid interpretation of the recovered integer conversion and address addition.
- Usable stack-relative storage when redirection is selected.
- Execution reaching the represented operations despite the reported instruction-data problems.

These are conditional execution requirements, not checks performed by the function. Fault handling is unspecified.

## Behavior

### Clearing decision

Clearing is selected if either:

- Control bit 21 is zero; or
- The control word shifted right by 22 is zero.

The second condition means that all control bits 22–31 are zero.

The payload is retained only when control bit 21 is one **and** at least one control bit in positions 22–31 is one. Bits 0–20 do not affect this decision.

The represented logical alternative short-circuits: when the bit-21 condition already selects clearing, the second condition need not be evaluated. No control-word write is shown.

### Conditional state replacement

When clearing is selected, the operations precede the first destination-address evaluation in this order:

1. Test the overflow indication for nonzero.
2. If nonzero, replace address-base state with the recovered stack-relative address.
3. Replace the payload-state value with zero.
4. Evaluate the destination and perform the optional zero store.

The replacement address is associated with displacement `0x08` from the recovered stack anchor. The anchor is not established as the architectural stack pointer at function entry, nor as a validated frame boundary.

If the overflow indication is zero, no explicit address-base replacement occurs in this block.

When clearing is not selected, the representation skips the overflow-dependent replacement, payload clearing, and optional zero store. A nonzero overflow indication alone therefore does not redirect the address.

### Optional zero store: first address-base read

On the clearing path, the function reads address-base state to form the first destination. Call the value obtained by this read **B₁**.

The address expression converts B₁ through a recovered integer conversion and adds `0x180`. Four zero bytes are stored at the resulting destination.

B₁ describes the value used to form this particular destination. It is not a promise that address-base state remains unchanged afterward.

The supplied material does not declare the recovered address-base state or establish its storage relationship to the destination. In particular, it does not rule out the first store affecting that state through aliasing or another feature of the actual storage model. Ordinary writable-memory and nonwrapping-address assumptions do not supply this missing non-aliasing guarantee.

### Extracted-value store: second address-base read

After the conditional block, the function evaluates address-base state again. Call this read value **B₂**.

It forms a destination by applying the recovered integer conversion to B₂ and adding `0x188`. It stores the current unsigned payload shifted right by 23 as a four-byte value.

Thus:

- The stored integer is in the inclusive range 0–511.
- Payload bits 23–31 contribute to the value.
- Payload bits 0–22 do not contribute.
- The destination write remains four bytes wide, despite containing at most nine significant bits.
- On the represented clearing path, payload clearing makes the extracted value zero.

If clearing was selected, B₂ is read **after** the optional zero store. Equality between B₁ and B₂ is not established.

If clearing was not selected, there is no first store or B₁ read for that store. The function still evaluates address-base state for the extracted-value store.

The representation specifies the order of the two stores but does not independently establish the ordering of address evaluation and payload evaluation within the second store expression.

### Conditional address relationships

If B₂ equals B₁ and the address conversion and additions behave as ordinary nonwrapping arithmetic:

- The first destination begins at B₁ plus `0x180`.
- The second destination begins at B₁ plus `0x188`.
- Their starting addresses are eight bytes apart.
- Each write covers four bytes.
- The four bytes between them are not targeted by either represented store.

Without base stability, none of those relative-layout conclusions follows merely from the two displacement constants. The second write may target a different region, overlap the first write, or target what would otherwise have been the intervening gap, subject to the actual address values.

For redirected execution, anchor-relative destinations of `0x188` and `0x190` likewise require the replacement base to survive until the corresponding reads, including across the first store, and require the stated arithmetic assumptions.

### Post-store control flow

After the extracted-value store, the representation evaluates the payload shifted right by 23 again for a nonzero test. This is a subsequent expression evaluation, not an explicit read-back of the destination memory.

A nonzero result leads directly to an invalid-instruction boundary.

A zero result enters a loop whose continuation condition tests the overflow indication:

- Zero selects another iteration.
- Nonzero permits loop exit.
- Loop exit leads to an invalid-instruction boundary.

The loop body contains no represented memory access, state update, or ordinary call. No operation in the loop is shown changing the overflow indication.

Under a literal interpretation with an unchanged overflow indication, zero causes indefinite looping and nonzero permits exit at the first condition check. Whether the recovered condition faithfully describes a stable machine condition is `documented_only`.

Neither invalid-instruction boundary establishes normal completion or a particular trap, signal, exception, or process-exit behavior.

## Side effects

### Memory accesses and ordering

The represented memory writes are:

| Operation | Address-base read | Displacement | Width | Value |
|---|---|---:|---:|---|
| Optional clearing store | B₁, evaluated on the clearing path | `0x180` | 4 bytes | Zero |
| Extracted-value store | B₂, evaluated after the conditional block | `0x188` | 4 bytes | Current payload shifted right by 23 |

There is no explicit read of existing destination contents.

Address-base state is nevertheless consumed separately for the two address expressions. This must not be collapsed into one cached base without establishing that the first write cannot affect the later read.

The observable ordering is:

1. Conditional address-state replacement.
2. Conditional payload clearing.
3. First address-base read and optional zero store.
4. Fresh address-base read and extracted-value store.
5. Subsequent payload-field test.
6. Terminal control path or loop.

The first three operations are absent where their conditions do not apply.

No transaction, rollback, lock, memory barrier, or atomicity guarantee is represented. If the first store faults, the second is not established to occur. If the second faults, no undo of a completed first store is shown.

Byte order is unspecified. Zero has identical byte contents under ordinary byte orders; nonzero extracted values require target byte-order information to determine individual bytes.

### Execution state

The clearing path explicitly sets the payload-state value to zero. With a nonzero overflow indication, it also explicitly replaces address-base state.

No explicit assignment changes the control word or overflow indication. This is not an architectural preservation guarantee.

The absence of an explicit later address-base assignment also does not prove stability: the relationship between the memory stores and recovered address-base storage remains unknown.

### Ownership and lifetime

No allocation, release, reference-count operation, or ownership transfer is represented.

Neither incoming addresses nor redirected stack-relative addresses are established as owned scratch storage. Stack relativity alone does not make the stores safe or grant permission to overwrite the destinations.

### Errors

No status result, error code, cleanup path, or local exception handler is represented.

Address faults and concrete invalid-instruction handling are outside the established behavior. The contract does not assume that an external handler preserves state, returns, or releases resources.

## Dependencies

No ordinary callable dependency with an independently identified opaque function ID is established.

The invalid-instruction termination markers indicate that the represented control flow ends at unsupported instruction data. They do not establish an external function with a known ABI.

Consequently, there is no supported dependency contract for return behavior, global-state preservation, memory ownership, or cleanup. The incoming execution state and stack-relative reference are environmental inputs, not identified callees.

## Acceptance examples

These are conditional static acceptance criteria, not runtime test results. Unless otherwise specified, assume usable destinations and ordinary nonwrapping address arithmetic. Examples using one base for both stores additionally require that the first store not change address-base state.

### Clearing with a stable incoming base

Given control word `0x00000000`, any payload, zero overflow indication, and address-base state B that remains stable across the first write:

- Clearing is selected.
- The payload becomes zero.
- The first read yields B, and four zero bytes are stored at B plus `0x180`.
- The second read also yields B, and four zero bytes are stored at B plus `0x188`.
- The subsequent extracted-field test is zero.
- A fixed zero overflow indication keeps execution in the loop.

Without the stability premise, the second destination is determined by its fresh base read.

### Bit 21 alone

With control word `0x00200000`, clearing is selected because bits 22–31 are all zero.

Even a nonzero incoming payload is cleared before extraction. This control value does not select payload retention.

### Upper control bit alone

With control word `0x00400000`, clearing is selected because bit 21 is zero.

Having a nonzero upper control field is insufficient without bit 21 also being set.

### Retained payload with a zero extracted field

Given control word `0x00600000`, payload `0x007FFFFF`, and zero overflow indication:

- Clearing is not selected.
- No store at displacement `0x180` occurs.
- No explicit address redirection occurs.
- The four-byte integer zero is stored at the address formed from the second-store base read plus `0x188`.
- Execution enters the loop.

A nonzero payload does not imply a nonzero extracted field.

### Smallest nonzero extracted field

Given control word `0x00600000` and payload `0x00800000`:

- Clearing is not selected.
- The extracted-value store writes the four-byte integer 1.
- Its destination uses the address-base value read for that store.
- The subsequent nonzero test leads directly to an invalid-instruction boundary, regardless of the overflow indication.

### Maximum extracted field

With retention selected and payload `0xFFFFFFFF`, the extracted-value store writes the four-byte integer 511.

It does not write the original payload or a one-byte truncation. The subsequent nonzero test leads directly to an invalid-instruction boundary.

### Stable stack-relative redirection

Given clearing selected, nonzero overflow indication, and replacement address-base state that remains equal to anchor plus `0x08` through both address reads:

- The first zero store targets anchor plus `0x188`.
- The second zero store targets anchor plus `0x190`.
- Under the fixed-condition interpretation, the loop exits at its first condition check and reaches an invalid-instruction boundary.

These addresses require base stability, not just writable stack-relative destinations.

### First store changes the later base read

Conditionally, suppose clearing is selected, the first address-base read yields B, and completing the first zero store causes the later address-base read to yield C rather than B.

Then:

- The first store targets the address formed from B plus `0x180`.
- The second store targets the address formed from C plus `0x188`.
- The second store writes zero on the represented clearing path.
- An eight-byte separation and an untouched four-byte gap cannot be required.

This example specifies the consequence if such a storage relationship exists. It does not assert that the supplied material proves that relationship realizable on the actual target.

## Uncertainties

The following remain `documented_only`:

- **Instruction validity:** Bad instruction data and omitted unreachable material are reported.
- **Calling convention:** Incoming execution state is consumed without explicit arguments.
- **Address-base storage:** Its declaration, architectural identity, and alias relationship with store destinations are unavailable.
- **Base stability:** The first store is not proven unable to affect the second address-base read.
- **Stack layout:** The recovered anchor is not tied to a validated entry-stack layout.
- **Address arithmetic:** The expressions include an integer conversion, but actual address width, truncation, signed-overflow behavior, and boundary wrapping are not validated.
- **Target memory rules:** Alignment, byte order, store atomicity, and valid address ranges are unspecified.
- **Condition fidelity:** The architectural meaning and stability of the overflow indication are unverified.
- **Terminal behavior:** Invalid-instruction markers do not determine runtime handling.

An intent to clear state and publish an extracted upper-bit field is inferred only. No application protocol, object layout, or synchronization purpose is established.

## Evidence

Established static evidence comprises:

- A clearing condition based on control bit 21 and control bits 22–31.
- Overflow-dependent address-base replacement within the clearing path.
- Payload clearing before an optional four-byte zero store.
- An address-base evaluation for that optional store.
- A separate address-base evaluation for the subsequent four-byte extracted-value store.
- Unsigned payload extraction by a 23-bit right shift.
- A subsequent extracted-field test selecting direct invalid-instruction termination or an overflow-dependent loop.
- No represented normal return, ordinary dependency call, allocation, or release.

The nine-bit result range follows from the 32-bit unsigned payload interpretation. Fixed relative store layout follows only with the additional base-stability and address-arithmetic premises.

No runtime tests have been performed. No concrete ABI, non-aliasing guarantee, executable validity, or platform trap behavior has been verified.
