<!-- Trusted provenance: F-2110eeefadda; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2110eeefadda

## Purpose

F-2110eeefadda performs a sequence of memory writes derived from explicit arguments, hidden entry-state values, fixed literals, and global memory. It then invokes a dynamically selected target with four constructed arguments.

The supplied representation establishes a state-mutating routine, not a self-contained operation on its four declared arguments. Correct invocation requires additional register state, valid writable addresses, readable global state, and a valid indirect-call target.

**Inferred intent:** some writes resemble saved-context construction and low-level initialization. The evidence does not establish a particular device, execution mode, data structure, or initialization protocol. Those interpretations are not requirements of this contract.

## Interface

### Explicit inputs

| Input | Represented width | Established use |
|---|---:|---|
| Argument 1 | 32 bits | Written to the first word of a hidden destination region. |
| Argument 2 | 32 bits | Written to the second word of that region. |
| Argument 3 | 32 bits | No use appears in the supplied body. |
| Argument 4 | Signed 32-bit integer | Saved in the destination region and used in address arithmetic. |

Argument 3 being unused does not establish an alternative calling convention or permission to omit it.

### Hidden entry inputs

The recovered body depends on four values not supplied through the declared parameters:

- A pointer to a destination region containing at least five consecutive 32-bit words.
- Two additional 32-bit values saved into that region.
- A value used as the base for the final indirect-call target.

The source of these values is not established. They are interface prerequisites, not values this routine initializes before use.

### Global inputs and destinations

The routine accesses:

- A global pointer-valued location.
- A global 16-bit destination.
- The destination selected by the global pointer.
- A global location associated with address `0x000000FD`.
- Three other global locations used for literal writes or an address-producing read.

These global descriptions identify separate roles in the recovered representation. They do **not** establish separate, nonoverlapping storage. The supplied warning explicitly permits overlapping global symbols.

### Outputs

There is no explicit return value. Observable results consist of:

1. Memory writes made before the indirect invocation.
2. The invocation itself and its arguments.
3. Any additional effects of the invoked target.
4. Normal return if that target returns normally and execution reaches the represented return.

No allocation, deallocation, ownership transfer, status result, or explicit error result is established.

### Width model

The representation uses 16-bit halfwords, 32-bit words, and pointer-to-integer conversions consistent with a 32-bit address model. The address calculations below use that model where explicitly stated.

This does not establish that every signed arithmetic operation has portable, defined language semantics. Overflow, signed right shift, alignment, and pointer/integer conversions require separate qualification.

## Behavior

The following sequence describes the order exposed by the supplied representation. It is not an independently verified machine-instruction trace. In particular, access ordering inside a single expression and the identities of overlapping global locations remain unresolved.

### 1. Save five words through the hidden destination pointer

The routine writes five consecutive 32-bit words, in this order:

| Byte offset from hidden destination | Value written |
|---:|---|
| `0x00` | Argument 1 |
| `0x04` | Argument 2 |
| `0x08` | Argument 4, represented in the 32-bit destination word |
| `0x0C` | First hidden saved value |
| `0x10` | Second hidden saved value |

Argument 3 is not saved.

The represented sequence does not advance or replace the hidden destination pointer. It merely uses offsets from it. There is no bounds check, null check, or alignment check before these stores.

Aliasing is significant: if this destination overlaps global storage used later, the saved words can change subsequent reads.

### 2. Store part of a pointer, then write through the global pointer

The routine reads the global pointer value, extracts its low-order 16 bits, and writes those bits to the global 16-bit destination.

It then performs a 16-bit write through the global pointer, using the value read from the global 16-bit destination.

Two distinctions are required:

- The extracted value comes from the **pointer representation**, not from the memory to which it points.
- The pointer and the 16-bit destination are accessed again in the second operation. The representation does not establish a retained pointer value or retained halfword value across the two statements.

Under nonaliasing, stable-memory conditions, both halfword destinations receive the low 16 bits of the same pointer value. That simplification is conditional. If the first halfword write changes the stored global pointer, the second operation can select a different address.

The relative evaluation order of the pointer read and halfword-value read within the second assignment is not established by the supplied representation.

### 3. Construct and store an address derived from Argument 4

The routine forms an address by adding `0xF4` to Argument 4.

It then:

1. Stores that address as a pointer-sized value at the address formed by adding `0x134` to Argument 4.
2. Writes the low 16 bits of the constructed address to the constructed address itself.

