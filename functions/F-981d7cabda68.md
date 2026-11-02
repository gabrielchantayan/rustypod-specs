<!-- Trusted provenance: F-981d7cabda68; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-981d7cabda68

## Purpose

F-981d7cabda68 performs a short sequence of memory updates and indirect control transfers:

1. Writes a fixed byte to a location derived from hidden incoming state.
2. Invokes a first dependency with two explicit inputs and a doubled form of the second input.
3. Uses part of that dependency’s result both as an address base and as data for a two-byte write.
4. Invokes a second, unresolved dependency with the two halves of the first result and a shifted value derived from hidden incoming state.

This is an operational description, not an established application-level purpose. The available static evidence does not establish an object type, protocol, allocation operation, dispatch scheme, or error-handling role.

The apparent interface is incomplete: two explicit inputs are visible, but execution also depends on incoming state not represented as ordinary parameters. A caller cannot safely invoke this operation using only the explicit inputs unless the surrounding execution convention supplies that state.

## Interface

### Explicit inputs

| Input | Established representation | Observable use |
|---|---|---|
| First input | Opaque 32-bit quantity | Forwarded as the first argument of the first dependency. |
| Second input | Integer represented as 32 bits in the recovered model | Forwarded as the second argument of the first dependency; also shifted left by one bit for its third argument. |

No meaning, permitted range, ownership, or pointer interpretation is established for the first input.

The signed integer representation of the second input matters when interpreting its left shift. For nonnegative values through 2,147,483,647 divided by two, rounded down—that is, 1,073,741,823—the mathematical doubled value fits in a signed 32-bit integer. Behavior outside that range is not settled by the supplied representation alone.

### Hidden incoming state

The operation additionally requires:

- A base value used to locate the initial byte write and to derive an argument for the final dependency.
- A continuation-associated base value used to derive the first dependency’s entry address.
- A callable destination for the final indirect transfer, whose origin is unresolved.

These are execution preconditions, not extra public parameters established by the evidence. Their initialization, calling convention, and preservation rules are undocumented.

### Outputs

There is no explicit result value in the recovered interface.

Observable results consist of:

- The initial byte write.
- Any effects of the first dependency.
- A subsequent two-byte write derived from that dependency’s result.
- Any effects of the final dependency.
- A normal return only if the intervening operations complete and the final dependency returns.

The apparent lack of a return value does not establish an ABI-level guarantee about residual return-register contents.

### Width and layout model

The visible operations distinguish these widths:

- One byte for the initial write.
- Two bytes for the later write.
- A 32-bit first explicit input.
- A recovered 32-bit integer second input.
- A 64-bit result from the first dependency, separated into low and high 32-bit halves.

The recovered representation uses 32-bit address-related conversions. The original target’s complete address model, integer semantics, byte order, and alignment requirements are not independently supplied.

## Behavior

### 1. Initial byte write

Before either visible dependency invocation, the operation writes the value 40, hexadecimal `0x28`, to the byte located six bytes beyond the hidden incoming base.

This is a single-byte store. The visible operation does not read the previous byte and does not explicitly write neighboring bytes.

Successful execution requires that the computed address be valid for writing. There is no visible null check, bounds check, object-type check, or protection check before this access.

The initial write precedes the first dependency invocation. Consequently, the first dependency may observe that write if it accesses the same storage. Its effects cannot be treated as independent of the byte update.

### 2. First dependency invocation

The first dependency is designated **F-INDIRECT-01** in this document. This is a document-local opaque identifier, not a recovered function identity.

Its entry address is derived by adding 3,860 bytes, hexadecimal `0xF14`, to the hidden continuation-associated base. The visible expression identifies a callable address at that displacement; it does not establish an intervening load of a function pointer from a table slot.

The arguments are supplied in this order:

1. The first explicit input, as a 32-bit quantity.
2. The second explicit input.
3. The second explicit input shifted left by one bit.

For a nonnegative second input whose doubled value remains representable as signed 32-bit, the third argument is twice the second argument.

For negative inputs or positive inputs whose doubling exceeds the signed range, the evidence does not establish a portable arithmetic contract. A literal interpretation of the recovered signed expression and the behavior of an underlying machine instruction need not coincide. Acceptance criteria for those cases require additional instruction-level evidence.

