<!-- Trusted provenance: F-4f4a905e7fde; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4f4a905e7fde

## Purpose

F-4f4a905e7fde performs two indirect calls interleaved with global-state clearing, writes a zero byte through an address derived from the second call’s result, and conditionally returns an opaque 64-bit stack-derived value.

A further continuation contains another indirect call, a byte read, two explicitly sized stores, and a final indirect call. That continuation requires a pointer formed from a stack-relative location to compare equal to null. Under the conventional assumption that this stack location has a nonzero address, the function returns before reaching the continuation.

The static evidence does not establish an application-level purpose. The zero assignments suggest state clearing, but do not establish initialization, reset completion, resource release, or any other higher-level guarantee.

This contract distinguishes:

- **Established static evidence:** operations, operands, widths, and ordering expressed in the supplied representation.
- **Conditional inference:** consequences requiring stated assumptions about addresses, integer operations, memory accessibility, or returning callees.
- **documented_only:** unresolved behavior that must not be replaced with an invented implementation guarantee.

## Interface

### Inputs and execution context

The represented interface declares no explicit parameters. Behavior nevertheless depends on incoming machine state:

- An incoming link-register-derived value supplies the basis for the first indirect-call target.
- An opaque stack-derived quantity supplies the represented 64-bit return value.
- The active stack supplies a location at represented stack-relative offset `0x2A8`. A byte-addressed pointer to that location controls the early return and participates in continuation address calculations.

These dependencies do not establish a public calling convention. The representation alone does not determine the physical argument registers, stack-frame layout, instruction state, or complete set of caller obligations.

### Output

Both represented normal exits return the same stack-derived 64-bit quantity. The representation does not use a dependency’s result as the function’s return value and does not define a success or failure status.

The reference to incoming stack storage must not be interpreted as proof that its original contents are captured before the calls. The timing of the return-value read, and its exposure to aliased writes or dependency effects, remain unresolved.

A normal return requires all executed calls and memory operations to complete. No local recovery path is represented.

### Data widths and layouts

The representation distinguishes:

- Explicit one-byte accesses for the result-derived zero store and the continuation’s byte read and byte store.
- A signed 32-bit value for the second call’s result and the signed comparison controlling the loop.
- Unsigned 32-bit values for the continuation’s captured call result and zero-extended selector.
- An explicit 32-bit integer store in the continuation.
- A 64-bit return quantity.
- Two global objects whose assignment widths are **not supplied**.

For this contract, the two globals are:

- **Global A:** the object cleared before the first indirect call.
- **Global B:** the object whose address-like designation represents starting address `0x0000001D`.

Both receive zero assignments. Neither assignment establishes a byte store, a 32-bit store, or any other particular write extent. In particular, Global B’s address-like designation does not establish its type.

Pointer-to-integer conversions are represented through signed integer expressions. A 32-bit address interpretation is suggested, but the actual pointer width and conversion rules are not independently established.

No aggregate layout, allocation boundary, buffer length, or ownership annotation is supplied.

## Behavior

### Initial ordered operations

The first pass through the represented loop-shaped region has this observable ordering:

1. Establish a signed comparison value of zero.
2. Assign zero to Global A, with unknown write width.
3. Form a byte-addressed pointer to the stack-relative location at offset `0x2A8`.
4. Invoke **F-U01** with two zero-valued arguments.
5. Invoke **F-U02** without explicit arguments, interpreting its result as a signed 32-bit integer.
6. Shift that signed result right by four and use the result as the address of a one-byte zero store.
7. Assign zero to Global B, with unknown write width.
8. Test the stack-derived pointer against null. If nonnull, return the represented stack-derived 64-bit quantity.

The ordering is part of the contract. Global A’s assignment precedes both calls. The result-derived byte store follows F-U02 and precedes Global B’s assignment. The null comparison follows both writes.

This sequence does not establish that either global remains zero afterward. Executed dependencies may modify globals, and later memory operations may alias them. Their preservation is not established.

### Address derived from F-U02

F-U02’s signed 32-bit result is shifted right by four before conversion or use as a write address. The function does not first validate the result or the resulting address.

For a nonnegative result, the shift has the numerical effect of division by sixteen with the fractional portion discarded.

For a negative result, the exact address depends on the target’s signed right-shift and integer-to-pointer semantics. An arithmetic right shift is a plausible interpretation, not an independently established machine guarantee.

The resulting store writes exactly one byte with value zero. It is not a store through the unshifted result and is not an explicitly wider integer store.

If its starting address equals Global B’s represented starting address, the representation still contains two ordered zero writes:

