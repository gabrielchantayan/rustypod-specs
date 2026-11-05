<!-- Trusted provenance: F-b6981cd60aac; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b6981cd60aac

## Purpose

F-b6981cd60aac obtains a destination address, derives an inspection address through an indirect invocation, and performs two ordered classification calls. Their results select rejection, a two-byte processing dependency, or an eligibility check followed by a one-byte processing dependency.

On an accepted path, the function transfers two 16-bit portions from a mutable four-byte result slot into the destination, then calls a completion dependency. The transfer is sequential: the first destination write precedes the read of the result slot’s upper portion. Destination/result-slot overlap can therefore change the second transferred value.

The result slot initially contains the third input’s pointer representation. That initialization does not guarantee its value at transfer time. Earlier dependencies may affect it through aliases, the selected processor receives its address directly, and overlapping destination writes may subsequently modify it.

“Destination,” “classification,” “processing,” “completion,” and “rejection” describe inferred roles. Allocation, validation, resource release, and application-level meanings are not established.

## Interface

The function takes four positional inputs and has no explicit return value.

| Input | Established representation | Explicit use |
|---|---|---|
| First | Opaque 32-bit value | First argument to F-603a53835f04. |
| Second | Opaque 32-bit value | Argument to F-603a53835f04 and the selected processor. |
| Third | Object-like pointer in the represented 32-bit address model | Initializes the result slot, supplies the indirect-call receiver, and is passed to the selected processor. |
| Fourth | 32-bit integer | Initializes the address slot, which is subsequently overwritten by the indirect-call result. |

Two mutable local storage locations participate in the behavior:

- **Result slot:** four bytes, initialized with the third input’s pointer representation. Its address is passed to the selected processor and later to F-9b718979506d.
- **Address slot:** four bytes, initialized with the fourth input and then replaced by the indirect-call result. Both classification calls receive its address.

F-603a53835f04 returns the retained destination address. On an accepted path, this function writes 16 bits at destination byte offset zero and 16 bits at byte offset two. On rejection, it passes the destination to F-7f008c1060db.

Changing the result slot does not itself replace the third-input value used as the receiver or explicit processing argument.

### Preconditions for ordinary execution

The function does not explicitly check that:

- The third input permits reading its dispatch-table pointer.
- That table permits reading a callable entry at byte offset eight.
- The entry is callable with the third input as receiver.
- The final address-slot value permits the byte reads selected by the branch.
- The destination permits both 16-bit writes on an accepted path.
- Address arithmetic, alignment, and lifetimes satisfy the execution model.
- Each dependency’s own requirements are satisfied.

There are no explicit null, capacity, alignment, or address-overflow checks. Invalid address calculations are not established to have defined wrapping behavior.

Destination/result-slot disjointness is not established as a precondition. Overlap examples remain conditional on every actual access being valid, including accesses outside the result slot’s own four-byte extent.

## Behavior

### Initialization and destination acquisition

Before the first dependency call, the function initializes the result slot from the third input and the address slot from the fourth input.

It calls F-603a53835f04 with five ordered arguments:

1. First input.
2. Zero.
3. Zero.
4. Zero.
5. Second input.

The returned address is retained as the destination without an explicit validity check.

This call precedes the indirect-method lookup. Changes it makes to memory reachable through the third input can therefore affect that lookup.

Neither this call nor any other dependency has an established general preservation contract. The result slot’s initialization must not be treated as proof that its contents remain unchanged until processing. If a dependency can reach the slot through an alias, its effects may alter those contents before the processor is entered.

### Indirect invocation

After destination acquisition returns, the function:

1. Reads a dispatch-table pointer through the third input.
2. Reads a callable entry at byte offset eight within that table.
3. Invokes that entry with the third input as receiver.
4. Stores its return value into the address slot.

No opaque identity is supplied for this indirect dependency.

The fourth input’s stored value is overwritten before the classification calls. It is not the explicit inspection base used later.

### Ordered classification

F-c9c29fb7e6f0 is called twice:

