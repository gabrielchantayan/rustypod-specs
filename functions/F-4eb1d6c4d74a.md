<!-- Trusted provenance: F-4eb1d6c4d74a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4eb1d6c4d74a

## Purpose

The established behavior is an ordered sequence of memory writes, two indirect calls, and a terminal bad-instruction indication. The function first records two supplied values in caller-addressed storage, performs several address-dependent halfword writes, updates fixed storage locations, invokes two unresolved dependencies, and clears additional locations.

A higher-level purpose is not established. In particular, the evidence does not justify describing this function as an initializer, allocator, reset routine, or supervisor-service wrapper. Some storage labels suggest such roles, but labels do not establish behavior.

This contract describes the supplied static representation, not verified executable behavior. The representation explicitly reports bad instruction data and truncated control flow. Consequently, its ordering and expressions can be specified, but their correspondence to valid machine instructions remains uncertain.

## Interface

### Explicit inputs

The visible interface has four arguments:

| Input | Visible width and interpretation |
|---|---|
| Destination address | Pointer to storage accessed initially as consecutive 32-bit integer elements. Its numerical address also participates in later address calculations. |
| Second argument | A 32-bit value with no visible use. |
| Recorded value | A 32-bit signed integer. It is stored unchanged and separately used to form a memory address. |
| Selector value | A 32-bit unsigned integer. It is stored unchanged and separately used to select a byte read. |

The unused second argument has no direct effect in the visible body. This does not establish anything about argument evaluation by a caller or an external calling convention.

### Additional entry-state dependencies

The body also depends on values not supplied through those four explicit arguments:

- A register-carried value used as a signed 32-bit quantity.
- A register-carried 16-bit value.
- A link-register-derived address used to locate the first indirect call target.
- A pointer-sized value represented as coming from incoming stack state.
- A global source value whose low 16 bits are read twice.
- All memory reached through the calculated addresses and unresolved callees.

These are genuine behavioral inputs to the visible representation. A contract based only on the four explicit arguments would be incomplete.

### Output and termination

No ordinary return value is established. The visible path ends at a bad-instruction termination marker rather than a normal return.

The two indirect calls have different visible uses:

- The first call supplies a value subsequently treated as a pointer to writable 16-bit storage.
- The second call receives that pointer. Its result is not used visibly.

No ownership transfer, allocation guarantee, release obligation, or lifetime extension is established for that pointer.

### Width and address model

The representation uses 32-bit integer quantities, 16-bit stores, and an 8-bit read. Its pointer arithmetic is consistent with a 32-bit address model and four-byte integer elements.

Address formulas below use that apparent model. Where signed arithmetic could overflow or a signed shift could be invalid under a source-language interpretation, the corresponding 32-bit bit-pattern calculation is stated conditionally. This does not resolve the original instruction semantics.

Endianness, alignment requirements, memory protection, and fault behavior are not supplied.

## Behavior

The following order is the visible order. Completion of any later step requires all earlier operations and calls to complete and return as needed. There are no visible branches, retries, validation checks, or recovery paths.

### 1. Record the supplied values

The function writes the recorded value as 32 bits at the destination address.

It next writes the selector value as 32 bits at destination offset 4.

The destination must therefore support these accesses for execution to proceed. No size, null, alignment, or accessibility check appears before either write.

### 2. Write an address-derived halfword

The function forms the address at destination offset 8 and writes the low 16 bits of that address into the 16-bit storage at that same address.

This is not a store of either supplied value. The written halfword depends on the destination’s numerical address.

The first three visible writes therefore occupy:

- Four bytes at offset 0.
- Four bytes at offset 4.
- Two bytes at offset 8.

A destination object large enough for these ten bytes is not, by itself, sufficient for the rest of the function.

### 3. Write a sign-derived halfword at a calculated address

A separate halfword destination is calculated using both the destination address and its numerical value interpreted through a 32-bit signed integer conversion.

Under four-byte element scaling, the unsimplified address calculation is:

- Begin at destination offset 8.
- Add four times the sum of nine and 256 times the destination’s signed numerical value.

Under modulo-\(2^{32}\) address arithmetic, this simplifies to:

\[
1025P + 44 \pmod {2^{32}},
\]

where \(P\) is the destination address bit pattern.

