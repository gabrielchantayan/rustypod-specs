<!-- Trusted provenance: F-1aaa6714ffa6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1aaa6714ffa6

## Purpose

F-1aaa6714ffa6 modifies memory through a mixture of fixed numeric destinations, input-derived addresses, memory-selected addresses, and shared storage. A sign check separates an early termination path from a longer continuation. Neither path establishes a normal return.

The application-level purpose is unknown. Associations with reset, instruction, interrupt, or prefetch state do not establish that the function initializes a processor, configures exceptions, or performs recovery.

This is a **conditional static contract**, not a claim that the supplied representation executes under ordinary language semantics. In particular, the representation applies right shifts of 32 to signed 32-bit values and contains potentially invalid signed arithmetic. The access requirements below apply only under an execution model that defines the relevant operations and reaches those accesses.

## Interface

### Inputs

| Input | Representation | Role |
|---|---|---|
| First explicit argument | Opaque 32-bit value | No visible use |
| Selector | Unsigned 32-bit integer | Determines values, address calculations, and an address-selection region |
| Third explicit argument | Opaque 32-bit value | No visible use |
| Origin | Signed 32-bit integer | Determines an initial byte destination and a derived base |
| Additional incoming pointer | Pointer to signed 32-bit storage | Used as an address, converted to a number, and itself stored |

The additional pointer is associated with an incoming slot represented at offset 0x250. The slot’s address participates independently in an address calculation. This does not establish the offset’s reference point, a calling convention, or a physical stack layout.

Other inputs are the memory contents observed during execution and numeric references to F-e47d7b2e9c88 and F-8afdaab91693.

### Outputs, preconditions, and ownership

There is no established return value or ordinary return path. Observable results are memory effects completed before an early or terminal control-flow boundary.

Every access requires appropriate mapping, permissions, alignment, extent, and lifetime. The function does not visibly validate these conditions. Numeric address zero is not excluded. There is no visible error result, rollback, recovery action, allocation, deallocation, or ownership transfer.

Explicit byte, halfword, and integer accesses have widths of 8, 16, and 32 bits. A pointer-valued store has the width of the unresolved pointer representation. Assigning a value of a particular width to a shared object does not, by itself, establish that object’s physical access width.

## Behavior

### Numeric interpretation

The selector is reduced in two stages: subtract 0xA8 using unsigned 32-bit arithmetic, then clear bit 31. The resulting **masked selector** is between zero and 0x7FFFFFFF. Its unsigned half is the **reduced selector**, between zero and 0x3FFFFFFF.

Doubling the subtraction result modulo 2³² produces the numeric address of the **selection region**. Offsets within that region refer to byte distances; they do not independently imply wrapping pointer arithmetic.

Two bases require an interpretation beyond ordinary valid 32-bit signed operations:

- The **origin base** results from shifting the signed origin left by two and then right by 32.
- The **reference base** results from converting a reference to F-e47d7b2e9c88 to signed 32-bit form and shifting it right by 32.

Neither base is established as zero or as a sign mask. A machine-specific masked shift interpretation must not be assumed.

Unsigned 32-bit arithmetic wraps modulo 2³². That rule does not settle signed overflow, signed left shifts, right shifts of negative values, or pointer conversions. All byte offsets below are conditional on those operations having defined meanings.

### Selection-region preparation and destination selection

The initial effects prepare several potentially overlapping locations before observing the address-selecting halfword.

The required order across these preparations is:

- An 8-bit store at origin plus 0x0897E8B8 carries the reduced selector’s low byte.
- Shared storage at 0x0897E8C0 receives 0xB8, followed by a character store at selector plus six carrying the selector converted to character form.
- A 32-bit store at the origin base carries the original selector.
- The selection region receives 0xFFFF through 16-bit stores at offsets four and six.
- An 8-bit store at origin-base offset four carries 0xB8, before another 16-bit 0xFFFF store at selection-region offset two.
- Shared storage at 0x0897EA28 receives the low 16 bits extracted from the represented selection-region address.

