<!-- Trusted provenance: F-fcf5dd427ad6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fcf5dd427ad6

## Purpose

F-fcf5dd427ad6 conditionally produces two 32-bit output values and one byte-sized output. Its decision depends on equality with a captured global value, a nonzero word read from an input object, and a nonzero result from F-90f09657a7b6.

On the accepted path, the function derives the byte output from F-cb3a1a094c4f, writes the constant 16 to one word output, invokes F-e19a821f5d85 with local scratch storage, and writes a product to the other word output. It then returns 1. On the rejected path, it returns 0 after its initial output-clearing writes.

These statements describe the recovered static behavior. The business meaning of the global value, object field, byte values, constant 16, and product is not established. No runtime tests have been performed.

## Interface

The recovered interface has seven arguments and a 32-bit return value. Widths below describe the apparent 32-bit target interface, not a portable language-level ABI specification.

| Position | Contract role | Width and interpretation |
|---|---|---|
| 1 | Object address | A 32-bit integer used as an address base. The function accesses a 32-bit word at byte offset 4. |
| 2 | First forwarded value | An opaque 32-bit value passed to F-90f09657a7b6 and F-cb3a1a094c4f. |
| 3 | Comparison value | A 32-bit integer compared for equality with a global value captured at entry. |
| 4 | Second forwarded value | An opaque 32-bit value passed only to F-90f09657a7b6. |
| 5 | Constant-output address | Address of storage receiving a 32-bit zero initially and, on the accepted path, a 32-bit value of 16. |
| 6 | Product-output address | Address of storage receiving a 32-bit zero initially and, on the accepted path, a product computed from local scratch values. |
| 7 | Byte-output address | Address of one byte written only on the accepted path by this function. |

The return value is a 32-bit value:

- **0:** The comparison, nonzero-field check, or F-90f09657a7b6 result rejected the operation.
- **1:** Execution reached the end of the accepted path.

The latter is conditional on all required accesses, calls, and arithmetic completing normally. There is no separate reported error value for an invalid address, dependency failure outside its ordinary return convention, or invalid scratch data.

### Storage requirements

The fifth and sixth arguments must support 32-bit writes immediately upon entry, regardless of whether the comparison will succeed.

The object must support the required 32-bit reads at byte offset 4 whenever execution reaches them. A mismatching comparison value prevents those reads by this function; it does not prevent the initial output writes.

The seventh argument must support a one-byte write if the accepted path is reached. This function does not dereference it on a normally rejected path.

No non-aliasing requirement is explicitly enforced. Overlapping arguments can change the behavior because the writes and repeated reads occur in a particular order. Examples below assume disjoint storage unless stated otherwise.

## Behavior

### Entry snapshot and output initialization

The function first captures one global 32-bit value for use during this invocation. That captured value is used both for the equality decision and, if the operation is accepted, as an argument to F-e19a821f5d85.

It then performs these explicit writes in order:

1. A 32-bit zero to the product-output address.
2. A 32-bit zero to the constant-output address.

The global capture precedes both writes. If either output address aliases the global storage, these writes do not retroactively change the captured value.

The byte output is not initialized here.

### Acceptance decision

The decision is short-circuited:

- If the comparison value does not equal the captured global value, the function rejects the operation without reading the object field or calling any dependency.
- If the comparison matches, the function reads the 32-bit word at object byte offset 4.
- If that word is zero, it rejects the operation without calling a dependency.
- If that word is nonzero, the function obtains the word at the same offset again and supplies it to F-90f09657a7b6, followed by the first and second forwarded values.
- A zero result from F-90f09657a7b6 rejects the operation. Any nonzero result selects the accepted path.

The recovered expression contains separate field accesses for the nonzero check and the first dependency’s argument. It does not establish that these accesses are volatile or that they necessarily produce separate physical memory transactions in the original executable. It does establish no explicit saved field value spanning all subsequent uses.

