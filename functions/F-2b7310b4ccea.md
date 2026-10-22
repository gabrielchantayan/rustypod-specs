<!-- Trusted provenance: F-2b7310b4ccea; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2b7310b4ccea

## Purpose

F-2b7310b4ccea sets or clears one selected bit in a memory-resident 32-bit word. A 32-bit unsigned selector determines both a record displacement and a bit position. A separate signed control determines whether the bit is cleared or set.

The function performs a full-word read-modify-write and returns zero on normal completion. It does not return the previous bit value, indicate whether the value changed, or report whether the selected storage was valid.

The address calculation supports interpreting the storage as records with a 32-byte stride and a flag word at byte offset 4. The application-level meaning of the records and bits is not established.

## Interface

### Explicit inputs

- **Selector:** a 32-bit unsigned integer. Its lowest three bits select bit positions 0 through 7. Higher bits participate in displacement calculation, subject to the unsigned arithmetic described below.
- **Control:** a 32-bit signed integer. Zero requests clearing the selected bit; every nonzero value requests setting it.

The control is tested for equality with zero. Its magnitude and sign do not otherwise influence the stored value. In particular, negative controls request setting, not clearing.

### Implicit input

The function obtains a base value from global storage and uses it in the target-address calculation.

The supplied body does not establish the base value’s initialization, storage width, allocation provenance, lifetime, or application-level ownership. It also does not establish the type and execution semantics needed to resolve every aspect of the subsequent base-address additions and address conversion.

### Output

On normal completion, the function returns a 32-bit zero value.

There are no distinct normal results for an already-satisfied request, a changed word, a negative control, or an application-invalid selector. There is no explicit failure return.

### Storage preconditions

Successful execution requires that the calculated target support a 32-bit read followed by a 32-bit write. The storage must remain usable through both accesses, with any required alignment and access permissions supplied by the execution environment.

No selector bounds check is present. Being representable as a 32-bit unsigned selector does not establish that the resulting target belongs to an allocated object or is otherwise accessible.

The function supplies no synchronization mechanism. Any requirement for serialized access must be satisfied externally.

## Behavior

### Selecting the bit

The lowest three selector bits determine a bit position from 0 through 7. The corresponding mask has exactly one set bit and ranges from numeric value 1 through 128.

Consequently:

- Only bits 0 through 7 of the loaded word can be directly selected.
- Bits 8 through 31 are never selected.
- Mask construction does not require a shift outside the established safe range.

These are numeric bit positions within the 32-bit word. They do not establish which physical byte contains those bits without an endianness specification.

### Calculating the displacement

First, the unsigned selector is shifted right by three bits. This is a logical shift, producing a value from 0 through 536,870,911.

That value is multiplied by 32 using unsigned 32-bit arithmetic. The multiplication therefore wraps modulo \(2^{32}\), independently of pointer width or the semantics of later address operations.

For selector \(s\), the established displacement is:

\[
d = \left(32 \times \left\lfloor s/8 \right\rfloor\right) \bmod 2^{32}.
\]

The possible displacements are multiples of 32, from zero through `0xFFFFFFE0`.

This has several unconditional integer consequences:

- Selector bits 0 through 2 do not affect the displacement.
- Selector bits 3 through 29 determine the displacement.
- Selector bits 30 and 31 do not affect the displacement.
- Selectors differing only in their highest two bits produce the same displacement and select the same bit.

The last two properties are not conditional on a 32-bit pointer model. They follow from the unsigned multiplication before any subsequent addition to the global base.

A mathematical record index can still be described as the selector divided by eight and rounded down. However, a mathematical product of that index and 32 is not necessarily the displacement used by the function: the product must first be reduced to 32 bits.

### Forming the target address

The function combines the globally obtained base value with the already-wrapped displacement and then the fixed offset of four bytes, and converts the result into a pointer to a 32-bit word.

The intended layout relationship is therefore:

**target = address formed from base + wrapped 32-bit displacement + 4 bytes**

It is not generally correct to describe the target as an unbounded mathematical base plus 32 times the full record index plus four.

For example, selector `0x40000000` produces displacement zero, not a displacement of \(2^{32}\). That reduction has already happened before the base participates.

The supplied representation does not fully establish the arithmetic width, conversions, overflow behavior, or address validity of the subsequent base additions and pointer conversion. Those uncertainties must remain separate from the established unsigned displacement wrapping.

For selectors whose multiplication does not wrap, and where the later address operations have the expected non-overflowing meaning, the layout is a 32-byte record stride with the target word four bytes into each record.

### Reading and updating the word

The function reads the 32-bit word at the computed target.

For control zero, the replacement word has the selected bit cleared. For every nonzero control, the replacement word has the selected bit set.

In either case, all unselected bits retain their values from the word read by this invocation. This includes all upper 24 bits. The control is not copied into the word and does not replace an entire byte or field.

The replacement is written as a full 32-bit word to the same computed target.

### Observable access order

The explicit access sequence in the supplied body is:

1. Obtain the base value from global storage.
2. Read the computed target as a 32-bit word.
3. Write the replacement as a 32-bit word to that same target.
4. Return zero.

The target is calculated before the target-word read. The write uses that calculated target rather than obtaining the base again.

Both control cases perform the target-word read and write. There is no explicit shortcut when the requested bit already has the desired value. Setting an already-set bit or clearing an already-clear bit still reaches the full-word store.

This describes the accesses represented by the supplied body. It does not establish hardware transaction counts, compiler ordering guarantees, memory barriers, or device-specific visibility.

## Side effects

### Target mutation

The explicit mutation is one 32-bit target-word store. Relative to the value read by this invocation, only the selected bit can change.

This preservation rule does not guarantee preservation of intervening concurrent updates. If another writer modifies the word after this function reads it, the later full-word store can overwrite that modification, including a modification to an unselected bit.