Only after these preparations is the unsigned 16-bit value at selection-region offset zero observed. Its zero-extended value becomes a numeric destination for a 32-bit store. The payload is the original selector masked with 0x897E8B80.

The selected destination therefore lies between zero and 65535. It is not an entry-time snapshot: preceding overlapping stores may determine its value. The selector-plus-six calculation uses unsigned 32-bit addition; the origin-based addition has no corresponding established wrapping rule.

### Sign gate and sample retention

The representation separately obtains a retained 32-bit reference to F-e47d7b2e9c88, the reference base, and a 16-bit sign field derived from another signed reference conversion shifted right by 31.

The sign field is stored at reference-base offset four before the retained reference is tested as signed 32-bit. A negative retained reference reaches the early termination boundary. None of the continuation effects described below belongs to that branch.

On the nonnegative path, a signed 16-bit sample is retained from reference base plus reduced selector. The address addition uses unsigned 32-bit arithmetic because the reduced selector is unsigned.

Two identical write pairs bracket this sample observation. Each pair stores the sign field at reference-base offset 0x12 and then stores the retained reference, through a 32-bit access, at the numeric address equal to that reference. Thus the first pair can affect the sample; the second can change its storage without changing the retained sample.

After the second pair, reference-base offset eight receives a 32-bit result formed by sign-extending the retained sample and shifting it right by 16. Under arithmetic right shift, this result is zero for a nonnegative sample and 0xFFFFFFFF for a negative sample.

Under arithmetic right shift, the sign field is likewise 0x0000 or 0xFFFF. Agreement between that field and the retained reference’s sign additionally depends on consistent interpretation of the separately expressed references.

### Configuration and incoming-pointer effects

The continuation first assigns zero to prefetch-associated shared storage. Bit two of the retained F-e47d7b2e9c88 reference selects these alternatives:

| Bit two | Configuration value | Destination receiving the reference base | Destination receiving the reduced selector |
|---|---:|---:|---:|
| Clear | 0x58 | 0x5C | 0x170 |
| Set | 0x5C | 0x60 | 0x180 |

Both listed destination stores are 32-bit. Between them, shared storage associated with numeric address six receives the masked selector shifted right by 17 and converted to unsigned 16-bit form. Instruction-associated composite storage then receives the reduced selector’s low byte, zero-extended to 16 bits. Supervisor-call-associated storage receives the reduced selector after the second destination store.

The subsequent pointer-target effects have a significant overwrite order:

- The configuration value is stored through a 16-bit access at the incoming pointer’s target.
- The reference base shifted left by eight is stored through 32-bit accesses at the target and at byte offset 20.
- The pointer’s signed 32-bit numeric conversion is then stored through a 32-bit access at the target.

A separate 32-bit observation comes from the numeric address equal to twice the pointer’s signed 32-bit conversion. Its narrowed 16-bit representation is stored at the pointer target. The **full retained observation**, rather than this narrowed representation, subsequently supplies both destination and payload for a 32-bit self-address store.

Two later stores distinguish the incoming slot’s address from the pointer value:

| Address basis | Destination calculation | Payload and width |
|---|---|---|
| Incoming slot’s address | Thirty-two times its signed 32-bit conversion, plus 0x18 | Pointer’s signed conversion shifted right by 31; 32-bit store |
| Incoming pointer | Four times its signed 32-bit conversion, plus 0x18 | Incoming pointer; pointer-representation-width store |

With arithmetic right shift, the first payload is zero or 0xFFFFFFFF. The signed products and additions are not established as wrapping.

Between these two stores, an instruction-associated byte component is freshly observed. Instruction-associated composite storage receives a value consisting of that byte in bits 0–7 and the 16-bit value five in bits 8–23. Its numeric value is 0x000500 plus the observed byte. This establishes field placement, not the receiving object’s physical width.

The representation does not resolve whether stores overlapping the incoming slot alter subsequent evaluations of the incoming pointer. Neither universal reloading nor universal immunity to such aliasing is assumed.

### Retained snapshot and mutable destination state