On rejection, the function returns 0 without further explicit output writes.

The rejected path’s final output contents are guaranteed to be the initial zeros only when no intervening dependency or aliasing effect changes them. In particular, rejection through a zero result from F-90f09657a7b6 follows an external call whose side effects are unknown.

### Accepted path: byte and constant outputs

After a nonzero result from F-90f09657a7b6, the function reads the object word at byte offset 4 again. It passes that current word and the first forwarded value to F-cb3a1a094c4f.

This reread matters: F-90f09657a7b6 may have changed the object or other accessible state. The contract does not require the field value supplied to the second dependency to equal the value supplied to the first.

The return from F-cb3a1a094c4f controls only the byte value selected by this function:

- A zero result selects byte value 0.
- Any nonzero result selects byte value 4.

The function writes the selected byte to the byte-output address. It then writes the 32-bit constant 16 to the constant-output address.

A zero result from F-cb3a1a094c4f does **not** turn acceptance into rejection. It changes the byte output from 4 to 0 while the accepted path continues.

### Accepted path: scratch-dependent product

The function next calls F-e19a821f5d85 with:

- The address of a local scratch region represented as two 32-bit integer elements.
- The global value captured at entry.

A separate local 32-bit integer scalar also participates in the subsequent computation. The visible function body does not initialize either the scratch elements or that scalar before this call.

After the call, the function multiplies the first scratch element by the separate scalar and writes the result to the product-output address. The second element of the two-element scratch region is not explicitly read by this function.

There is an important unresolved storage contract here: the visible call passes a pointer to the two-element region, but does not explicitly pass the address of the separate scalar. The supplied material does not independently establish a valid layout or write mechanism through which F-e19a821f5d85 initializes that scalar. Accordingly, the product is specified only conditionally on the dependency and actual storage layout supplying defined operand values.

The recovered operands are signed 32-bit integers. The source establishes a multiplication and a 32-bit destination, but does not establish portable overflow behavior. Do not assume wrapping arithmetic, saturation, or overflow reporting. Product examples below use representable results.

After the product write, the function returns 1. There is no visible validation of scratch values or dependency result at this stage.

## Side effects

The explicit externally directed write order is:

| Order | Write |
|---|---|
| 1 | Zero to the 32-bit product output. |
| 2 | Zero to the 32-bit constant output. |
| 3, accepted path only | Either 0 or 4 to the byte output. |
| 4, accepted path only | 16 to the 32-bit constant output. |
| 5, accepted path only | The scratch-dependent product to the 32-bit product output. |

F-90f09657a7b6 and F-cb3a1a094c4f run before the accepted-path byte write. F-e19a821f5d85 runs after the byte and constant writes but before the product write. Dependency side effects may therefore interleave with, or affect the final visibility of, these writes.

Aliasing consequences include:

- If an initialized word output overlaps the object field, the field check observes storage after the initialization writes.
- If the two word outputs identify the same storage, the final product write follows and supersedes the accepted-path constant write.
- If the byte output overlaps a word output, later word writes may overwrite the byte.
- If a dependency can access output storage indirectly, its effects must be considered at its actual position in the sequence.

No allocation, release, handle acquisition, or ownership transfer is explicitly visible in this function. That does not establish that its dependencies perform none.

The scratch address is passed to F-e19a821f5d85 for the duration of a call. The supplied evidence establishes neither permission to retain that address nor a dependency guarantee against retention. Caller-provided storage is not explicitly released by this function.

No explicit synchronization, rollback, or exception recovery is present.

## Dependencies

| Opaque function ID | Invocation and consumed result | Unknown contract elements |
|---|---|---|
| F-90f09657a7b6 | Receives the current object field, first forwarded value, and second forwarded value. Zero rejects; nonzero accepts. | Argument meanings, side effects, address interpretation, ownership, exceptional behavior, and state preservation. |
| F-cb3a1a094c4f | Receives a freshly read object field and the first forwarded value. Zero selects byte 0; nonzero selects byte 4. | Meaning of the predicate, side effects, ownership, exceptional behavior, and state preservation. |
| F-e19a821f5d85 | Receives local scratch storage and the entry-captured global value. No returned value is consumed; scratch values are subsequently used. | Scratch footprint, initialization guarantees, relationship to the separate scalar, retention of the pointer, and other side effects. |

