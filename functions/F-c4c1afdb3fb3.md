<!-- Trusted provenance: F-c4c1afdb3fb3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c4c1afdb3fb3

## Purpose

F-c4c1afdb3fb3 derives a memory address from a 16-bit unsigned value and two integer inputs, reads two adjacent signed 16-bit values immediately before that address, and returns their difference through a signed 16-bit return interface.

The first input identifies where to obtain the unsigned address component. It is not itself added to the final read address. The value fetched through that input participates directly in address calculation.

The established behavior is a small, unchecked memory-reading operation. The supplied evidence does not establish a larger data structure, application purpose, error protocol, allocation convention, or ownership transfer.

## Interface

### Inputs

The recovered interface has three integer inputs:

| Input | Role |
|---|---|
| First input | Integer used as an address base for a 16-bit unsigned read at byte offset 28. |
| Second input | Integer added to the fetched unsigned value when deriving the address for the final reads. |
| Third input | Integer multiplied by negative two when deriving that address. |

The first input is used as an address-bearing integer, not as an explicitly typed pointer in the supplied representation. The second and third inputs are arithmetic operands. No evidence establishes that either is a pointer, count, index, byte length, or element displacement, although such interpretations may be plausible.

The recovered representation distinguishes:

- An unsigned 16-bit memory value fetched using the first input.
- Two signed 16-bit memory values fetched using the derived address.
- Integer arithmetic wider than those 16-bit values.
- A signed 16-bit result.

The address-oriented integer types appear consistent with a 32-bit model. Exact ABI and language data-model information were not supplied, so claims depending on 32-bit address wrapping are conditional rather than unconditional guarantees.

### Output

The function returns a signed 16-bit value.

When the mathematical difference between the two final memory values is in the range −32,768 through 32,767, the returned value is exactly that difference.

When that difference is outside the signed 16-bit range, returning it requires a narrowing conversion. The supplied representation establishes that narrowing occurs but does not independently establish its exact out-of-range semantics. Low-16-bit truncation followed by signed interpretation is a conditional target behavior, not an unconditional portable guarantee.

There is no separate status result, explicit error indicator, or output-buffer argument.

### Preconditions

For ordinary completion with a defined result:

- The address represented by the first input, plus 28 bytes, must support a readable 16-bit unsigned access.
- The address calculation must have defined behavior under the actual target’s integer and address rules.
- Both final addresses must support readable signed 16-bit accesses.
- Each access must satisfy any alignment, mapping, and object-access requirements imposed by the execution environment.
- If an exact, target-independent signed result is required, the final difference must fit the signed 16-bit return range.

No checks for these conditions are visible.

## Behavior

### Initial read

The function first fetches one unsigned 16-bit value from the address 28 bytes after the address represented by the first input.

Call this fetched value the **address component**. Its numeric range is 0 through 65,535. It contributes as an unsigned value; values with the high bit set are not treated as negative signed 16-bit numbers.

Only this 16-bit field is visibly accessed through the first input. The evidence does not establish the existence or contents of any surrounding object.

### Address derivation

For inputs whose arithmetic does not encounter overflow or address-conversion ambiguity, define the **reference address** mathematically as:

**address component + second input − 2 × third input**

The function then uses the two addresses:

- Reference address minus 4 bytes.
- Reference address minus 2 bytes.

The first input’s address is not added to this expression. For example, relocating the storage containing the address component while retaining the same stored value does not, by itself, relocate the final two reads.

The third input changes the reference address in two-byte increments. Increasing it by one decreases the reference address by two, provided all involved arithmetic remains defined. Increasing the second input by one increases the reference address by one; the function does not enforce an even address or alignment.

### Integer-width considerations

The initial 16-bit value is explicitly treated as unsigned before participating in the wider addition. In the recovered expression, that unsigned operand affects the usual arithmetic conversions for the surrounding additions.

Under a 32-bit integer interpretation:

- The third input’s multiplication by negative two is a signed 32-bit operation before its result participates in the unsigned addition.
- That multiplication is mathematically representable only when the third input is between −1,073,741,823 and 1,073,741,824, inclusive.
- Outside that interval, signed multiplication overflow prevents a portable source-level guarantee.
- Where the multiplication is defined, additions involving the unsigned operand use unsigned arithmetic and therefore wrap modulo 2³² in that interpretation.

These rules do not establish what an arbitrary wrapped integer means as a usable memory address. The supplied evidence also does not settle every detail of the subsequent address-offset operations or integer-to-pointer conversions.

Consequently, the simple mathematical address formula is definitive for the ordinary non-overflowing case. Behavior at arithmetic or address-space boundaries remains target-dependent or otherwise unspecified by the available evidence.

### Final reads and subtraction

The function reads:

- A signed 16-bit value at reference address minus 4.
- A signed 16-bit value at reference address minus 2.

It subtracts the second value from the first.

The two locations are adjacent two-byte positions. The four bytes occupied by those two values immediately precede the reference address. Nothing is visibly read at the reference address itself.