A signed 32-bit observation from shared storage at 0x0897EA50 becomes a retained **snapshot**. A reference to F-8afdaab91693 is retained afterward for terminal addressing.

Shared storage at 0x0897EA80 then receives four times the incoming pointer’s signed 32-bit conversion. A separately expressed F-8afdaab91693 reference, shifted left by two, becomes another retained payload.

The two shared-state roles must remain distinct:

- The snapshot from 0x0897EA50 is not refreshed.
- The destination state at 0x0897EA80 is observed again when used.

A 32-bit self-address store takes its destination and payload from observations of 0x0897EA80. The relative order of those operand observations within that expression is unspecified. The following 32-bit store places the reduced selector at a destination selected by a later observation of 0x0897EA80 plus four. An overlapping self-address store may therefore redirect the later store.

The snapshot controls two subsequent 32-bit stores. Four times the snapshot supplies both destination and payload for the first. For the second, the destination is the snapshot plus a displacement formed by shifting the snapshot left by six and then right by 17. Its payload is the retained, left-shifted reference to F-8afdaab91693.

These signed calculations require a defined execution model. Replacing them with unsigned wraparound is not justified by the unsigned selector calculations.

### Terminal effects and completion

The snapshot’s unsigned upper 16 bits are assigned to a reset-associated component. Interrupt-associated shared storage then receives zero.

Eight times the earlier retained F-8afdaab91693 reference supplies a terminal base. Two 32-bit zero stores occur at offsets zero and four, followed by a character store at offset 16.

The character payload comes from converting the terminal base to character form, adding eight after integer promotion, and converting again for storage. Under low-eight-bit conversion semantics, the stored byte is the terminal base’s low byte plus eight modulo 256.

These shared assignments follow in the listed order:

| Destination | Assigned value |
|---|---|
| 0x00000086 | 0x897EA700 |
| 0xEA700000 | 0x66 |
| 0x0897EB26 | 0xEAEC |
| 0x12FD4C34 | 0x1A |
| 0x897EA65C | Pointer to numeric address 0x897EA65C |

Their physical access widths are unresolved.

A fresh observation of the reset-associated 16-bit component then supplies the upper half of a composite whose lower half is 0xEA70. The composite is assigned to reset-associated composite storage. Finally, shared storage at 0x12FD4CB8 receives 0x5C and execution reaches the terminal boundary.

The fresh component observation cannot unconditionally be replaced by the snapshot’s upper half: intervening overlapping stores may change it. Reaching the final assignment is not evidence of a success return.

## Side effects

No disjointness is established among input-derived destinations, fixed-address storage, shared components, reference-derived destinations, the incoming pointer target, or its incoming slot.

The contract therefore distinguishes retained observations from later reads. The signed sample, full-width redirection observation, snapshot, and retained dependency references preserve their captured roles. The selection halfword, mutable destination state, and composite components are observed at their specified later positions.

Repeated stores remain separate ordered effects even when a later store overwrites the same bytes. Equivalent final memory alone is insufficient to establish equivalent represented behavior.

No atomicity, synchronization, cross-thread visibility, device protocol, or compiler-level observability guarantee is established. Failure may leave earlier effects in place; partial-access behavior is unknown. There is no established ownership change or lifetime extension.

## Dependencies

| Opaque function ID | Established relationship |
|---|---|
| F-e47d7b2e9c88 | Numeric references contribute to sign handling, reference-base derivation, configuration, and self-address stores |
| F-8afdaab91693 | Numeric references contribute a shifted payload and terminal addressing |

Neither dependency is visibly invoked. Consequently, the contract does not require or assume either dependency to preserve globals, allocate storage, release memory, or transfer ownership.

The separately expressed references are not silently merged into a single observation. Their numeric interpretation and stability across expressions remain **documented_only** where the supplied representation does not settle them.

No opaque dependency identity is supplied for either termination boundary. A normal return, particular processor exception, or specific diagnostic is not established.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Each assumes that preceding accesses complete and that all relevant arithmetic, conversions, and addresses have defined interpretations.