1. With the address-slot address, one, and one.
2. With the same slot address, one, and zero.

Both calls precede branch selection. A nonzero first result does not suppress the second call.

Each result is treated as an unsigned 32-bit quantity. The first call can modify the address slot before the second call receives it; the second can modify it again. Later explicit inspection reads use the slot’s resulting value, not necessarily the indirect-call return.

| First classification | Second classification | Selected behavior |
|---|---|---|
| Nonzero | Any value | Reject without this function’s inspection-byte reads. |
| Zero | Nonzero | Invoke the two-byte processor. |
| Zero | Zero | Read the eligibility byte; reject or invoke the one-byte processor. |

The condition also checks whether the classifications have a zero bitwise intersection. This does not change the accepted cases: a zero first classification necessarily has zero intersection, while a nonzero first classification already fails the separate zero requirement.

No restricted result range is established. A classification containing only the highest bit is nonzero and has the same branch significance as any other nonzero value.

### Two-byte processing

When the first classification is zero and the second is nonzero, the function obtains the bytes at offsets two and four from the current inspection address.

F-5a2673aeb07d receives:

1. Result-slot address.
2. Second input.
3. Third input.
4. Offset-two byte.
5. Offset-four byte.

Both byte values are obtained before processor entry. Their relative evaluation order is not established by the supplied expression.

The offset-two byte is not subject to an eligibility test on this branch. Every eight-bit pattern is permitted as a byte argument.

If the processor returns normally, execution proceeds to result transfer. Its return value is not explicitly consumed. Its effects on the result slot or other reachable state are not otherwise specified.

### Zero-classification eligibility

When both classifications are zero, the function first reads the byte at inspection offset two.

- A nonzero value selects rejection.
- A zero value permits the offset-four byte read and the one-byte processing call.

The zero test applies to the complete byte. Signedness does not change this zero/nonzero decision.

On eligibility success, F-d9951cfd4fb4 receives:

1. Result-slot address.
2. Second input.
3. Third input.
4. Offset-four byte.

The eligibility read precedes the offset-four read. Failed eligibility prevents this function’s offset-four read and prevents either processor from being called.

If F-d9951cfd4fb4 returns normally, execution proceeds to result transfer. Its return value is not explicitly consumed.

### Sequential result transfer

After either processor returns normally, the function performs these ordered actions:

1. Obtains the low 16 bits of the result slot’s current value.
2. Writes those 16 bits to destination offset zero.
3. Reads the two-byte portion at result-slot offset two.
4. Writes that portion to destination offset two.
5. Calls F-9b718979506d with the result-slot address.

The evidence establishes these transfer portions and their ordering, not a particular machine-instruction width for obtaining the first portion.

There is no established four-byte snapshot before the first destination write. The upper-portion read occurs after that write.

Under a little-endian interpretation, if no overlap or other interference changes the relevant result-slot bytes during transfer, the two writes reproduce the four-byte representation present when transfer begins. This is a conditional consequence, not an unconditional copy guarantee.

If the first destination write overlaps the result slot’s upper two bytes, it can change the second transferred portion. The second destination write can also alter the slot after the upper portion has been read.

The result slot’s transfer-time contents depend on its history:

- If the processor leaves the slot unchanged, transfer starts from the slot contents present when the processor was entered, subject to any other intervening interference.
- Transfer starts from the originally stored third-input representation only if no earlier dependency or other interference changed that representation, the processor leaves it unchanged, and it remains unchanged through the beginning of transfer.
- A processor leaving the slot unchanged does not establish preservation by destination acquisition, indirect invocation, or classification calls.

For example, conditional on the relevant alias being available, destination acquisition could retain an alias to the result slot and a classification call could modify the slot through that alias. An unchanged slot during processing would then retain the earlier modification, not restore the third-input representation.

The writes are separate 16-bit operations. No atomic four-byte publication or synchronization is established.

F-9b718979506d receives the slot after both destination writes, including any overlap-induced modifications. It need not receive the exact representation left by the processor.

