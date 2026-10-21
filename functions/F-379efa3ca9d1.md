<!-- Trusted provenance: F-379efa3ca9d1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-379efa3ca9d1

## Purpose

F-379efa3ca9d1 selects a word within a caller-supplied address region and sets the bits specified by an input mask. Its direct update is a read, inclusive bitwise OR, and write back to the same location.

A signed selector greater than 12 causes a call to F-a817d8d58c10 before any target-word access. The supplied representation includes a non-return annotation associated with that path, but the dependency’s implementation is unavailable. If the dependency returns normally, the displayed continuation performs the update; it does not explicitly reject the selector or return early.

The likely intent is to maintain flags in one of thirteen regularly spaced records. That interpretation is inferred, not established: negative selectors are not rejected, and no allocation, record type, or region length is supplied.

## Interface

The function has three inputs and no explicit result value:

| Input | Represented type | Behavioral role |
|---|---|---|
| Base address | Signed integer | Starting value for address calculation; not represented as a typed pointer. |
| Selector | Signed integer | Controls the dependency call and selects an eight-byte displacement. |
| Mask | Unsigned integer | Supplies bits to set in the selected word. |

**Width qualification.** The supplied representation uses integer types without explicit bit widths or an accompanying ABI definition. A 32-bit interpretation is used below for concrete word values and address examples. Under that interpretation, the base and selector are signed 32-bit values, the mask is an unsigned 32-bit value, and the accessed word is four bytes. Those widths are not independently established by the fragment alone.

The target address is calculated in two stages:

1. Add eight times the selector to the base address.
2. Add a further byte displacement of 96, hexadecimal `0x60`, to obtain the word address.

Thus, where all arithmetic is defined, the target displacement relative to the base is 96 plus eight times the selector.

The eight-byte stride does not establish that an entire eight-byte record is read or written. Only the selected unsigned word is explicitly accessed. Under the 32-bit interpretation, the four bytes immediately following that word are not part of the direct update.

### Preconditions for defined completion

The caller must supply an address calculation that is valid under the actual execution model and a target location that supports the represented word read and write. Relevant requirements include:

- Valid arithmetic for the selector multiplication and address additions.
- A meaningful integer-to-address conversion.
- Sufficient readable and writable storage at the selected address.
- Any alignment required by the target.
- Appropriate synchronization if the storage can be accessed concurrently.

The function does not explicitly check these conditions. It also does not accept a region length or establish that the target belongs to the caller’s intended object.

## Behavior

### 1. Signed selector check

The function first compares the selector with 12 using signed ordering.

- Selectors of 12 or less do not call F-a817d8d58c10.
- Selectors greater than 12 call F-a817d8d58c10.
- Negative selectors take the first path; there is no lower-bound check.

This is not an unsigned range check. A negative selector must not be treated as a large positive value for purposes of deciding whether to call the dependency.

### 2. Conditional dependency call

When the selector exceeds 12, the dependency is called before the displayed target-address calculation and before either target-word access. No explicit arguments are supplied to that call.

The source annotation indicates non-returning behavior, but does not establish the mechanism or the dependency’s effects. Possible termination, exception behavior, state modification, and resource handling remain unknown.

If the dependency does not return, the subsequent word update is not reached. If it returns normally, the displayed continuation proceeds using the supplied base, selector, and mask. There is no additional bounds check.

Because the dependency executes first, it may affect the memory later read by the update. The post-call update must therefore not be specified as necessarily using the word’s value from function entry.

### 3. Address selection

For execution that reaches the update, the selector contributes an eight-byte stride and the fixed displacement contributes 96 bytes.

Representative mathematical displacements are:

| Selector | Displacement from base |
|---:|---:|
| −13 | −8 bytes |
| −12 | 0 bytes |
| −1 | 88 bytes |
| 0 | 96 bytes |
| 1 | 104 bytes |
| 12 | 192 bytes |
| 13 | 200 bytes, reached only if the dependency returns |

These values describe arithmetic displacement, not permission to access the resulting location.

The represented signed arithmetic does not establish modular address wrapping. Under a 32-bit signed interpretation, multiplication by eight is representable only for selectors from −268,435,456 through 268,435,455. A negative selector below that interval bypasses the dependency check but still exceeds the representable multiplication range. Addition to the base can overflow independently, and the subsequent fixed displacement introduces another address-arithmetic constraint.

No reliable fault, wraparound, or recovery behavior is established for those cases.

### 4. Word update

At the selected address, the function reads one unsigned word, forms its inclusive bitwise OR with the mask, and writes the result back to that address.

For every bit position in the accessed word:

- A set mask bit makes the corresponding stored bit one.
- A clear mask bit preserves the corresponding bit from the read value.

Consequently, the direct operation sets bits but does not clear them. With stable ordinary memory and no intervening modification, repeating the same mask produces the same stored value after the first application.