1. **Reduction at the subtraction boundary.** Selector 0xA8 gives zero for the subtraction result, masked selector, reduced selector, and selection-region address. Address selection observes a halfword at numeric address zero. Introducing a null-address bypass would change this behavior.

2. **Unsigned underflow without invented pointer wrapping.** Selector 0xA7 gives subtraction result 0xFFFFFFFF, masked selector 0x7FFFFFFF, reduced selector 0x3FFFFFFF, and selection-region address 0xFFFFFFFE. This does not establish how subsequent pointer offsets behave.

3. **Unsigned address selection.** A selection halfword of 0xFEDC chooses destination 0x0000FEDC, not a sign-extended destination. The stored payload remains the selector masked with 0x897E8B80.

4. **Early boundary.** A negative retained F-e47d7b2e9c88 reference reaches the early boundary after the sign-field store at reference-base offset four. The bracketing pairs, configuration effects, pointer effects, and terminal effects are absent.

5. **Sample capture survives overwrite.** Under arithmetic right shift, a retained sample of −1 causes 0xFFFFFFFF to be stored at reference-base offset eight even if the second bracketing pair changes the original sample bytes.

6. **Configuration alternatives.** Clear bit two selects configuration 0x58 and destinations 0x5C and 0x170. Set bit two selects 0x5C and destinations 0x60 and 0x180. The two intervening shared assignments remain between those destination stores.

7. **Full-width redirection survives narrowing.** A captured observation of 0x12345678 contributes 0x5678 to the pointer-target halfword under low-halfword conversion semantics. Its subsequent self-address store still uses destination and payload 0x12345678.

8. **Composite field placement.** An observed instruction-associated byte of 0x34 gives composite value 0x000534. An observed reset component of 0x1234 gives 0x1234EA70. Neither result establishes physical assignment width.

9. **Mutable destination differs from snapshot.** If an overlapping store changes 0x0897EA80, the reduced-selector destination uses the later contents plus four. The separate snapshot from 0x0897EA50 remains unchanged.

10. **Terminal character conversion.** If terminal addressing is defined and the terminal base’s low byte is 0xFC, low-eight-bit conversion semantics give stored byte 0x04 at offset 16.

11. **Fresh reset-component observation.** If an intervening overlapping effect changes the reset component after the snapshot-derived assignment, the final reset composite uses that changed component, not an unconditionally cached snapshot field.

## Uncertainties

The following remain **documented_only**:

- Application purpose, executable input domain, and the runtime meanings of both termination boundaries.
- Interpretation of signed 32-bit shifts by 32.
- Signed overflow, signed-left-shift validity, and negative-value right shifts.
- Character signedness, narrowing behavior, pointer width, and pointer/integer conversions.
- Incoming-slot layout, offset origin, and calling convention.
- Pointer provenance, mappings, accessibility, alignment, and lifetime.
- Shared-object declarations, physical access widths, component overlap, and byte order.
- Slot aliasing effects on later incoming-pointer evaluations.
- Operand-observation ordering within individual expressions.
- Numeric interpretation and stability of separately expressed dependency references.
- Concurrency, device semantics, partial-access behavior, and failure handling.

These unknowns are not resolved by assuming disjoint storage, valid addresses, uniform wrapping, immutable references, or benign external behavior.

## Evidence

**Established static evidence:** The supplied representation supports the input widths, unsigned selector reduction, explicit access widths, delayed unsigned-halfword address selection, sign-dependent early boundary, bracketed sample capture, full-width retained redirection, separate snapshot and mutable destination state, fresh component observations, and terminal boundary described above.

**Representation limits:** The supplied material reports invalid instruction data and overlapping shared symbols. Those reports constrain interpretation; they do not prove a particular hardware fault, complete memory layout, or executable instruction sequence.

**Inferred intent:** Low-level memory and shared-state manipulation is plausible. Initialization, exception configuration, recovery, and successful completion are not established.

**Runtime evidence:** No runtime tests have been performed.