F-INDIRECT-01 produces a result represented as 64 bits. The subsequent behavior uses:

- Its low 32 bits as an address-related value and as a later argument.
- Its high 32 bits as another later argument.
- Its low 16 bits as data for a memory write.

No success flag, null-result check, or error classification is visible between this return and the following write.

### 3. Result-derived two-byte write

After F-INDIRECT-01 returns, the operation writes the low 16 bits of its result to a location four bytes beyond the address represented by the result’s low 32 bits.

The same returned bits therefore have two distinct roles:

- The low 32 bits determine the write’s address base.
- The low 16 bits determine the stored bit pattern.

The upper 32 bits do not visibly contribute to this address or stored value.

The destination access is two bytes wide. Its byte ordering is not established. On a little-endian target, the least significant byte would occupy the lower address; on a big-endian target, the ordering would be reversed.

The conversion to a two-byte signed representation does not establish a portable numeric interpretation for every possible low-half value. The observable intent of the recovered operation is a two-byte store derived from those low bits; exact original instruction semantics remain the authority where the recovered type conversions are ambiguous.

There is no visible validation that the destination:

- Refers to allocated or live storage.
- Is writable.
- Satisfies any required alignment.
- Lies within an object.
- Is disjoint from the initial byte-write location.
- Remains valid after effects performed by F-INDIRECT-01.

The write is unconditional along the visible normal path after the first dependency returns.

### 4. Final indirect invocation

The second dependency is designated **F-INDIRECT-02**, another document-local opaque identifier. Its callable destination is unresolved.

The visible arguments, in order, are:

1. The low 32 bits of the result from F-INDIRECT-01.
2. The high 32 bits of that result.
3. The hidden incoming base shifted right by 20 bits.

The first two arguments are recovered as integer-sized quantities. Their interpretation as signed values, unsigned values, pointers, handles, or packed fields is not established. Their bit-level origin is established.

The third argument’s exact value depends on right-shift semantics. For a nonnegative 32-bit base value, arithmetic and logical right shifts agree. For a value with its top bit set, they may differ:

- A logical shift produces a value from 0 through 4,095.
- An arithmetic signed shift produces a value from −2,048 through 2,047 across the full signed input range.

The evidence does not independently settle the original instruction’s treatment of this shift.

The result-derived two-byte write occurs before this invocation. F-INDIRECT-02 may therefore observe that write.

The recovered representation shows a return after this invocation, but the accompanying control-flow evidence reports an unresolved indirect transfer. A conventional call-and-return interpretation is provisional; whether the original transfer is a call, tail transfer, or another dispatch mechanism is documented_only.

## Side effects

### Directly established writes

| Order | Destination | Width | Value |
|---|---|---:|---|
| First | Hidden incoming base plus 6 bytes | 1 byte | `0x28` |
| After F-INDIRECT-01 returns | Address represented by its low 32 result bits, plus 4 bytes | 2 bytes | Low 16 result bits |

The visible body contains no explicit read of either destination before writing it.

This does not imply that dependencies perform no reads or writes. Both dependencies are opaque and may access the same storage, other memory, or external state.

### Ordering and aliasing

The established normal-path order is:

initial byte write → F-INDIRECT-01 → result-derived two-byte write → F-INDIRECT-02.

No disjointness condition is established for the two destinations. If they overlap, the later write can replace some or all of the earlier byte’s effect. F-INDIRECT-01 may also modify the initial destination before the second write occurs.

This sequence establishes execution order, not atomicity, synchronization, or visibility guarantees to other threads or devices.

### Ownership and lifetime

No explicit allocation, release, ownership transfer, or reference-count operation is visible.

In particular:

- The first dependency’s result is not established to be a newly allocated pointer.
- The result-derived write does not prove ownership of its destination.
- Forwarding result halves to the final dependency does not prove ownership transfer.
- Neither dependency may be assumed to preserve global state or to retain or release memory.

### Errors and interruption

There is no visible recovery or rollback path.

