<!-- Trusted provenance: F-b63ebee7ff72; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b63ebee7ff72

## Purpose

F-b63ebee7ff72 conditionally updates a signed 32-bit value in a caller-supplied record. An update is eligible only when:

1. An unsigned selector equals the record’s first byte.
2. The proposed value does not exceed the signed bound selected within the record.
3. The proposed value is not less than the record’s existing signed value.

After an eligible update, a nonzero value obtained through a global reference enables a second write. That write stores the sum of another signed record field and the proposed value at offset 36 from the referenced destination.

This is an operational description, not evidence of the record’s application-level meaning. Interpreting the fields as a current position, limit, or progress counter would be inferred intent.

The function has no explicit result or error-reporting channel.

## Interface

### Inputs

The function receives three arguments:

| Input | Representation | Role |
|---|---|---|
| Record address | Address of byte-addressable storage | Supplies the discriminator, mutable value, additive field, and selected bound. |
| Selector | Unsigned 32-bit integer | Must equal the record’s first byte; also selects the bound. |
| Proposed value | Signed 32-bit integer | Candidate value for the mutable field. |

The widths above follow the recovered byte, unsigned-integer, and signed-integer types. The supplied material does not independently establish the target ABI, pointer width, byte order, or machine alignment rules.

### Record layout

Offsets are byte offsets from the record address.

| Offset | Width | Interpretation |
|---|---:|---|
| 0 | 1 byte | Unsigned discriminator |
| 4 | 4 bytes | Signed existing value; overwritten on acceptance |
| 8 | 4 bytes | Signed additive field; read only when the secondary write is enabled |
| 16 + 4 × selector | 4 bytes | Signed selected upper bound |

No meaning is established for bytes between these fields or for any other record storage.

The selector is not truncated to a byte for comparison. The discriminator is compared numerically with the full unsigned selector. Therefore, a selector greater than 255 cannot match and returns before the selected-bound access.

On a matching path, the selected bound starts at an offset between 16 and 1036 inclusive. At selector 255, its four-byte extent ends at offset 1039. The selector-derived multiplication and offset addition do not overflow a 32-bit unsigned integer on any path that reaches this access.

### Global destination state

The function consults a value obtained by dereferencing a global reference. This contract calls the accessed storage the **destination-reference cell**.

- A zero cell value disables the secondary write.
- A nonzero cell value is used as a destination base.
- The secondary output is a signed 32-bit store at byte offset 36 from that base.

The supplied material does not establish the cell’s declaration, pointer representation, initialization mechanism, ownership, or lifetime.

### Return and ownership

There is no explicit return value. Rejected proposals and accepted proposals both return normally when their required memory accesses and arithmetic are valid.

The function contains no allocation, deallocation, or ownership-transfer operation. It neither establishes nor extends the lifetime of the record, the global reference, the destination-reference cell, or the secondary destination.

## Behavior

### 1. Discriminator gate

The first operation is a one-byte read at record offset 0.

If that byte does not equal the unsigned selector, the function returns. On this path, it does not access the selected bound, the mutable value, the additive field, or the global destination state.

This gate is an exact equality test. A selector whose low byte matches the discriminator but whose numeric value exceeds 255 is rejected.

### 2. Selected upper-bound gate

When the discriminator matches, the function reads a signed 32-bit bound at offset:

**16 + 4 × selector**

It compares the proposed value with this bound using signed ordering.

- A proposed value greater than the bound causes an immediate return.
- A proposed value equal to or less than the bound proceeds.

Equality at the upper bound is accepted by this gate.

The function does not first inspect the existing mutable value. Consequently, an upper-bound rejection requires no read of that value.

### 3. Existing-value gate

After passing the upper-bound gate, the function reads the signed 32-bit existing value at offset 4.

- A proposed value less than the existing value causes an immediate return.
- A proposed value equal to or greater than the existing value proceeds.

Equality with the existing value is accepted. Acceptance is therefore not limited to strict increases: proposing the existing value can still cause a store to the record and a secondary output write.

In ordinary stable memory, the acceptance condition is:

**existing value ≤ proposed value ≤ selected bound**

If the existing value is greater than the selected bound, no signed proposed value satisfies both gates. The function does not repair that state or report it.

### 4. Primary store

After both signed comparisons pass, the function stores the proposed value as a signed 32-bit value at record offset 4.

This store occurs before the function consults the destination-reference cell. It is not conditional on the secondary destination being enabled or valid.

The function has no rollback operation. A later invalid access does not provide a contractual guarantee that the primary store is undone.

### 5. Optional secondary store

After the primary store, the function reads the destination-reference cell.

If the cell value is zero, the function returns without reading the additive field at record offset 8 and without performing the secondary write.

If the cell value is nonzero, the function uses the referenced destination base and reads the signed 32-bit additive field at record offset 8. It forms the sum:

**additive field + proposed value**

It then stores that result as a signed 32-bit value at byte offset 36 from the destination base.

The represented operation uses the global destination reference for both the enablement check and the destination calculation. It does not establish an explicit stable snapshot of that reference across the two uses. A contract requiring stability must therefore impose it as an environmental precondition rather than assume synchronization supplied by this function.

The relative evaluation order of the destination-address expression and the sum expression is not independently established. Both belong to the secondary assignment, which follows the primary store.

### Signed arithmetic boundary

The secondary addition is expressed using signed integer operands. No explicit widening, saturation, overflow check, or unsigned conversion is shown.

For a defined, portable signed-32-bit result, the mathematical sum must lie within:

**−2,147,483,648 through 2,147,483,647**