If completion returns normally, this function returns without another explicit destination write. Completion may nevertheless modify the destination through its own behavior or aliases.

### Rejection

Rejection occurs when:

- The first classification is nonzero; or
- Both classifications are zero and the offset-two eligibility byte is nonzero.

F-7f008c1060db receives:

1. Retained destination address.
2. Two.
3. Five.
4. 9216, equivalently 0x2400.

If that dependency returns normally, this function returns.

On rejection, this function does not call either processor, perform the two result-transfer writes, or call F-9b718979506d. This does not rule out memory changes by earlier dependencies or the rejection dependency.

## Side effects

The directly established interactions are:

- Initialization of two four-byte local slots.
- Destination acquisition.
- Dispatch-table and callable-entry reads.
- Indirect invocation.
- Two ordered classification calls exposing a mutable address slot.
- Conditional inspection-byte reads.
- Exactly one processing call on an accepted path.
- Ordered result-slot reads and destination writes.
- Completion after both writes, or rejection instead.

No direct global-variable access is shown. Dependency preservation of globals or aliased memory is not established.

### Aliasing and ordering

The result slot is directly exposed to the processor and completion. The supplied evidence does not establish that earlier dependencies are unable to reach it through other aliases.

Consequently, all claims about its retained initial value require preservation throughout the preceding execution, not merely an unchanged slot during the selected processor.

Destination/result-slot overlap is not excluded. The first destination write may affect the later upper-portion read, and either destination write may affect the slot subsequently presented to completion.

Inspection-byte arguments are obtained before processor entry. Processor changes to inspection memory do not retroactively change those already-obtained values.

The destination contents produced by the explicit writes are not guaranteed to persist after completion.

### Ownership and lifetime

No allocation, retention, ownership-transfer, or release contract is established for:

- The destination.
- The third input.
- The indirect-call result.
- Addresses installed in the address slot.
- Representations installed in the result slot.

F-9b718979506d must not be assumed to free, release, retain, or merely inspect the result.

Exposing a local-slot address during a call does not establish permission to use it after this function’s activation ends.

### Failure and partial progress

There is no explicit rollback, recovery, or returned error status.

If a dependency does not return normally, later operations are not established to occur. The same limitation applies when a required memory access cannot complete.

Possible partial progress includes:

- Acquisition and both classifications before rejection.
- Earlier alias-mediated slot modifications before processing.
- Processor effects before any destination write.
- The first destination write before failure of the upper-portion read or second write.
- Both destination writes before completion fails.

In an overlapping case, the first write may already have changed the result slot even if transfer does not complete.

## Dependencies

| Opaque function ID | Established interaction |
|---|---|
| F-603a53835f04 | Called before indirect lookup with the first input, three zeros, and the second input; returns the retained destination. |
| F-c9c29fb7e6f0 | Called twice with the address-slot address; trailing arguments are one and one, then one and zero; returns 32-bit classifications. |
| F-5a2673aeb07d | Selected for zero first classification and nonzero second classification; receives the result-slot address, second and third inputs, and two inspection bytes. |
| F-d9951cfd4fb4 | Selected when both classifications and the eligibility byte are zero; receives the result-slot address, second and third inputs, and the offset-four byte. |
| F-9b718979506d | Receives the result-slot address after both destination writes. |
| F-7f008c1060db | Receives the destination, two, five, and 9216 on rejection. |

An additional dependency is selected indirectly through dispatch-table byte offset eight. Its identity is unavailable.

These interactions do not establish dependency internals, normal-return guarantees, global preservation, alias restrictions, ownership, or cleanup semantics.

## Acceptance examples

These are conditional static criteria, not executed tests. Unless stated otherwise, assume valid accesses and normally returning dependencies.

1. **First classification rejects.**  
   For classifications one and zero, both classification calls occur before rejection receives the destination, two, five, and 9216. This function performs neither inspection-byte read, neither processing call, nor result transfer.

2. **Disjoint nonzero classifications still reject.**  
   For classifications one and two, the intersection is zero, but the first classification is nonzero. Rejection follows.