The dependency call order is fixed on the accepted path: F-90f09657a7b6, then F-cb3a1a094c4f, then F-e19a821f5d85. A rejected path invokes either no dependency or only F-90f09657a7b6.

No dependency is assumed to preserve globals, object fields, output storage, or memory ownership.

## Acceptance examples

These are conditional contract examples, not observed test results.

### Comparison mismatch

Given valid, disjoint word-output addresses and a comparison value different from the entry-captured global value:

- Both word outputs receive zero.
- The object field is not accessed by this function.
- No dependency is called.
- The byte output is not explicitly written.
- The return value is 0.

### Matching comparison with a zero field

Given matching comparison values and an object field that is zero when checked:

- The initial word-output clears occur before the field read.
- No dependency is called.
- The byte output is not explicitly written.
- The return value is 0.

### First dependency rejects

Given a matching comparison, a nonzero checked field, and a zero return from F-90f09657a7b6:

- F-cb3a1a094c4f and F-e19a821f5d85 are not called.
- There are no accepted-path output writes.
- The return value is 0.
- Final word-output zeros are conditional on F-90f09657a7b6 not modifying that storage.

### Accepted operation with zero byte selection

Given acceptance by F-90f09657a7b6, a zero return from F-cb3a1a094c4f, and defined product operands 6 and 7:

- The function explicitly writes byte 0, then word 16.
- After F-e19a821f5d85, it writes product 42.
- It returns 1.

This example additionally requires disjoint outputs and dependency behavior that does not invalidate the stated final contents.

### Accepted operation with nonzero byte selection

Under the same conditions, except that F-cb3a1a094c4f returns any nonzero value:

- The selected byte is exactly 4, not the dependency’s return value truncated to a byte.
- The constant output is still 16.
- The product calculation and return value are unchanged.

### Object mutation between dependencies

If F-90f09657a7b6 changes the object field and returns nonzero, F-cb3a1a094c4f receives the field value read after that call. A newly zero field is still passed to the second dependency; this function does not repeat the earlier nonzero gate.

### Aliased word outputs

If both word-output addresses designate the same valid 32-bit storage, the accepted path writes 16 there before ultimately writing the product there. The final explicit write is the product, not 16.

## Uncertainties

**Established static evidence:** entry capture of a global value; ordered initial clears; short-circuited decision; dependency ordering; repeated object-field access; byte values 0 and 4; constant 16; scratch-dependent multiplication; and returns 0 or 1.

**Inferred intent:** the return appears to indicate acceptance, and the byte appears to encode a two-way classification. Neither interpretation establishes application-level meaning.

**documented_only unknowns:**

- The global’s semantic role and permitted values.
- The object’s full layout, allocation extent, alignment, and field meaning.
- Whether the object field is an address, identifier, handle, or another integer.
- The precise physical memory-access behavior of the recovered field expressions.
- The scratch layout and initialization of the separate product operand.
- Overflow behavior for an out-of-range signed product.
- Dependency side effects, ownership rules, and exceptional behavior.
- Concurrency guarantees and synchronization requirements.
- Platform behavior for invalid, misaligned, or overlapping accesses.

These unknowns must not be replaced with assumptions that dependencies are pure, that all scratch operands are initialized, or that arithmetic wraps.

## Evidence

This contract is based solely on the supplied recovered function body and opaque dependency mapping. It distinguishes explicit operations from unresolved dependency and storage contracts.

No dependency implementations, independent ABI documentation, runtime traces, or execution results were supplied. No runtime tests have been performed.
