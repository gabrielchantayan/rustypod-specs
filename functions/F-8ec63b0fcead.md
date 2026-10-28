<!-- Trusted provenance: F-8ec63b0fcead; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8ec63b0fcead

## Purpose

F-8ec63b0fcead selects a 32-bit return value through two ordered dependency-mediated decisions and, if neither decision selects an early return, a field-based fallback.

The established behavior is a priority chain:

1. A zero result from the first comparison selects one global return value.
2. Otherwise, a zero result from the second comparison selects a different global return value.
3. Otherwise, a field associated with the input selects between a previously read field value and a third global return value.

The inferred intent is selection among context-dependent values with a default. The meanings of the contexts, selectors, field, and returned values are not established. In particular, the evidence does not establish that the result is a Boolean, status code, string pointer, object pointer, or owned resource.

## Interface

### Input

The function accepts one integer-valued argument, represented as a 32-bit `int` in the supplied material. This argument is used as an address base, not merely as a numeric option.

Two byte offsets are relevant:

| Offset from input base | Use |
|---|---|
| `0x28` — 40 bytes | The computed address is passed to F-44362605ab79. This function does not explicitly load the contents at that address before the call. |
| `0x2c` — 44 bytes | A 32-bit integer field is read on the final selection path. The supplied representation contains two separate reads of this field. |

No complete object layout is established. There is no evidence about fields preceding these offsets, the total object size, allocation method, or alignment guarantee.

The input is not checked for null, address-range validity, alignment, object lifetime, or arithmetic overflow.

### Output

The declared result is a 32-bit integer. On normal completion, it comes from exactly one of four sources:

- The first global return slot.
- The second global return slot.
- The first of two reads of the field at offset 44.
- The final global fallback slot.

No explicit normalization, masking, sign conversion, or Boolean conversion is shown for the selected return value. A negative or zero value from a selected source is not rejected.

A 32-bit intermediate value returned by each value-producing dependency is passed to the comparison dependency. The supplied material does not establish whether that intermediate value represents an address, handle, scalar, or another encoded quantity.

### Contract terminology

The following names identify roles, not domain meanings:

- **First return slot:** the global value selected by a zero result from the first comparison.
- **Second selector slot:** the global value supplied as the second argument of the second comparison.
- **Second return slot:** the global value selected by a zero result from the second comparison.
- **Final fallback slot:** the global value selected when the final field check observes zero.
- **Fixed selector:** the fixed-address argument supplied to the first comparison.

These roles must not be treated as immutable constants. The source shows accesses to global storage, and dependency effects on that storage are unknown.

## Behavior

### First decision

The function first calls F-3ad65987df9a with no arguments.

After that call returns, its 32-bit result becomes the first argument to the comparison dependency. The second argument is the fixed selector. The supplied representation presents this selector as a fixed address associated with named data; its contents and interpretation are not established.

The comparison dependency returns an integer. Only equality to zero matters:

- If the result is exactly zero, the function reads the first return slot and returns that value.
- If the result is nonzero, execution continues to the second decision.

A negative comparison result has the same branch effect as a positive nonzero result. The function does not test ordering or distinguish different nonzero values.

When the first comparison returns zero, this function does not call F-44362605ab79, does not perform the second comparison, and does not explicitly read the field at offset 44.

The first return slot is selected after the comparison returns. A value cached before the dependency calls would not preserve the shown access order if those calls can change that slot.

### Second decision

This decision is reached only when the first comparison returns nonzero.

The function computes the input base plus 40 bytes and passes that computed value to F-44362605ab79.

This is an address calculation followed by a call. It is not an explicit load of a 32-bit value from offset 40. Any access to the object or subobject at that address belongs to F-44362605ab79 and is not described by the available body.

After F-44362605ab79 returns, the function supplies:

- Its 32-bit result as the first comparison argument.
- The value read from the second selector slot as the second comparison argument.

The selector-slot read therefore occurs after F-44362605ab79 has completed. The evidence does not permit substituting an earlier value on the assumption that the dependency preserves globals.

The second comparison again distinguishes only zero from nonzero:

- If the result is exactly zero, the function reads the second return slot and returns that value.
- If the result is nonzero, execution continues to the field-based selection.

When this comparison returns zero, the final field reads and final fallback-slot read are not performed by this function.

### Field-based selection

This path is reached only when both comparisons return nonzero.

The supplied representation shows two reads of the same 32-bit field at input base plus 44 bytes, with distinct roles:

1. The first read supplies the candidate return value.
2. The second read determines whether the global fallback is selected.

If the second read observes zero, the function reads and returns the final fallback slot.

If the second read observes a nonzero value, the function returns the value obtained by the first read. It does not return the second read merely because that read controlled the branch.

For stable ordinary memory, this behaves like “return the field if nonzero; otherwise return the fallback.” That shorthand is not a complete description of the supplied representation when the two reads can differ.

In particular:

- A nonzero first read followed by a zero second read selects the fallback.
- A zero first read followed by a nonzero second read returns zero.
- Two different nonzero reads return the first value.

These distinctions describe the supplied static representation. Whether an executable performs two separately observable memory reads is documented_only: the material does not establish volatile qualification, synchronization, optimization, or machine-instruction behavior.

### Address arithmetic and validity

The offset calculations use the integer-valued input as an address base. The offsets themselves fit within 32 bits, but adding them to an arbitrary input may cross an address-space or signed-integer boundary.

