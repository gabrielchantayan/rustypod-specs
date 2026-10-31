<!-- Trusted provenance: F-5059efbcd70f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5059efbcd70f

## Purpose

F-5059efbcd70f coordinates an initial dependency call, an operation on a caller-supplied object, an optional additional call, and conditional propagation of a 16-bit field into an object obtained from another dependency.

The function returns the 32-bit result produced by F-b2ac012cd863. That result is retained while subsequent calls and conditional memory operations occur; it is not replaced by their results.

The apparent intent is to populate an unset field in a context-like object from a field in the supplied object when the second input is nonzero. This interpretation is **inferred intent**, not an established object model. In particular, the function obtains the context-like object separately for the zero check and for the eventual write. Those calls are not established to return the same address.

## Interface

### Inputs

The function has two inputs represented as 32-bit integer values in the supplied source:

- **Object address:** Used both as an argument to dependencies and as the base address for direct memory access. The function accesses a 16-bit field at byte offset `0xFA` from this address.
- **Control value:** Passed unchanged to F-b2ac012cd863 and subsequently tested for equality with zero. Every nonzero value selects the same local conditional behavior, including values represented as negative signed integers. A dependency may nevertheless distinguish different nonzero values.

The supplied source does not establish a public calling convention, a concrete object type, or the target’s complete address-arithmetic rules. The address-shaped first input must not be interpreted as an ordinary numerical quantity merely because it is represented by an integer type.

### Output

The output is a 32-bit value with no established interpretation. It is exactly the value returned by F-b2ac012cd863, provided the function reaches its normal return.

No success code, failure code, Boolean conversion, or interpretation of that value is established locally.

### Memory layout used directly

Only these fields are directly accessed:

| Base | Byte offset | Access width | Use |
|---|---:|---:|---|
| Supplied object address | `0xFA` | 16 bits | Initial zero test and, conditionally, a later copy source |
| Address returned by F-fdd8589eca2d during the conditional check | `0x80` | 16 bits | Zero test |
| Address returned by a subsequent F-fdd8589eca2d call | `0x80` | 16 bits | Conditional write destination |

No surrounding structure layout, allocation size, alignment guarantee, or field meaning is established.

The zero tests are insensitive to whether the 16-bit fields are interpreted as signed or unsigned: only the distinction between zero and nonzero matters. The eventual copy transfers all 16 bits without widening, narrowing, normalization, or arithmetic.

## Behavior

The following describes normal-return execution. If a dependency does not return normally, or a required memory access cannot complete, the supplied source does not establish recovery behavior.

### Initial calls and retained result

First, F-fdd8589eca2d is called with no explicit arguments.

Its 32-bit result is passed as the first argument to F-3e5212464798. The other two arguments are the supplied object address and a zero-valued literal, in that order. Any result from F-3e5212464798 is ignored.

Next, F-b2ac012cd863 is called with the supplied object address and the original control value. Its 32-bit result is retained for the eventual return.

These calls occur regardless of whether the control value is zero. No field at offset `0xFA` is read directly by F-5059efbcd70f before these calls.

### Optional call

If the control value is nonzero, F-5c5958cfd5fe is called with no explicit arguments. Its result, if any, is ignored.

If the control value is zero, that call is skipped.

The optional call occurs before the function’s first direct read of the supplied object’s 16-bit field. Consequently, any changes that earlier dependencies make to relevant memory can affect the later tests.

### Conditional field propagation

After the optional call, the function reads the 16-bit field at offset `0xFA` from the supplied object.

This read occurs even when the control value is zero. A zero control value disables propagation but does not make the object address unnecessary.

Further processing depends on the following ordered conditions:

1. The value just read from the supplied object must be nonzero.
2. The control value must be nonzero.
3. A newly obtained destination field must be zero.

If the source field is zero, no further dependency is called by this function.

If the source field is nonzero but the control value is zero, no further dependency is called by this function.

If both are nonzero, F-fdd8589eca2d is called again. The function then reads the 16-bit field at offset `0x80` from the address returned by that call.

If this checked field is nonzero, propagation stops without a direct write.

If the checked field is zero:

- F-fdd8589eca2d is called once more.
- The 16-bit source field at offset `0xFA` is read again from the supplied object.
- That newly read 16-bit value is written to offset `0x80` of the address returned by the most recent call.

The last write is therefore not necessarily made to the object whose field was checked. The function does not compare the two returned addresses and does not check the final destination’s existing field value.

The copy also does not reuse the value obtained by the initial source-field test. A dependency may change that field between the test and the copy. In that case, the later value is copied, including zero.

### Return

After any applicable propagation, the function returns the retained result from F-b2ac012cd863.

The source does not establish an early return based on that result. A value that a caller might interpret as failure does not locally suppress the optional call, field checks, or write.

## Side effects

### Direct memory effects

F-5059efbcd70f performs:

- One 16-bit read from the supplied object on every normal-return path.
- One additional 16-bit read from a dependency-provided address when the source test and control test both permit it.
- One additional 16-bit source read and one 16-bit destination write when the checked destination field is zero.

There is at most one direct memory write in the function.

That write can overwrite a nonzero field if the final F-fdd8589eca2d call returns an address different from the checked address, or if relevant memory changes between the check and the write. The function provides no local check-and-set atomicity.