This is not simply destination offset 44. The numerical destination address is multiplied as part of the calculation.

The stored value is the low 16 bits of the register-carried signed quantity shifted right by 31 positions. With an arithmetic right shift, it is:

- `0x0000` when that quantity is nonnegative.
- `0xFFFF` when that quantity is negative.

Arithmetic right-shift behavior is an interpretation of the displayed signed shift, not independently verified instruction evidence.

### 4. Reconstruct a pointer and perform another halfword write

The function reads the global source value and takes its low 16 bits. It combines those bits with the preserved upper 16 bits of the incoming stack-derived pointer value. The resulting value is retained for a later 32-bit zero write.

The visible assignment reconstructs a pointer value. It does not itself establish a write to the incoming stack location.

The global source value is read again for the next halfword write. Its low 16 bits are written to the address formed by shifting the recorded value left by 20 bits.

Under a 32-bit wrapping interpretation, that address is:

\[
V \times 2^{20} \pmod {2^{32}},
\]

where \(V\) is the recorded value’s bit pattern. Only its low 12 bits contribute to the resulting address.

The two global reads must not be treated as necessarily identical. For example, the intervening access model could involve volatile storage. The evidence provides no stability guarantee.

The shift is displayed on a signed operand. Its valid source-language domain and actual machine behavior are unresolved.

### 5. Update fixed storage before the first call

Three writes follow in this order:

1. Store `0x28` in storage identified with address `0x00000028`.
2. Store zero in a separately labeled global location associated by its name with a supervisor call.
3. Store the register-carried 16-bit value in storage identified with address `0x000897F5`.

The declared widths of the first two global destinations are not included. The final write is represented as a 16-bit value assignment.

The supervisor-related label does not establish that this write executes a supervisor instruction or invokes an operating-system service.

### 6. Read a selected byte and invoke the first dependency

The function calculates a byte address by:

- Taking the selector value’s bitwise exclusive-or with `0x0897F3A8`.
- Multiplying the result by `0x200`.

Under 32-bit unsigned arithmetic, the resulting address is:

\[
((S \mathbin{\mathrm{xor}} \texttt{0x0897F3A8}) \times 512)
\bmod 2^{32}.
\]

It reads one byte at that address.

The first indirect call target is formed by adding `0xA84` to the incoming link-register-derived value. The visible call supplies:

1. The selected byte, converted through a character type to an integer.
2. Integer zero.

Whether bytes `0x80` through `0xFF` become negative integers depends on character signedness. The supplied evidence does not independently establish that property.

The call result is treated as a pointer. There is no visible failure, null, or alignment check.

### 7. Clear the returned halfword and invoke the second dependency

The function writes a 16-bit zero through the pointer returned by the first dependency.

It then passes the same pointer to the second indirect dependency.

The second target is calculated by clearing the low two bits of `0x0897F3D5` and adding `0x604`. The resulting numerical target is `0x0897F9D8`.

No additional target dereference is visible in that calculation. Whether the target represents executable code under the actual execution mode is not established.

### 8. Perform final writes and reach the terminal marker

If the second dependency returns, the following writes occur in order:

1. Store `0xF484` in storage identified with address `0x0000897F`.
2. Store zero in storage identified with address `0x44BF9D40`.
3. Store zero in a separately labeled reset-associated global location.
4. Store a 32-bit zero through the reconstructed pointer retained from step 4.
5. Store zero in storage identified with address `0x00000034`.

The visible path then reaches a bad-instruction termination marker.

The reset-associated label does not establish that execution resets. If writing that location actually has device effects, those effects require information outside the supplied evidence.

## Side effects

### Memory and aliasing

The function performs writes to caller-addressed storage, calculated addresses, and fixed or labeled global locations. It also reads a calculated byte and the global halfword source.

There are no visible non-aliasing guarantees. Calculated destinations may overlap:

- Earlier destination writes.
- Fixed storage.
- The global source.
- Memory used by either dependency.
- The halfword returned by the first dependency.

Order matters when accesses overlap. An earlier write may change a later read, and a later write may overwrite an earlier result.

No rollback is visible. If an access faults or a dependency does not return, earlier effects may already have occurred.

### Dependency effects and ownership