No lock, atomic read-modify-write primitive, retry loop, or memory barrier is visible. Atomicity and loss-free concurrent updates are not established.

### Aliasing and global state

There is no explicit store directed at the global base-value location as such. Nevertheless, the supplied evidence does not prove that the calculated target cannot overlap that location or another significant object.

If the target aliases the base-value storage, the target-word store can modify that storage. The current invocation still writes through its previously calculated target; it does not recompute the target after that store.

Claims that only an isolated table object changes require external layout and non-aliasing guarantees.

### Ownership and lifetime

No allocation, deallocation, ownership transfer, or reference-management operation is visible. Access through the global base does not establish who owns the storage or who must release it.

No called function is involved whose memory-management behavior needs to be assumed.

### Errors and abnormal execution

The body contains no explicit checks for an unusable base, insufficient allocation size, invalid alignment, inaccessible memory, or concurrent modification. It also contains no explicit recovery path for an unsuccessful memory access.

Unsigned displacement wrapping is established behavior, not an explicitly reported error. Whether the resulting address is valid is a separate question.

If an access cannot complete, the function provides no specified nonzero status. Fault delivery, external recovery, and other platform consequences remain unresolved. The zero return applies only to normal completion.

## Dependencies

No callable dependencies are visible in the supplied function body. There are therefore no additional opaque function IDs to list.

The function depends on:

- Global storage supplying the base value.
- Storage supporting the calculated 32-bit read and write.
- The execution environment’s base-arithmetic, pointer-conversion, alignment, and memory-access rules.
- Any external lifetime and synchronization discipline required for that storage.

No unseen initializer, allocator, validator, or synchronization routine is assumed to provide guarantees.

## Acceptance examples

These are conditional expectations derived from static evidence, not executed tests. Except where stated otherwise, they assume usable storage, stable relevant state, no concurrent modification, and subsequent address operations that produce the described accessible target.

### Clearing a low bit

With selector 3, control zero, and target word `0xA5A500FF`:

- The displacement is zero.
- The fixed word offset is four bytes.
- The selected bit is bit 3.
- The stored word is `0xA5A500F7`.
- Normal return is zero.

All unselected bits retain their read values.

### Setting with a negative control

With selector 6, control −1, and target word `0x12340000`, the function stores `0x12340040` and normally returns zero.

The negative control requests setting because it is nonzero. It is neither rejected nor interpreted as a request to clear.

### Crossing the first record boundary

Selector 7 produces displacement zero and selects bit 7.

Selector 8 produces displacement 32 and selects bit 0.

Where ordinary non-overflowing address formation applies, these target the words at base plus 4 and base plus 36 bytes respectively. Advancing from selector 7 to selector 8 moves to the next record rather than selecting bit 8 of the first word.

### Already-satisfied requests

With selector 0, nonzero control, and target word `0x80000001`, the stored value remains `0x80000001`.

With selector 0, zero control, and target word `0x80000000`, the stored value remains `0x80000000`.

Both executions still include a target-word read and full-word write. Equal initial and final values do not imply that the write is omitted.

### High-selector equivalence

Selectors `0x00000000`, `0x40000000`, `0x80000000`, and `0xC0000000` all produce displacement zero and select bit 0.

This displacement equivalence follows unconditionally from unsigned 32-bit multiplication. It is not contingent on pointer width.

With the same base value and address-formation semantics, these selectors supply identical operands to the subsequent address calculation and therefore form the same target. Successful access still requires that target to be usable; the function establishes no application-level permission or bounds policy for these selectors.

### Maximum selector

Selector `0xFFFFFFFF` produces a shifted value of `0x1FFFFFFF`. Its mathematical product with 32 is \(2^{34}-32\), but the unsigned 32-bit displacement is `0xFFFFFFE0`. The selected bit is bit 7.

The wrapped displacement is established. The final address and its accessibility cannot be specified without the unresolved base-addition and conversion semantics.

### Inaccessible target

If the computed target cannot be read or written, no normal error-return value is established. An acceptance procedure must use the execution environment’s actual invalid-access behavior rather than expect the function to return a nonzero status.

## Uncertainties

### Inferred intent

The 32-byte stride, four-byte word offset, and eight selectable low bits suggest a flag word within a record layout.

That interpretation does not establish record count, flag meanings, or whether high-selector equivalence is an intended application feature.

### Documented-only unknowns

The following remain **documented_only**:

- The base value’s meaning, initialization, storage width, and lifetime.
- Allocation size and application-valid selector range.
- The exact semantics of base addition and conversion to the target pointer.
- Pointer width and address-space constraints.
- Alignment requirements and invalid-access behavior.
- Target endianness.
- Device-specific or externally observable access effects.
- Caller synchronization and concurrent-access rules.
- Whether the target can alias global base storage or other state.
- External calling-convention details beyond the exposed scalar behavior.

These unknowns do not weaken the established 32-bit unsigned displacement wrapping. Conversely, knowing the wrapped displacement does not resolve the final address’s validity.

## Evidence

The supplied static body establishes:

- A 32-bit unsigned selector and 32-bit signed control.
- Logical right shift by three for displacement calculation.
- Multiplication by 32 in unsigned 32-bit arithmetic.
- Displacement wrapping before combination with the global base.
- A fixed additional offset of four bytes.
- Selection of one mask bit using the selector’s lowest three bits.
- One 32-bit target-word read and one 32-bit target-word write.
- Clearing for zero control and setting for nonzero control.
- A constant 32-bit zero normal return.
- No explicit calls, bounds validation, allocation, synchronization, or recovery branch.

The record-layout interpretation is inferred intent. Subsequent address semantics, ownership, and external access guarantees remain unresolved.

No runtime tests have been performed.
