<!-- Trusted provenance: F-5b9ccbce1d3f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5b9ccbce1d3f

## Purpose

F-5b9ccbce1d3f exposes a state-mutating operation with a 32-bit address-like input and a signed 16-bit input. On the ordinary returning path represented by the supplied static evidence, it:

- Doubles the 16-bit input with truncation to 16 bits.
- Passes an address derived from the first input to an unresolved dependency.
- Updates several independently obtained memory locations.
- Reads an unsigned byte through the upper half of a dependency’s 64-bit result.
- Uses that byte to select a 32-bit write location and compute a bounded integer.
- Calls F-c260788e6b26 and populates a 20-byte region identified by the upper half of its result.

No application-level purpose, object type, allocation policy, or ownership transfer is established.

The recovered representation also contains a substantial continuation involving F-943b4f706744 and additional unresolved calls. Under the explicit byte widths and arithmetic shown, that continuation cannot be reached from the ordinary path. It is documented separately rather than treated as an alternative supported operation.

This contract distinguishes represented operations from a verified executable specification. The supplied material reports invalid instruction data and incomplete control-flow recovery. No runtime tests have been performed.

## Interface

### Explicit inputs

| Input | Represented width | Contract |
|---|---:|---|
| Address-like input | 32 bits | Used to form an address 80 bytes beyond its supplied value. No validation is shown. |
| Numeric input | Signed 16 bits | Doubled, then retained in a 16-bit storage location. Later writes use either those 16 bits or their low 8 bits. |

The representation describes a procedure without an explicit return value. A normal return does not provide a success indicator.

### Implicit inputs

Behavior also depends on state not supplied through the two explicit inputs:

- An incoming link-related register used to derive the first indirect call target.
- An incoming 32-bit register used by the initial arithmetic gate.
- An incoming address-like register used for a one-byte write at offset 9.
- A global 32-bit base value loaded partway through the operation.
- Memory and results supplied by unresolved dependencies.
- Writable memory identified by the upper 32 bits of F-c260788e6b26’s result.

These implicit inputs prevent the explicit argument pair alone from determining behavior.

### Width and layout conventions

The represented arithmetic and addresses are 32-bit unless a narrower or wider access is specified.

The doubled numeric value is the low 16 bits of twice the original signed 16-bit input. Consequently:

- A 16-bit destination receives that complete truncated bit pattern.
- An 8-bit destination receives its low byte.
- The operation is not saturating arithmetic.
- The represented behavior should not be replaced with unrestricted mathematical doubling.

A 64-bit dependency result is split into low and high 32-bit halves. On the ordinary path, the high half is used as an address. This establishes how the result is consumed, not the dependency’s semantic return type.

Byte order and alignment requirements are not established. Multi-byte writes are specified by width and address, without assuming a particular byte order.

## Behavior

### 1. Initial transformation and first dependency call

The numeric input is doubled and truncated to 16 bits before the first visible call.

The first unresolved dependency receives:

1. The 32-bit address-like input plus 80.
2. The value 50.

The address addition has no shown bounds, overflow, or accessibility check. The call target depends on incoming register state; its identity is not established.

This call occurs before the initial gate is evaluated. Its effects can therefore occur even when the procedure does not reach the ordinary returning path.

### 2. Initial arithmetic gate

The gate uses an incoming 32-bit register value plus 91. The representation separately records:

- Whether the 32-bit signed result is negative.
- Whether signed addition overflows.

Execution proceeds past the first gate only when the signed result is nonnegative.

A later gate compares the recorded negativity and overflow conditions. Under ordinary 32-bit two’s-complement arithmetic, the combined conditions admit incoming signed values from −91 through 2,147,483,556 inclusive.

In particular:

- −91 produces zero and passes.
- −92 produces a negative result and fails.
- 2,147,483,556 produces the largest signed 32-bit result and passes.
- 2,147,483,557 overflows into a negative result and fails the first gate.

On that arithmetic model, every value passing the first gate also satisfies the later gate. This is a deduction from the represented conditions, not independent evidence about the original executable.

A failed gate leads to a terminal invalid-data marker in the supplied representation. That marker does not establish a particular runtime exception, deliberate trap, or error-reporting API.

### 3. Updates before the later gate

When the first gate passes, the following operations occur in order:

1. An unresolved dependency receives the first dependency’s result and a fixed data address.
2. The low 16 bits of this new result are written at address “new result plus 4.”
3. An unresolved dependency is called with no explicit arguments.
4. The low byte of the doubled numeric input is written at offset 9 from an incoming address-like register.
5. Another unresolved dependency is called with no explicit arguments.
6. Its result becomes the destination base for subsequent writes.