Under the represented 32-bit pointer model, the first write occupies four bytes and the second occupies two bytes.

The second write stores address bits. It does not copy data from another object.

### 4. Perform a self-address word write

The routine constructs another address by multiplying Argument 4 by `0x10` and adding `0x11`. It writes that address value into the 32-bit word located at that address.

If address arithmetic is interpreted modulo \(2^{32}\), this destination has low four bits equal to `1`. The resulting word access is therefore not naturally four-byte aligned.

Successful execution requires a platform and mapping that permit this access, or other execution semantics that explain it. The representation contains no alignment repair or fallback.

Argument 4 is represented as signed. Consequently, the expression alone does not establish defined portable behavior for values whose multiplication or addition overflows a signed 32-bit integer.

### 5. Derive another address and write the same halfword twice

Starting with the address formed by adding `0xF4` to Argument 4, the routine:

- Converts that address to a signed integer.
- Multiplies the integer by two.
- Right-shifts the result by ten bit positions.
- Uses the resulting value as a pointer to a 16-bit destination.

It writes the low 16 bits of that resulting address to the destination, then repeats the same halfword write to the same destination.

The duplicate stores are present in the supplied representation. This contract does not collapse them into one store. They may be distinguishable when the destination is device memory or otherwise has access-sensitive behavior. Volatile qualification and the exact machine-access trace are not established.

The calculation is not unconditionally equivalent to an unsigned right shift by nine. That simplification fails to account for signed conversion, multiplication overflow, and negative-value right-shift semantics.

### 6. Write a fixed halfword relative to Argument 4

The routine writes the 16-bit value `0x0050` at Argument 4 plus `0xF6`.

When the address arithmetic is valid, this is the halfword immediately following the halfword written at Argument 4 plus `0xF4`. Together they occupy four consecutive bytes, but their byte order is not established because target endianness is not supplied.

The pointer-valued write at Argument 4 plus `0x134` is a separate destination.

### 7. Perform fixed-global assignments

The routine next performs three assignments, in order:

1. Write `0xF430` to the global location associated with `0x000000FD`.
2. Write `0x0897F430` to another global location.
3. Write zero to a third global location.

The supplied text does not expose declarations sufficient to fix all three access widths. In particular, the short literal `0xF430` does not by itself establish a 16-bit store.

No interpretation as a vector table, hardware register bank, or control structure is required. Such interpretations would need additional evidence.

### 8. Read an address-producing global and write through it

The routine reads another global value, converts it to an unsigned integer, and uses that integer as a pointer to 32-bit words.

It writes the 32-bit value `0x0897F430` through that pointer.

The value read here is live state at this point in execution. Earlier writes may affect it if storage overlaps. It must not be assumed to equal its entry value.

### 9. Invoke the dynamically resolved dependency

The final target is formed by adding `0x0897F314` to the hidden target-base value.

The routine invokes that target with these four arguments:

| Position | Constructed argument |
|---:|---|
| 1 | The address-producing pointer from the preceding stage, advanced by four 32-bit elements, converted to signed integer, then multiplied by `0x01000000` |
| 2 | `0` |
| 3 | `0x5000` |
| 4 | `0x56` |

Advancing a pointer to 32-bit words by four means a byte displacement of `0x10`, not four bytes.

Under modulo-\(2^{32}\) arithmetic, the first argument retains only the low eight bits of the advanced address, moved into bits 31 through 24. This is a conditional arithmetic interpretation, not proof that the represented signed multiplication is defined for every input.

No result from the target is consumed. If it returns normally, the routine reaches its represented return without further explicit memory writes.

## Side effects

The routine makes direct writes to caller-associated memory, computed addresses, global state, and a global-selected destination. It also transfers control to an unresolved target.

It provides no checks for:

- Writable or mapped destinations.
- Required access alignment.
- Arithmetic overflow.
- Valid global pointer contents.
- A callable final target.
- Overlap between saved-state storage, computed destinations, and globals.

There is no represented rollback. If a later access faults or the target does not return, earlier completed writes are not undone by any shown mechanism.

No memory ownership policy is established. The routine visibly neither allocates nor frees memory. The indirect target may have additional memory effects, including ownership-related effects; these cannot be inferred from the caller.