The function contains no explicit handling for such a case. The evidence does not establish whether these additions are governed by source-language signed-overflow rules, machine-address wrapping, or another execution model. A portable contract must therefore require address calculations valid under the actual execution environment rather than assume wrapping.

The final path requires valid 32-bit reads at offset 44. Earlier paths may return without those reads, but this does not establish that an invalid input is safe: dependencies may access memory or inspect shared state in undocumented ways.

## Side effects

The function body shows no explicit writes to caller-provided memory or global storage. It does not explicitly allocate, release, retain, copy, or transfer ownership of any resource.

Its explicit effects consist of:

- Calling dependencies in the order described above.
- Passing dependency results and selector arguments to the comparison dependency.
- Reading selected global slots.
- Conditionally reading the 32-bit field at offset 44.
- Returning the selected 32-bit value.

The absence of explicit writes is not a purity guarantee. Any dependency may have effects not visible in this body, including changes to global slots or the input-associated memory.

The input-derived address is passed to F-44362605ab79 without an explicit ownership operation. This alone establishes neither borrowing nor ownership transfer. Likewise, a returned value must not be assumed to be an owned allocation merely because its bits could represent an address.

No cleanup is shown after dependency calls or before any return.

## Dependencies

| Opaque function ID | Established use | Unknown behavior |
|---|---|---|
| F-3ad65987df9a | Called first, with no arguments; its 32-bit result is passed to the first comparison. | Meaning of the result, memory effects, ownership behavior, failure behavior, and global-state preservation. |
| F-44362605ab79 | Called only after a nonzero first comparison; receives the input-derived address at offset 40; its 32-bit result is passed to the second comparison. | Required object layout, reads or writes through its argument, result meaning, ownership behavior, and failure behavior. |
| F-unmapped-01 | Local opaque identifier for the comparison dependency, whose published opaque ID was not supplied. Called with two arguments; an integer result of zero selects an early return. | Whether it compares strings, identities, values, or other representations; argument requirements; effects; error conventions; and ownership behavior. |

F-unmapped-01 denotes the same dependency at both call sites. The name is local to this contract, not an asserted published identifier.

No evidence establishes that zero means “equal” or “success.” The supported statement is only that zero takes the corresponding early-return branch.

If a dependency does not return normally, the subsequent behavior described here is not reached. No local recovery mechanism is shown.

## Acceptance examples

These examples are conditional acceptance criteria derived from static evidence, not executed tests.

### First decision wins

Given that F-3ad65987df9a returns normally and the first F-unmapped-01 call returns zero:

- The result must be the first return slot’s value as read after that comparison.
- F-44362605ab79 must not be called by this function.
- The second comparison and final field reads must not occur.

This remains true if the selected global value is itself zero or negative.

### Second decision wins

Given a nonzero first comparison and a zero second comparison:

- F-44362605ab79 must receive the input base plus 40 bytes.
- The second selector-slot value must be obtained after that helper returns.
- The result must come from the second return slot after the second comparison.
- The final field reads must not occur.

### Stable nonzero field

Given two nonzero comparison results and a field that remains equal to `37` across both reads:

- The function must return `37`.
- It must not read the final fallback slot on this path.

The same selection rule applies to a stable negative field value.

### Stable zero field

Given two nonzero comparison results, two field reads of zero, and a final fallback-slot value of `91` at the time of its read:

- The function must return `91`.

The fallback may itself be zero; no further fallback is shown.

### Differing field observations

Subject to an execution model that preserves the two represented reads:

- First read `12`, second read zero: return the final fallback-slot value.
- First read zero, second read `12`: return zero.
- First read `12`, second read `19`: return `12`.

These cases distinguish the candidate value from the branch-controlling observation. They do not assert that unsynchronized concurrent mutation is valid in the original execution environment.

### Dependency mutation of a selected global

If the first comparison changes the first return slot and then returns zero, the shown order selects the value read after the call, not a value captured before it.

Likewise, if F-44362605ab79 changes the second selector slot, the second comparison receives the subsequently read selector value.

## Uncertainties

The following are documented_only:

- The semantic type and lifetime of the input-associated object.
- The content, encoding, and extent of either selector.
- The widths of pointer-like dependency arguments in the actual ABI beyond the supplied 32-bit representations.
- The declarations and synchronization properties of global storage.
- Whether the two field reads remain distinct in executable behavior.
- The allowed address-arithmetic range and overflow semantics.
- Whether dependencies mutate globals, access the input object, allocate memory, release resources, or transfer ownership.
- Any dependency-specific error representation.
- Reentrancy, thread safety, and signal safety.
- Whether the selected return values encode scalars, addresses, handles, or statuses.

No explicit validation, error translation, retry, or exception recovery is present. That does not establish the absence of failures inside dependencies or during invalid memory access.

## Evidence

Established static evidence supports the ordered call structure, zero-versus-nonzero branch conditions, offsets of 40 and 44 bytes, conditional global reads, two represented field reads, and selected return sources.

The supplied declarations support treating the explicit integer input, field, comparison results, and return as 32-bit quantities. They do not establish a complete external ABI or object schema.

The interpretation as context-sensitive selection with fallback is inferred intent only. Domain meanings, comparison semantics, ownership rules, and executable memory-observation guarantees remain unestablished.

No runtime tests have been performed.