### Dependency effects

All dependency effects are additional to these direct accesses and are unknown from the supplied evidence. No dependency is established to preserve globals, preserve either object’s contents, return a stable address, or avoid allocation or release.

The retained operation result is independent of later dependency return values. Memory state is not similarly preserved: subsequent behavior uses fresh reads after intervening calls.

### Ownership and lifetime

The function contains no explicit allocation, deallocation, reference-count adjustment, or ownership transfer.

This does not establish that its dependencies perform none of those actions. For normal completion, each address must remain usable when it is actually accessed, including after intervening dependency calls.

### Preconditions and errors

There are no explicit null checks, bounds checks, alignment checks, or error-handling branches.

At minimum, normal completion requires:

- A readable 16-bit field at the supplied object’s offset `0xFA` when the unconditional read occurs.
- A readable 16-bit field at offset `0x80` of the conditional check address, if that path is taken.
- A readable source field and writable final destination field when copying occurs.
- Normal return from every dependency reached on that path.

The source establishes no behavior for invalid addresses, unsuitable alignment, expired objects, or address-offset overflow. It also establishes no local rollback if an earlier dependency has already changed state before a later call or access fails.

## Dependencies

| Opaque ID | Invocation and established role |
|---|---|
| F-fdd8589eca2d | Called first without explicit arguments; its result is forwarded to F-3e5212464798. Called again only when the source field and control value are nonzero; that result supplies the check address. Called a third time only when the checked field is zero; that result supplies the write address. |
| F-3e5212464798 | Called once with the initial F-fdd8589eca2d result, the supplied object address, and literal zero. Its return value is unused. |
| F-b2ac012cd863 | Called once with both original inputs. Supplies the function’s returned 32-bit value. |
| F-5c5958cfd5fe | Called once if and only if the control value is nonzero, assuming execution reaches that decision. Receives no explicit arguments; its result is unused. |

F-fdd8589eca2d is called one, two, or three times on a normal-return path. Nothing establishes that those calls return equal values or share an ownership convention.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not observed runtime results. Unless an example explicitly allows mutation, assume all reached dependencies return normally and do not alter the relevant fields.

### Zero control still requires a source read

Given a zero control value:

- The initial F-fdd8589eca2d call, F-3e5212464798 call, and F-b2ac012cd863 call occur in that order.
- F-5c5958cfd5fe is not called.
- The supplied object’s 16-bit field is read.
- No conditional destination lookup or direct write occurs.
- The result from F-b2ac012cd863 is returned.

This holds whether the source field is zero or nonzero.

### Nonzero control with a zero source

Given a nonzero control value and a source field that is zero when tested:

- F-5c5958cfd5fe is called after F-b2ac012cd863.
- The source field is then read.
- No additional F-fdd8589eca2d call occurs.
- No direct write occurs.

### Nonzero checked destination

Given a nonzero control value, a nonzero source field, and a nonzero field at offset `0x80` of the second F-fdd8589eca2d result:

- The destination field is read.
- No third F-fdd8589eca2d call occurs.
- The source is not reread for copying.
- No direct write occurs.

### Stable destination with an unset field

Given a source field of `0xBEEF`, nonzero control, and the same valid destination address returned by both conditional lookups, with its field initially zero:

- The checked destination field is zero.
- The source is read again.
- `0xBEEF` is written as a 16-bit value to the destination field.
- The retained operation result is returned unchanged.

### Different check and write addresses

Given that the second F-fdd8589eca2d call returns one address whose field is zero, while the third returns a different address whose field is `0x1234`:

- The write targets the third call’s address.
- Its existing `0x1234` value is not checked locally.
- The current source value overwrites that field.

### Source changes before copying

Given an initially nonzero source field and a zero checked destination field, suppose the third F-fdd8589eca2d call changes the source field to zero:

- The function rereads zero from the source.
- It writes zero to the final destination.
- The earlier nonzero test does not prevent this write.

### Return value does not gate propagation

Given any 32-bit result from F-b2ac012cd863, including zero or a high-bit-set value:

- Subsequent local decisions still depend on the control value and memory fields, not that result.
- On normal completion, the same 32-bit result is returned.

## Uncertainties

The following are **documented_only**:

- Semantic names and types of the objects and fields.
- The meaning of the control value beyond its local zero/nonzero test.
- The meaning of the returned 32-bit value.
- Dependency implementations, failure behavior, memory effects, and ownership rules.
- Whether repeated F-fdd8589eca2d calls normally return the same address.
- Exact ABI, endianness, alignment requirements, and address-overflow behavior.
- Whether accessed memory has volatile, device, concurrent, or other special semantics not represented by the supplied source.
- Any synchronization or lifetime guarantees supplied by callers or dependencies.

The “populate an unset field” interpretation must not replace the established behavior: the final destination is not rechecked, and the copied source is freshly read.

## Evidence

The established behavior above comes solely from the supplied static source representation: call ordering, explicit arguments, zero-test short-circuiting, byte offsets, 16-bit accesses, conditional write, and preservation of the operation result.

No dependency bodies, external type definitions, caller contracts, or runtime observations were supplied. No runtime tests have been performed. The acceptance examples are specifications conditional on their stated assumptions, not claims of executed verification.