Outside that range, the supplied material does not establish a reliable output value. In particular, modulo wraparound must not be treated as an established contract.

## Side effects

### Record mutation

On an accepted proposal, the function explicitly writes four bytes at record offset 4. It does so even when the proposed value equals the existing value.

Rejected proposals perform no explicit stores in the represented function.

### Secondary destination mutation

When an accepted proposal reaches a nonzero destination-reference cell, the function explicitly writes four bytes at destination offset 36.

There is no explicit write to the destination-reference cell itself. However, the supplied material provides no non-aliasing guarantee. A computed output address could overlap the record, the cell, or other accessible storage.

### Aliasing consequences

The primary store precedes the global-state read. If record offset 4 aliases the destination-reference cell, the primary store can affect whether or where the secondary write occurs.

Likewise, if the secondary output overlaps record offset 4, the final record value need not remain equal to the proposed value. If it overlaps another record field, that field can change despite not being an explicit primary-store target.

These are consequences of possible address overlap, not evidence that such overlap is intended or valid for the original application.

### Access and lifetime preconditions

Memory requirements are path-dependent:

- Every invocation requires a valid one-byte read at the record address.
- A matching discriminator requires a valid four-byte selected-bound read.
- Passing the upper-bound gate requires a valid four-byte read at offset 4.
- Acceptance requires a valid four-byte write at offset 4 and valid access through the global reference to the destination-reference cell.
- An enabled secondary write additionally requires a valid four-byte read at offset 8 and a valid four-byte write at destination offset 36.

All reached typed accesses must meet the target’s alignment and memory-access requirements. Address calculations must produce valid addresses for their respective accesses.

The function performs no explicit null checks, bounds checks against an allocation size, synchronization, or recovery from inaccessible memory. A zero record address is not a supported sentinel: the first operation dereferences it.

## Dependencies

No called functions are present in the supplied material. There are therefore no additional opaque function IDs to list.

The function depends directly on:

- The caller-supplied record storage.
- A global reference and the destination-reference cell reached through it.
- Optional writable destination storage.
- The target’s integer, pointer, alignment, and memory-access semantics.

No callee behavior is assumed. In particular, this contract does not rely on an unknown helper preserving globals or allocating, owning, or releasing memory.

## Acceptance examples

These examples are conditional specifications, not reports of execution. Unless stated otherwise, they assume valid, stable, suitably aligned, non-overlapping storage and representable signed arithmetic.

### Discriminator mismatch

Given discriminator 7 and selector 8, the function returns after reading the discriminator. It does not consult any bound or perform either store.

The proposed value is irrelevant on this path.

### Full-width selector comparison

Given discriminator 7 and selector 263, the function rejects the invocation at the first gate. The selector’s low byte being 7 does not make it match.

No bound access at an offset derived from 263 occurs.

### Proposal above the bound

Given matching discriminator and selector, selected bound 20, and proposed value 21, the function returns before reading the existing mutable value. No store occurs.

### Proposal below the existing value

Given matching discriminator and selector, selected bound 20, existing value 12, and proposed value 11, the function reads both comparison fields and returns without storing.

### Accepted proposal with secondary output disabled

Given matching discriminator and selector, selected bound 20, existing value 12, proposed value 15, and a zero destination-reference cell, the function stores 15 at record offset 4.

It does not read the additive field or perform a secondary store.

### Equality remains an accepted operation

Given selected bound 20, existing value 12, proposed value 12, additive field 100, and an enabled secondary destination, the function stores 12 at record offset 4 and stores 112 at destination offset 36.

The lack of a numeric increase does not suppress secondary-output refresh.

### Signed negative values

Given selected bound −2, existing value −10, proposed value −5, and additive field 20, the proposal passes both signed comparisons.

With the secondary destination enabled, the primary result is −5 and the secondary result is 15.

### Inverted interval

Given selected bound 4 and existing value 9, no proposed value is accepted. Values above 4 fail the upper-bound gate; values at or below 4 fail the existing-value gate.

### Largest matching selector

Given discriminator 255 and selector 255, the selected bound is read at byte offset 1036. The function requires that four-byte access only after the discriminator matches.

### Unrepresentable secondary sum

Given an otherwise accepted proposal of 1 and an additive field of 2,147,483,647, the mathematical sum exceeds the signed 32-bit maximum.

The contract does not specify a wrapped secondary result. The primary store precedes this arithmetic, but behavior outside the supported arithmetic precondition is not established.

## Uncertainties

The following remain **documented_only**:

- Target ABI, pointer width, byte order, and exact alignment requirements.
- Application-level meanings of the discriminator, bounds, mutable value, and additive field.
- Actual allocation sizes and object lifetimes.
- The global reference’s declaration and initialization.
- Intended aliasing restrictions.
- Behavior under concurrent mutation, signal-driven mutation, or memory-mapped I/O.
- Exact machine-level load counts and evaluation order within the secondary assignment.
- Results of signed overflow, invalid accesses, or invalid address arithmetic.

A plausible intent is to maintain a nondecreasing value within a selected upper bound and mirror a translated value elsewhere. That interpretation is inferred, not required for the established behavior.

## Evidence

The supplied static function representation establishes the three ordered gates, record offsets, signed comparisons, primary store, zero-controlled secondary store, additive expression, and absence of explicit function calls.

The selector range and maximum selected-bound extent follow from the full-width equality comparison against a one-byte discriminator.

The material does not include target type declarations, surrounding callers, allocation information, synchronization rules, or machine-level execution evidence. No runtime tests have been performed.