An invalid write address, invalid callable address, alignment violation, or failure inside a dependency may prevent later steps from executing. Platform-specific fault behavior is unknown.

If F-INDIRECT-01 does not return, the result-derived write and final invocation are not reached. If the second write fails, the final invocation is not reached along the visible sequence. Earlier effects are not visibly undone.

## Dependencies

| Opaque ID | Established interaction | Not established |
|---|---|---|
| F-INDIRECT-01 | Entry address derived from hidden continuation-associated state plus `0xF14`; receives three arguments; returns a represented 64-bit value. | Identity, semantic purpose, full ABI, effects, ownership rules, result validity, failure convention, state preservation. |
| F-INDIRECT-02 | Unresolved destination; receives the low and high result halves and a shifted hidden-base value after the two-byte write. | Target selection, semantic purpose, complete control-transfer form, return behavior, effects, ownership rules, state preservation. |

These labels distinguish invocation sites only. They do not prove the destinations are different functions or exclude either destination from resolving to F-981d7cabda68.

## Acceptance examples

These are conditional static examples, not observed executions.

### Ordinary arithmetic and result splitting

Assume:

- The second explicit input is 7.
- Both callable destinations are valid and return normally.
- Address calculations and writes are valid.
- F-INDIRECT-01 returns the 64-bit pattern `0x89ABCDEF00002000`.

Then:

- The initial byte write precedes F-INDIRECT-01.
- F-INDIRECT-01 receives the unchanged first input, 7, and 14.
- The later two-byte write targets address `0x00002004`.
- That write stores the two-byte pattern `0x2000`.
- F-INDIRECT-02 receives `0x00002000` and `0x89ABCDEF` as its first two 32-bit argument patterns.

The third final argument still depends on the hidden base and shift semantics.

### Zero returned result

Assume F-INDIRECT-01 returns zero.

The visible operation attempts a two-byte zero write at address 4. It does not visibly treat zero as failure or skip the write. Whether address 4 is writable is environment-dependent; acceptance must not require successful completion merely because the returned value is zero.

### Overlapping writes

Assume the hidden base is `0x00001000` and F-INDIRECT-01 returns a value whose low 32 bits are `0x00001002`.

The initial write targets `0x00001006`. The later two-byte write also begins at `0x00001006` and stores the pattern `0x1002`. Subject to successful accesses, the later store replaces the initially written byte at that address. Its final byte value depends on endianness.

### Shift boundary

For a second input of 1,073,741,823, the mathematical third argument to F-INDIRECT-01 is 2,147,483,646 and remains signed-32-bit representable.

For 1,073,741,824, that mathematical doubling is not signed-32-bit representable. A wraparound expectation is not established by this evidence.

### First dependency does not return

If F-INDIRECT-01 terminates execution or transfers control without returning, the initial byte write has already been attempted, but neither the result-derived write nor F-INDIRECT-02 is reached through the visible continuation.

## Uncertainties

The following remain documented_only:

- The application-level purpose and data structure layout.
- The mechanism supplying hidden incoming state.
- Whether the hidden base remains usable across dependency effects.
- The original address width and exact address-overflow behavior.
- Left-shift behavior for negative or overflowing inputs.
- Right-shift behavior for negative hidden-base values.
- Endianness and alignment requirements.
- Exact signed-conversion behavior where recovered types are ambiguous.
- Both dependencies’ identities, effects, and memory ownership contracts.
- The origin and control-transfer semantics of the final destination.
- Any concurrency, memory-mapped I/O, or fault-handling requirements.

No inferred object model or guessed dependency behavior is required by this contract.

## Evidence

**Established static evidence:** one byte write of `0x28`; one indirectly addressed invocation with three arguments; a represented 64-bit result; a two-byte write using its low bits; a final unresolved indirect invocation with three arguments; and the visible ordering described above.

**Inferred intent:** the fixed byte may represent a state update, and the result-derived address may refer to a structured object. Neither interpretation is established or used as a requirement.

**Evidence limits:** the supplied recovered representation contains incomplete incoming-state and control-flow recovery. Its type expressions support the width distinctions described here but do not independently prove every original machine-level arithmetic or ABI detail.

No runtime tests have been performed.