Each loaded value is interpreted as signed before subtraction. In a 32-bit integer model, the subtraction is performed after promotion to the wider integer type. Its mathematical range is −65,535 through 65,535, so that subtraction itself does not overflow a signed 32-bit integer.

The wider difference is then converted to the signed 16-bit return type. Out-of-range narrowing is distinct from overflow during subtraction: the subtraction can be valid even when the return value cannot represent its result exactly.

### Observable ordering

The initial unsigned read must supply the value used to calculate both final addresses. It therefore precedes the dependent final reads in the established data flow.

The supplied expression does not establish which of the two final reads occurs first. Their positions as the left and right operands of subtraction establish their roles in the result, not a guaranteed left-to-right access order.

There is no visible reread of the initial field. If another actor changes it after it has been fetched, this invocation’s established calculation uses the fetched value. No atomicity or synchronization guarantee accompanies that observation.

## Side effects

The visible function body contains three memory reads and no explicit memory writes:

1. One unsigned 16-bit address-component read.
2. Two signed 16-bit operand reads.

It contains no visible calls, allocations, deallocations, ownership transfers, global assignments, logging operations, or error-state updates.

This does not guarantee that every attempted execution is externally inert apart from its result. An invalid access can fault. A read from memory-mapped hardware can have device-defined effects. Concurrently modified storage may raise consistency or language-memory-model issues. None of those cases is handled explicitly.

The function does not visibly acquire or release ownership of any referenced storage. Callers must arrange for the accessed storage to remain usable for the required reads. The evidence supplies no retention or lifetime extension mechanism.

## Dependencies

No called function appears in the supplied body. There are therefore no opaque callee IDs to enumerate.

The behavior depends on execution-environment facilities rather than visible callable dependencies:

- Representation and interpretation of signed and unsigned 16-bit values.
- Native byte order for the memory reads.
- Width and conversion rules of the address-bearing integer types.
- Integer-to-address interpretation.
- Alignment and memory-access rules.
- Out-of-range conversion to the signed 16-bit return type.

No assumption about an unknown callee preserving globals, retaining memory, or releasing memory is needed: no such callee is visible.

## Acceptance examples

These are conditional specification examples, not reported executions. Unless stated otherwise, assume readable, suitably aligned memory and an address calculation with no overflow or conversion ambiguity.

### Ordinary positive difference

Suppose:

- The initial field contains 100.
- The second input is 1,000.
- The third input is 3.

The reference address is 1,094. The final reads are at addresses 1,090 and 1,092.

If those locations contain signed values 20 and 7 respectively, the returned value is 13.

### Negative difference

Using the same addresses, if the first final value is −12 and the second is 5, the returned value is −17.

This distinguishes signed-value subtraction from subtraction of unsigned interpretations of the same stored bits.

### Unsigned address component

If the initial field has numeric unsigned value 65,535, its contribution to the reference address is positive 65,535, not −1.

Any acceptance check for this case must also ensure that the resulting addresses are valid under the target’s address rules.

### Negative third input

If the address component is 100, the second input is 1,000, and the third input is −3, the reference address is 1,106. The final reads occur at 1,102 and 1,104.

A negative third input is not visibly rejected. It reverses the direction of the two-times adjustment.

### Return-range boundary

If the first final value is 32,767 and the second is 0, the return is exactly 32,767.

If the first is −32,768 and the second is 0, the return is exactly −32,768.

Both differences fit the return type.

### Out-of-range return

If the first final value is 32,767 and the second is −1, the wider difference is 32,768.

The evidence establishes a signed 16-bit narrowing step, but not a target-independent result for this input. On a target that retains the low 16 bits and interprets them as two’s-complement signed, the result is −32,768. That expectation is valid only under the stated target condition.

### Invalid read address

If the initial read or either final read addresses unreadable memory, no normal error return is specified. An acceptance check must use the target’s actual fault behavior rather than require a sentinel value.

## Uncertainties

The following remain **documented_only**:

- Exact integer and pointer widths of the execution target.
- Exact behavior at signed multiplication overflow.
- Integer-to-pointer conversion and address-wrap semantics.
- Out-of-range signed 16-bit return conversion.
- Native byte order and alignment requirements.
- Which final operand is read first.
- Fault handling for invalid memory.
- Atomicity and behavior under concurrent mutation.
- Whether any accessed region is ordinary memory or device memory.

The apparent intent is to compute a difference between neighboring signed 16-bit entries selected by a derived address. That is an inference, not evidence of an array layout, index convention, or application-level meaning.

## Evidence

Established static evidence consists solely of the supplied function representation:

- Three integer inputs and a signed 16-bit return.
- An unsigned 16-bit read at byte offset 28 from the first input.
- A derived address combining that value, the second input, and negative twice the third input.
- Signed 16-bit reads four and two bytes before the derived address.
- Subtraction of the latter value from the former, followed by return-type narrowing.
- No visible branches, checks, stores, or calls.

No runtime tests have been performed. The acceptance examples describe conditional expected behavior; they are not execution evidence.