3. **Two-byte processing permits a nonzero eligibility-position byte.**  
   For classifications zero and seven, offset-two byte 255, and offset-four byte 18, F-5a2673aeb07d receives those byte values. The 255 does not cause rejection on this branch.

4. **Zero eligibility selects one-byte processing.**  
   With both classifications zero and offset two containing zero, the function reads offset four and calls F-d9951cfd4fb4. It does not call F-5a2673aeb07d.

5. **Failed eligibility prevents the later read.**  
   With both classifications zero and offset two containing 128, rejection follows the eligibility read. This function does not read offset four or transfer a result.

6. **Classification can replace the inspection address.**  
   If classification calls change the address slot, later byte reads use the resulting address. Validity of the original indirect-call result does not make an invalid replacement safe.

7. **Noninterfering transfer preserves the transfer-start representation.**  
   Suppose the result slot contains 0x12345678 when transfer begins. Under little-endian interpretation without interfering mutation, the writes carry 0x5678 and then 0x1234. Completion follows both writes. Final contents after completion remain dependent on completion’s behavior.

8. **Overlap can change the second portion.**  
   Suppose the same initial slot value is used, but the destination points two bytes into the slot. Assume both destination writes are valid, including the write beyond the slot’s extent. The first write replaces the upper portion with 0x5678. The later upper-portion read therefore obtains 0x5678. Immediately before completion, absent other interference, the slot contains 0x56785678.

9. **An unchanged processor does not prove initial-value preservation.**  
   Suppose an earlier dependency changes the slot to 0xA1B2C3D4 through a valid alias, and the processor leaves it unchanged. With little-endian, noninterfering transfer, the writes carry 0xC3D4 and 0xA1B2, regardless of a different originally stored third-input representation.

10. **Initial-value transfer requires preservation throughout.**  
    If no earlier dependency or other interference changes the initialized slot, the processor leaves it unchanged, and transfer is noninterfering, the transferred representation is the third input’s initial representation under the stated layout interpretation.

11. **The fourth input is overwritten.**  
    If executions differ only in the fourth input and dependencies produce identical relevant results and effects, subsequent explicit classification and inspection behavior is identical because the indirect return replaces the stored fourth input first.

## Uncertainties

The following remain **documented_only**:

- Application-level meanings of the destination, classifications, inspection bytes, and rejection constants.
- The indirect target’s identity and full contract.
- Dependency effects on globals, aliases, ownership, and lifetimes.
- Whether earlier dependencies actually possess aliases permitting result-slot modification.
- Whether processors normally replace or preserve the result slot.
- Whether an external contract prohibits destination/result-slot overlap.
- Whether completion modifies the slot, destination, or other reachable memory.
- Exception, termination, fault, and other non-returning behavior.
- Full ABI treatment of byte-valued arguments.
- Independently established target byte order.
- Relative evaluation order of the two byte arguments to F-5a2673aeb07d.
- Defined behavior for invalid addresses or overflowing address calculations.

Unknown alias availability does not prove that an earlier modification occurs. Equally, the absence of a direct result-slot argument to earlier dependencies does not establish preservation through all possible aliases.

## Evidence

The supplied static body establishes the four inputs, two four-byte slots, dependency arguments and ordering, indirect dispatch offset, 32-bit classifications, branch conditions, and conditional byte reads.

It establishes the transfer sequence: obtain the low portion, write the first destination portion, read the upper portion, write the second destination portion, then invoke completion. It does not establish a four-byte snapshot or exclude overlap.

The slot initialization occurs before destination acquisition, indirect invocation, both classifications, and processing. No supplied dependency implementation or preservation contract establishes that the initialized representation survives those intervening operations. Initial-value transfer therefore requires preservation by earlier dependencies and other interference as well as by the processor.

The dependency map supplies the opaque IDs. Role descriptions are inferred intent. Numerical byte-layout examples are conditional on little-endian interpretation.

No runtime tests have been performed.