- First, an explicitly one-byte store.
- Second, an assignment to Global B whose write width is unknown.

Equal starting addresses do not establish equal extents. They also do not justify merging the operations when access order, access width, or repeated writes are observable.

### Early return and loop exit

The early-return condition depends on the pointer formed from the stack-relative location, not on either dependency’s returned value.

Under a conventional model in which that stack location has a nonzero address, the function returns on its first pass. Neither F-U03 nor F-U04 is then invoked.

If the pointer compares equal to null, the represented loop condition compares signed zero against negative thirty-three. That comparison is false, so execution proceeds to the continuation rather than repeating the initial sequence.

Thus, the visible representation does not specify retries or a second pass. This conclusion concerns the represented control flow, not arbitrary control-state corruption by unknown callees.

### Conditional continuation

**documented_only reachability:** The continuation is structurally present, but its entry requires the stack-derived pointer to compare equal to null. The supplied evidence does not establish whether this can occur in a valid machine execution.

If reached, the continuation has the following order:

1. Invoke **F-U03** without explicit arguments and capture its result as an unsigned 32-bit value.
2. Read one unsigned byte from an address formed by multiplying the signed integer representation of the stack-derived pointer by `0x100`, then adding `0x1F`.
3. Zero-extend the byte to an unsigned 32-bit **selector**.
4. Store one byte at address `selector + 0x1E`. Its value is the byte-width conversion of the pointer-derived signed integer shifted left by five.
5. Store one 32-bit integer at address `selector + 0x70`. Its value is the pointer-derived signed integer shifted left by two.
6. Invoke **F-U04** with these three arguments, in order:
   - F-U03’s captured unsigned result shifted left by four.
   - The integer value eight.
   - F-U03’s captured unsigned result masked to its low 28 bits, then shifted left by three.
7. Return the represented stack-derived 64-bit quantity.

The selector read precedes both stores. The byte store precedes the 32-bit store. Both stores precede F-U04.

The selector is between zero and 255 inclusive. Therefore:

| Access | Represented starting-address range | Width |
|---|---|---|
| Selector-dependent byte store | `0x1E`–`0x11D` | One byte |
| Selector-dependent integer store | `0x70`–`0x16F` | Four bytes |

The second range describes starting addresses, not complete write extents. No alignment check is represented; some selector values produce unaligned 32-bit-store addresses.

Under the additional assumption that a pointer comparing equal to null converts to integer zero, the continuation simplifies to:

- Read the selector byte at `0x1F`.
- Store byte zero at `selector + 0x1E`.
- Store 32-bit zero at `selector + 0x70`.

These are conditional deductions, not evidence that the continuation is executable.

### Arithmetic boundaries

The transformations of F-U03’s captured result are unsigned 32-bit operations:

- The left shift by four discards bits above bit 31.
- Masking to the low 28 bits and then shifting left by three yields at most `0x7FFFFFF8`.

The selector-dependent address additions remain within the numerical ranges stated above.

By contrast, the pointer-derived multiplication and left shifts use signed integer expressions. General wraparound semantics are not established. The supplied representation does not resolve signed overflow, conversion of an out-of-range pointer value, or the exact machine realization of those expressions.

## Side effects

### Direct memory effects

On the initial path, the function directly represents:

- A zero assignment to Global A, of unknown width.
- A one-byte zero store through the address derived from F-U02.
- A zero assignment to Global B, of unknown width.

If the continuation is reached, it additionally represents:

- One explicit byte read.
- One explicit byte store.
- One explicit 32-bit integer store.

The global assignments must remain distinct from the explicitly sized accesses. No surrounding memory layout establishes that these locations are fields of one object, and numerical proximity alone does not establish ownership or shared storage.

### Dependency effects and final state

Every executed dependency may introduce additional effects. The supplied material does not establish that any dependency:

- Preserves either global.
- Preserves the stack-derived return storage.
- Avoids memory used by subsequent operations.
- Allocates, retains, transfers, or releases memory.
- Returns normally.
- Preserves external device or process state.

Accordingly, the contract specifies this function’s represented operations and their order. It does not guarantee an aggregate final memory state despite intervening unknown calls or aliasing.

### Ownership, preconditions, and failures

No ownership transfer is explicitly represented. Two zero arguments to F-U01 do not establish a null-buffer interface. Using F-U02’s result as an address does not establish that the result identifies allocated or owned storage.

For normal completion, every executed indirect target must be callable under the actual ABI, every read must be permitted, and every write must be permitted with its actual width and alignment requirements. Global B’s required write extent cannot be specified from the supplied evidence.