Both dependencies may affect memory or external state. Their effects are unknown and cannot be excluded from the overall behavior.

In particular, the second dependency must not be assumed to preserve globals, retain the pointer, release it, or leave its referenced storage valid. No later direct access through that returned pointer appears, although later calculated writes could alias it.

The visible body performs no explicit allocation or release operation.

### Errors

There is no explicit error result or error-handling branch. Invalid pointers, inaccessible addresses, unsupported alignments, invalid call targets, or dependency failures have no defined recovery behavior in the supplied representation.

No memory-ordering, atomicity, concurrency, or device-register guarantees are established.

## Dependencies

The following are document-local opaque IDs. They distinguish unresolved dependency sites and do not assert verified external identities.

| Opaque ID | Visible contract |
|---|---|
| F-000000000001 | Indirect call at the incoming link-derived address plus `0xA84`. Receives a converted byte and zero. Its result is used as a writable halfword pointer. Actual signature, side effects, ownership, and failure behavior are unknown. |
| F-000000000002 | Indirect call at calculated numerical target `0x0897F9D8`. Receives the pointer returned by F-000000000001 after a halfword zero has been written through it. Return value is unused. Other behavior is unknown. |
| F-000000000003 | Terminal bad-instruction marker in the supplied representation. It establishes no visible normal successor; it is not verified as a genuine callable runtime dependency. |

## Acceptance examples

These examples are conditional consequences of the static representation, not executed tests. Each assumes preceding operations complete and any required memory is accessible.

### Initial destination layout

Given destination address `0x00001000`, recorded value `0x12345678`, and selector `0x89ABCDEF`:

- The 32-bit value at `0x00001000` becomes `0x12345678`.
- The 32-bit value at `0x00001004` becomes `0x89ABCDEF`.
- The 16-bit value at `0x00001008` becomes `0x1008`.

Byte ordering is not specified.

### Scaled address and sign value

For destination address `0x00001000`, the sign-derived halfword destination is `0x0040102C` under the stated 32-bit arithmetic interpretation.

With arithmetic right shift, a nonnegative register-carried signed quantity produces `0x0000`; a negative quantity produces `0xFFFF`.

An implementation that writes this halfword merely at destination offset 44 would not match the displayed address expression.

### Selector-derived byte access

With selector `0x0897F3A8`, the calculated byte address is zero. The visible behavior attempts a byte read there rather than treating zero as a special case.

With selector `0x0897F3A9`, the byte address is `0x00000200`.

### Reconstructed pointer

If the incoming stack-derived pointer value is `0x12345678` and the first global read supplies low halfword `0xABCD`, the retained pointer becomes `0x1234ABCD`.

The later write through it is 32 bits wide. A platform requiring aligned word accesses may reject this address; no alignment correction is visible.

### Failure ordering

If F-000000000001 returns an unusable pointer, the next visible operation attempts a halfword zero write through it. No check intervenes, and F-000000000002 is reached only if that write completes.

If F-000000000002 does not return, none of the five final writes is reached through the visible path.

## Uncertainties

The following remain **documented_only**:

- Whether the supplied representation accurately decodes executable instructions.
- Actual calling convention and provenance of the implicit register and stack inputs.
- Exact widths and storage classes of several globals.
- Character signedness, signed-shift semantics, and signed-overflow treatment.
- Address-space rules, endianness, alignment, and access permissions.
- Whether named globals are ordinary memory, volatile storage, or device registers.
- Dependency signatures, effects, return guarantees, and ownership rules.
- Actual runtime behavior at the terminal bad-instruction marker.
- Any intended higher-level operation.

These uncertainties are not filled by assuming conventional allocation, initialization, reset, or exception behavior.

## Evidence

**Established static evidence:** the supplied representation shows the four-argument interface, ordered accesses and calls, constants, explicit halfword and word operations, implicit entry-state dependencies, and terminal bad-instruction marker.

**Derived conclusions:** address simplifications and example values follow from the displayed expressions under the explicitly stated 32-bit arithmetic interpretation. The sign-derived values additionally depend on arithmetic right shift.

**Inferred intent:** none is adopted as normative behavior. Storage labels alone are insufficient to establish subsystem roles.

**Verification status:** no runtime tests have been performed. No independent instruction validation or dependency implementation evidence was provided.