The write at offset 4 uses bits of the address-like result itself. It must not be reinterpreted as a generated identifier, a count, or a validated pointer field.

No null checks, range checks, or explicit failure handling are shown between these operations.

### 4. Main state updates

When the later gate passes:

1. The doubled 16-bit numeric value is written at offset 20 from the most recently returned destination base.
2. Its low byte is written at offset 26 from that same base.
3. A global location at numeric address 0xA2 is assigned the value 71.
4. A global 32-bit base value is read and retained.
5. An unresolved dependency is called with the argument 29.
6. The upper 32 bits of that dependency’s 64-bit result are interpreted as an address.
7. An unsigned byte is read at offset 13 from that address.
8. A 32-bit value of 51 is written at the retained global base plus that byte minus 2.
9. Another unresolved dependency is called with twice the difference between the low 32-bit result half and 54.

The width of the assignment at numeric address 0xA2 is not established by the supplied declaration information. The assigned value is established; an 8-, 16-, or 32-bit store must not be asserted without additional evidence.

The byte-indexed 32-bit write has several important properties:

- The byte ranges from 0 through 255.
- Its starting address ranges from two bytes before the retained global base through 253 bytes after it.
- The access is four bytes wide, not one byte.
- Some byte values produce unaligned addresses.
- There is no shown restriction excluding either the before-base writes or the unaligned writes.

The retained global base is loaded before the dependency called with 29. Later uses refer to that saved value. They are not evidence that the global itself remains unchanged across dependency calls.

The final call argument in this stage is a 32-bit arithmetic expression. If its signed mathematical value is outside the signed 32-bit range, the supplied representation does not independently resolve the original machine’s exact arithmetic behavior.

### 5. Bounded derived value and ordinary return

The unsigned byte read above is doubled and increased by 99. Its complete possible range is therefore 99 through 609.

The representation tests whether this derived value is nonnegative. With the shown unsigned-byte read and ordinary 32-bit arithmetic, the test always succeeds: neither overflow nor a negative result is possible.

The procedure then:

1. Calls F-c260788e6b26 with the value 115.
2. Interprets the high 32 bits of its 64-bit result as a destination address.
3. Writes that entire 64-bit result at the destination address.
4. Writes the derived value at destination offset 8 as 32 bits.
5. Writes the retained global base plus 108 at destination offset 12 as 32 bits.
6. Writes the retained global base minus 1 at destination offset 16 as 32 bits.
7. Returns without an explicit value.

The resulting layout is:

| Destination offset | Width | Stored content |
|---:|---:|---|
| 0 | 64 bits | Complete result returned by F-c260788e6b26 |
| 8 | 32 bits | Twice the selected unsigned byte plus 99 |
| 12 | 32 bits | Retained global base plus 108 |
| 16 | 32 bits | Retained global base minus 1 |

The first eight-byte store includes the address-bearing high half of the result. It is not merely a store of the low half.

The four stores are distinct and ordered. No atomic initialization guarantee is established.

### 6. Represented but arithmetically unreachable continuation

The supplied representation places a longer continuation after the nonnegative test. Reaching it would require twice an unsigned byte plus 99 to be negative, contradicting the shown widths and range.

Its operations are therefore **documented_only**, not acceptance requirements for ordinary execution.

At a high level, this continuation represents:

- Reads at offsets 12, 16, and 20 from the retained base.
- Two unresolved calls using values derived from that base and the retained base minus 1.
- Shift-derived addresses and three consecutive 32-bit writes at offsets 52, 56, and 60 from one such address.
- A call to F-943b4f706744 with a fixed data-derived argument and the value 67.
- Further unresolved calls interleaved with signed-byte, unsigned-byte, unsigned-halfword, and 32-bit reads.
- Additional 16-bit, 32-bit, and 8-bit writes whose addresses depend on intermediate arithmetic and otherwise unresolved secondary call results.
- A final call sequence ending in an unresolved indirect transfer.

Several dependencies in this continuation consume results carried outside an ordinary explicit return value. Its terminal control flow was not recovered completely.

No operational fallback, error-recovery path, or second successful mode is established by this material.

## Side effects

### Established direct writes on the ordinary path

In execution order, excluding writes performed inside dependencies:

1. A 16-bit address-derived value at offset 4 from a dependency result.
2. The doubled input’s low byte at offset 9 from an incoming address.
3. The doubled input’s 16-bit pattern at offset 20 from a later dependency result.
4. Its low byte at offset 26 from that same result.
5. The value 71 at numeric address 0xA2, with unresolved store width.
6. The 32-bit value 51 at the retained global base plus the selected unsigned byte minus 2.
7. The four destination stores described in the final layout.