There are no represented bounds checks, result checks, error-status conversions, or exception handlers. Invalid targets, inaccessible memory, alignment faults, arithmetic constraints, or dependency failures may prevent normal return. The resulting platform behavior is **documented_only**; no substitute status value is specified.

## Dependencies

The following IDs are specification-local and opaque. They do not imply known implementations or distinct underlying functions.

| ID | Represented target formation | Explicit arguments | Result use |
|---|---|---|---|
| **F-U01** | Incoming link-register-derived value plus `0xFFE` | Zero, zero | No result consumed |
| **F-U02** | Fixed target calculated as `0x0840447D` | None | Signed 32-bit result shifted right by four for a byte-store address |
| **F-U03** | Fixed target calculated as `0x0840445D` | None | Captured as unsigned 32-bit |
| **F-U04** | Fixed target calculated as `0x0840438F` | Three continuation arguments described above | No result consumed |

The latter three target values follow from the represented arithmetic. Their executable mappings, instruction-state interpretation, calling conventions, and function boundaries are unresolved.

No dependency contract establishes preservation, ownership behavior, error reporting, or termination.

## Acceptance examples

These are conditional analytical criteria, not executed tests. All examples assume earlier required operations complete unless a failure is explicitly stipulated.

### Nonnull stack-derived pointer

Assume the stack-derived pointer is nonnull, both initial calls return, and all initial writes are permitted.

Expected behavior:

- Assign zero to Global A before F-U01.
- Invoke F-U01 with two zeros, then F-U02 without explicit arguments.
- Perform the result-derived one-byte zero store.
- Assign zero to Global B using its unresolved object width.
- Return the represented stack-derived 64-bit quantity.
- Do not invoke F-U03 or F-U04.

This example does not require either global assignment to be a byte store.

### Nonnegative address-producing result

Assume F-U02 returns `0x00001230`.

The result-derived store starts at `0x00000123` and writes one zero byte. Global B’s subsequent zero assignment remains a separate operation of unknown width.

### Matching starting addresses

Assume F-U02 returns `0x000001D0` and Global B’s represented starting address resolves to `0x1D`.

The function performs:

1. A one-byte zero store starting at `0x1D`.
2. A zero assignment to Global B starting at the same represented address.

The second operation’s extent remains unknown. Acceptance must not require two byte stores, assume identical extents, or replace the pair with one access.

### Unknown global width

Suppose Global B is independently established to occupy more than one byte.

That fact is compatible with this contract. The contract requires its zero assignment after the explicit byte store; it does not narrow the assignment to the first byte merely because the represented starting address is `0x1D`.

### Conditional continuation arithmetic

Assume the continuation is reached, the null pointer converts to integer zero, F-U03 returns `0xFFFFFFFF`, and the selector read returns five.

Expected continuation behavior:

- Read the selector byte at `0x1F`.
- Store byte zero at `0x23`.
- Store 32-bit zero at `0x75`.
- Invoke F-U04 with `0xFFFFFFF0`, eight, and `0x7FFFFFF8`.

Completion requires the environment to permit the 32-bit store at the unaligned address `0x75`.

### Failure at the result-derived store

Assume the result-derived byte store cannot complete and the environment does not resume execution beyond it.

Global B’s assignment, the early return, and the continuation are not reached. No local recovery or failure return is represented.

## Uncertainties

- **documented_only:** Actual ABI, pointer width, instruction state, and calling convention.
- **documented_only:** Types, write widths, full extents, and semantic purposes of both global objects.
- **documented_only:** Dependency effects, preservation guarantees, ownership behavior, and failure modes.
- **documented_only:** Whether the stack-derived pointer can compare equal to null in a valid invocation.
- **documented_only:** Negative signed-right-shift behavior and integer-to-pointer address interpretation.
- **documented_only:** Signed overflow and pointer-derived conversion semantics in the continuation.
- **documented_only:** Return-storage read timing and susceptibility to aliased writes or dependency effects.
- **documented_only:** Memory mappings, permissions, alignment requirements, endianness, and access-sensitive device behavior.
- **Inferred intent:** The zero assignments and zero stores resemble state clearing. They do not establish a complete reset operation.

## Evidence

This contract rests solely on the supplied static function representation. It supports the represented call sequence, explicit access widths, arithmetic expressions, conditional paths, and return expression.

The representation explicitly distinguishes a one-byte pointer store from an assignment to a global object whose type is absent. Consequently, Global B’s zero assignment has unknown width throughout this contract, including the same-starting-address example.

No runtime execution, memory trace, dependency inspection, or tests have been performed. Acceptance examples are deductions under their stated conditions, not observed results.