A zero mask does not bypass the operation. The displayed behavior still includes a read and a write, even though the resulting numeric value equals the value read. Similarly, a mask whose bits are already set does not introduce an explicit no-write path.

This is a read-modify-write description, not evidence of an atomic operation. No lock, atomic primitive, retry, or synchronization mechanism is shown.

### 5. Completion

After the write, the function returns without an explicit result value. The prior word, resulting word, and success status are not returned through the interface.

No rollback or local error recovery is shown if execution fails before completion.

## Side effects

On the path that skips the dependency and completes normally, the explicit memory effects are one target-word read followed by one write to the same address. No other explicit data-memory access, allocation, release, logging operation, or global-variable access appears in the fragment.

That limited statement is not an ownership guarantee:

- The function does not explicitly acquire or release the target storage.
- It does not establish ownership of the base region.
- No transfer of ownership is expressed.
- F-a817d8d58c10 may have additional effects that cannot be constrained from the available evidence.

Under the four-byte word interpretation, the direct write covers four bytes. The representation does not specify byte order. A numeric OR result can be stated without choosing an in-memory byte order, but its byte-level encoding cannot.

Concurrent modification may be lost between the read and write. For example, a bit set by another participant after this function’s read can be overwritten if it was absent from both the read value and the mask. Actual concurrent-access legality depends on the execution model and memory type.

For device or otherwise specially observable memory, the read and write may themselves have effects even when the numeric value is unchanged. The fragment does not establish the access semantics for such memory.

## Dependencies

### F-a817d8d58c10

- **Invocation condition:** signed selector greater than 12.
- **Explicit arguments:** none shown.
- **Position:** before the displayed address calculation and target-word read/write.
- **Return value:** none consumed.
- **Annotation evidence:** the supplied representation includes a non-return warning.
- **Unknowns:** actual return behavior, termination mechanism, error signaling, global-state effects, memory mutation, and resource ownership or release.

A strict rule that selectors above 12 can never reach the update requires independently established non-return behavior from this dependency. Without that evidence, both non-return and normal-return continuations must remain documented.

## Acceptance examples

These examples are conditional behavioral expectations, not executed tests. Numeric examples assume 32-bit words, valid address arithmetic, suitable storage, and no intervening memory modification unless stated otherwise.

1. **Ordinary bit setting.** With selector 0, an initial word of `0x00000005` at base plus 96, and mask `0x0000000A`, the function skips the dependency and writes `0x0000000F` to that location.

2. **Upper selector boundary.** Selector 12 skips the dependency. The selected word is at base plus 192. The comparison does not classify this selector as exceeding the limit.

3. **Zero mask.** With selector 1, word `0x12345678` at base plus 104, and mask zero, the numeric word remains `0x12345678`. The displayed operation still reads and writes the location.

4. **All bits requested.** With selector 0 and mask `0xFFFFFFFF`, the resulting word is `0xFFFFFFFF`, regardless of its prior value.

5. **Negative selector.** Selector −1 skips the dependency and selects base plus 88, provided the arithmetic and access are valid. Rejecting it solely because it is negative would add behavior not present in the fragment.

6. **Selection at the base.** Selector −12 skips the dependency and selects the word at the base itself. The fixed 96-byte displacement does not imply that all selected addresses lie beyond the base.

7. **Above-limit non-return.** With selector 13, if F-a817d8d58c10 does not return, the displayed target-word read and write are not reached.

8. **Above-limit normal return.** With selector 13, if the dependency returns normally and the resulting access is valid, the function updates the word at base plus 200. Any dependency-induced change to that word is reflected in the subsequent read.

## Uncertainties

The following are **documented_only**:

- Exact integer widths, pointer width, ABI, byte order, and alignment rules.
- Defined behavior of signed overflow and integer-derived addresses in the actual execution environment.
- The dependency’s non-return guarantee and all of its side effects.
- The target allocation’s extent, lifetime, ownership, and record layout.
- Whether the memory is ordinary, shared, volatile, atomic, or device-backed.
- Fault behavior for inaccessible, misaligned, or otherwise invalid addresses.
- Whether negative selectors are intentional supported inputs or unchecked misuse.

The thirteen-record interpretation is inferred intent only. Neither a thirteen-record allocation nor a nonnegative-selector requirement is established.

## Evidence

Established static evidence consists of the signed comparison against 12, the conditional call to F-a817d8d58c10, the eight-byte selector stride, the fixed 96-byte displacement, the unsigned-word OR update, and the value-less return.

The supplied representation also contains a non-return annotation; the dependency body is unavailable. That annotation does not independently establish termination mechanics, memory effects, or ownership behavior.

No runtime tests have been performed. Acceptance examples are derived from the displayed operations under their stated assumptions.