The destinations are not proven disjoint. Earlier writes may affect later reads when addresses alias. Dependencies may also mutate memory between the listed operations.

### Ownership and lifetime

No explicit release operation is established on the ordinary path. This does not establish the absence of allocation or release inside dependencies.

The writable destination returned through F-c260788e6b26 must not be described as newly allocated, caller-owned, permanently valid, or transferred to another owner. Only its immediate use as a writable 20-byte region is visible.

### Errors and partial effects

There is no represented status result, rollback, or cleanup protocol.

An invalid call target, inaccessible address, unsupported alignment, or failed dependency can prevent completion after earlier side effects have already occurred. The exact failure mechanism is platform- and dependency-dependent.

## Dependencies

### F-c260788e6b26

Established use:

- Called with the integer 115.
- Its result is consumed as 64 bits.
- The upper 32 bits identify a destination for subsequent writes.
- The complete result is stored at that destination.

Unknown:

- Whether it allocates, retrieves, or reuses storage.
- Whether it can fail and how failure is encoded.
- Required alignment and available capacity.
- Ownership, lifetime, and effects on globals.

### F-943b4f706744

Appears only in the arithmetically unreachable represented continuation.

Established static use there:

- Receives a fixed data-derived argument and the value 67.

Its semantics, result, and side effects are unknown. It is not called on the ordinary returning path established by the range analysis.

### Unresolved indirect dependencies

Other calls lack identities in the supplied opaque dependency map. No function IDs are invented for them.

Their argument and result uses are described in execution order above. No preservation of globals, memory, ownership, or secondary register results is assumed.

## Acceptance examples

These are conditional static examples, not executed tests. Each assumes valid call targets, returning dependencies, sufficient accessible memory, and no intervening dependency effects that invalidate the described state.

### Doubled numeric input

- Input 3 produces a 16-bit value of 6 and an 8-bit value of 6.
- Input −1 produces the 16-bit pattern 0xFFFE and the byte 0xFE.
- Input 16,384 produces the 16-bit pattern 0x8000 and the byte 0x00.
- Input −32,768 produces the 16-bit pattern 0x0000 and the byte 0x00.

These examples distinguish truncation from saturation and wider signed arithmetic.

### Byte-selected destination

For a retained global base:

- Selected byte 0 causes the 32-bit value 51 to be written two bytes before the base; the final derived value is 99.
- Selected byte 1 causes that write one byte before the base; the derived value is 101.
- Selected byte 255 causes that write 253 bytes after the base; the derived value is 609.

All three cases take the represented nonnegative branch.

### Final destination record

If F-c260788e6b26 returns a 64-bit value whose high half identifies a writable destination, successful completion leaves:

- That complete 64-bit value at offset 0.
- The byte-derived integer at offset 8.
- The retained base plus 108 at offset 12.
- The retained base minus 1 at offset 16.

Expected memory bytes depend on target byte order.

### Early gate failure

For an incoming gate register value of −92, the first unresolved call still occurs. The subsequent ordinary-path writes are not reached in the represented flow. No particular runtime error can be accepted as established from the terminal invalid-data marker alone.

## Uncertainties

The following remain **documented_only**:

- Original executable behavior where instruction recovery was invalid or incomplete.
- Calling convention and values of implicit incoming registers.
- Identities and contracts of unresolved indirect dependencies.
- Whether the represented 64-bit results are semantic integers, packed values, or paired return registers.
- Store width at numeric address 0xA2.
- Byte order, alignment enforcement, and address-overflow behavior.
- Storage capacities, aliasing relationships, ownership, and lifetime.
- Whether dependencies return normally or modify relevant globals.
- The runtime meaning of the terminal invalid-data marker.
- Whether the unreachable continuation reflects valid code under some corrected control-flow interpretation.

Inferring allocation, initialization of a domain-specific object, or intentional low-address access would exceed the evidence.

## Evidence

**Established static evidence:** the supplied representation contains the argument widths, ordered calls and memory operations, access widths where explicitly expressed, constant offsets, and the two mapped dependency identities described above.

**Derived from that evidence:** an unsigned byte doubled and increased by 99 lies between 99 and 609; consequently, the negative continuation is unreachable under the displayed widths. The combined initial gates also yield the stated incoming-register interval under conventional 32-bit signed flag semantics.

**Inferred intent:** the final ordered stores resemble population of a small record. No stronger purpose or ownership claim follows.

**Verification limit:** no runtime tests, executable inspection, dependency inspection, or independent validation has been performed.