A conforming description must preserve alias-sensitive behavior. Replacing a later global read with an entry-time snapshot, merging the repeated halfword writes, or treating all global roles as disjoint changes behavior unless supported by additional evidence.

## Dependencies

**F-indirect-01** is a document-local opaque designation for the unresolved dynamic dependency. It is not a recovered identity or an entry in the supplied dependency map.

Its target is selected using the hidden target-base value plus `0x0897F314`. Its four argument expressions are specified above.

Unknown properties include:

- The target’s actual identity and calling convention.
- Whether it returns.
- Whether it preserves registers, globals, or pointed-to memory.
- Whether it allocates, releases, or takes ownership of memory.
- Whether it transfers control elsewhere.

The supplied representation reports uncertainty in recovering the terminal control transfer and models it as an indirect call. Thus, invocation followed by a possible return is the represented behavior; the precise underlying control-transfer mechanism remains unresolved.

No other callee appears in the supplied body.

## Acceptance examples

These examples are conditional specifications derived from static evidence. **No runtime tests have been performed.**

### Saved-word ordering

Given a valid, nonoverlapping hidden destination, Argument 1 equal to `0x11223344`, Argument 2 equal to `0x55667788`, Argument 4 equal to `0x00001000`, and hidden saved values `0xAABBCCDD` and `0x01020304`:

- The five destination words must receive those values in the order listed in the interface behavior.
- Argument 3 must not replace any of them.
- This condition describes word values, not byte order.

### Pointer-bit extraction

Given a stable global pointer value of `0x12345678`, writable halfword destinations, and no relevant aliasing:

- The global 16-bit destination receives `0x5678`.
- The halfword at address `0x12345678` receives `0x5678`.

If the first destination overlaps the stored pointer, this expected second address is not applicable; the later pointer read must determine it.

### Computed destinations without arithmetic overflow

Given Argument 4 equal to `0x00001000`, valid mappings for all affected destinations, and ordinary positive signed arithmetic:

- Address `0x00001134` receives pointer value `0x000010F4`.
- Address `0x000010F4` receives halfword `0x10F4`.
- Address `0x00010011` receives word `0x00010011`, provided the platform permits that unaligned access.
- The doubled-address-and-shift calculation produces address `0x00000008`, which receives halfword `0x0008` twice.
- Address `0x000010F6` subsequently receives halfword `0x0050`.

These conditions do not establish that such low addresses are usable on any particular system.

### Final-call construction under modular arithmetic

Given that the address-producing global supplies `0x00002030`, and assuming 32-bit modular arithmetic for the first call argument:

- The preceding indirect word write targets `0x00002030`.
- Advancing four word elements produces `0x00002040`.
- The final call receives arguments `0x40000000`, `0`, `0x5000`, and `0x56`.

This example does not assert defined signed-overflow behavior in a portable source-language implementation.

### Failure before invocation

If the self-address word store faults, the represented execution does not reach the later halfword writes, fixed-global assignments, or final invocation. Earlier completed stores remain outside any shown rollback mechanism.

## Uncertainties

The following are **documented_only**:

- Initialization and provenance of the hidden entry-state values.
- Exact widths and overlap relationships of incompletely declared globals.
- Target endianness, memory map, permissions, and alignment behavior.
- Whether repeated represented stores correspond to distinct machine-visible accesses.
- Signed overflow, pointer-conversion, and negative right-shift behavior on the actual target.
- Concurrent mutation or device semantics affecting global reads and writes.
- The identity, effects, ownership rules, and return behavior of F-indirect-01.
- Whether the terminal transfer is accurately characterized as an ordinary returning call.
- The domain-level purpose of the saved words and fixed-global writes.

These unknowns must not be resolved by assuming conventional structure layouts, benign callbacks, disjoint globals, or harmless duplicate writes.

## Evidence

**Established static evidence:** the supplied representation contains five initial word stores, pointer-bit extraction, pointer-derived writes, a self-address word store, two repeated halfword stores, fixed-global assignments, a global-selected word store, and a dynamically selected invocation.

**Inferred interpretation:** a 32-bit pointer model explains the represented word layout and four-element pointer advance. Context saving or low-level initialization is plausible but unproven.

**Evidence limits:** only the supplied recovered representation and its warnings were available. No declarations, surrounding callers, dependency implementation, instruction-level validation, or runtime observations were provided. No runtime tests have been performed.
