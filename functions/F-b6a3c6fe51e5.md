<!-- Trusted provenance: F-b6a3c6fe51e5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b6a3c6fe51e5

## Purpose

F-b6a3c6fe51e5 selects one of three outcomes from an integer selector. Two selectors cause a single-byte write at displacement 40 bytes from a caller-supplied base address. All other selectors return a fixed result without accessing memory through that base.

The established behavior is:

| Selector | Memory effect | Return value |
|---|---|---|
| 0 | Store byte value 5 at base + 40 | 0 |
| 15 | Store byte value 7 at base + 40 | The original base argument |
| Any other value | No store | 14 |

Calling the affected byte a status or mode field would be an interpretation, not an established fact. Neither its meaning nor the meaning of the return values is available.

## Interface

The function accepts two integer arguments and returns an integer:

- **Base argument:** An integer used to form a memory address on the two writing paths. It is also returned unchanged when the selector is 15.
- **Selector argument:** An integer compared for exact equality with 0 and 15.
- **Result:** Either 0, 14, or the original base argument, according to the table above.

The supplied representation declares both arguments and the result as `int`. This contract describes the apparent 32-bit interface. The declaration alone does not independently establish the target ABI; confirmation of that width is a documented uncertainty.

Under that 32-bit interpretation:

- Both input arguments and the result occupy 32 bits.
- Selector comparisons use the complete integer value, not merely its lowest byte.
- The memory destination is calculated using displacement 0x28, equal to decimal 40.
- Each store writes exactly 8 bits.
- Returning the base means returning its original 32-bit value, not the destination address and not the byte stored.

For either writing selector, a normal successful invocation requires the destination address to be writable. The supplied behavior performs no null check, bounds check, alignment check, object-type check, or allocation-size check.

For ordinary non-wrapping addresses, storage containing at least 41 bytes beginning at the base is sufficient to include the addressed byte. The function itself does not establish that such an object exists.

For other selectors, no readable or writable storage at the base is needed by the shown function body. This is a statement about its direct memory accesses, not a guarantee about unrelated calling-environment requirements.

## Behavior

### Selector 0

The function selects the zero case first. It stores the byte value 5 at the address 40 bytes beyond the supplied base, then returns integer 0.

The prior value of the destination byte is not consulted. This is an overwrite, not a bitwise update, increment, or conditional change based on existing memory.

The store precedes the normal return. If the store cannot complete, the supplied material establishes no normal result for that execution.

### Selector 15

If the selector is not zero and equals decimal 15, the function stores the byte value 7 at the same displacement and then returns the original base argument.

The supplied condition also excludes selector 1 before recognizing selector 15. That exclusion does not change which integer values take this path: a value equal to 15 is necessarily unequal to 1. In particular, selector 1 does not receive its own write or distinct result.

The return does not identify the address of the written byte. For example, a base of 4096 produces a destination of 4136 but a return value of 4096.

### Every other selector

Every selector other than 0 and 15 returns decimal 14. There is no direct memory load or store through the base on this path.

This includes selector 1, negative selectors under a signed interpretation, and values that merely have the same low byte as a recognized selector. For example, decimal 256 is not selector 0, and decimal 271 is not selector 15.

No validation precedes this result. A base that would be unusable on a writing path does not itself cause a memory access on this path.

### Observable ordering and access scope

There is at most one explicit data-memory access in an invocation: the selected single-byte store. No destination read precedes it. No source-buffer read, loop, retry, allocation, cleanup operation, or dependency call appears in the supplied body.

For the two writing paths, the observable functional order is destination store followed by normal return. For the remaining path, the only specified functional outcome is the return value 14.

This ordering does not establish thread synchronization, a memory barrier, or inter-thread visibility guarantees.

## Side effects

The only established direct mutation is an 8-bit overwrite at base + 40 for selectors 0 and 15.

Under ordinary writable-memory semantics, neighboring bytes are not explicitly modified. No wider store or read-modify-write operation is specified. The effect of using memory-mapped hardware, protected memory, or another unusual mapping is outside the available evidence.

The function does not allocate or release memory. It does not establish ownership of the base address, transfer ownership to another component, or change the lifetime of the addressed storage. The caller must arrange whatever storage validity is required by the selected path.

There is no explicit global-state access, logging, callback, or error-state update. This does not imply that the destination cannot be global storage: the caller may supply a base whose displaced address refers to such storage.

Invalid destinations have no defined recovery behavior in the supplied material. In particular, neither return value 0 nor return value 14 is an established report of a failed store.

## Dependencies

No function dependency is present in the supplied body. There are therefore no additional opaque function IDs to enumerate and no callee behavior needed to explain the three outcomes.

Execution still depends on the surrounding environment for:

- Integer and address widths.
- Conversion of the computed integer address into a writable memory location.
- The result of an invalid or inaccessible destination.
- Any concurrency or special-memory semantics.

These environmental requirements do not supply additional validation or error handling absent from the function.

## Acceptance examples

The following are conditional acceptance examples derived from static inspection. They have not been executed.

1. **Zero selector overwrites the designated byte.**  
   Given an ordinary writable destination at base + 40, with that byte initially 255, selector 0 must leave the byte equal to 5 and return 0. The prior byte value must not affect the result.

2. **Selector 15 returns the base, not the destination.**  
   Given base 4096 and a writable byte at address 4136, selector 15 must write 7 at address 4136 and return 4096.

3. **Selector 1 uses the fallback outcome.**  
   Given selector 1, the result must be 14, with no explicit access through the base. It must not write either 5 or 7.

4. **Selector matching is not byte truncation.**  
   Under the 32-bit interface, selectors 256 and 271 must each return 14 without a destination store. Their low-byte resemblance to 0 and 15 does not select a writing path.

5. **An unused base need not designate writable storage.**  
   Given selector 2, even a base whose displaced address would be inaccessible must not be dereferenced by this function. The specified normal result is 14.

6. **Repeated calls overwrite rather than accumulate.**  
   Given the same valid ordinary-memory base, a successful call with selector 0 followed by a successful call with selector 15 leaves the designated byte equal to 7. The results are respectively 0 and the base.

7. **An invalid writing destination has no specified normal outcome.**  
   Given selector 0 or 15 and an unwritable destination, acceptance must not require a particular return value, exception, or recovery action. Those outcomes are undocumented.

## Uncertainties

- **documented_only — ABI:** The apparent 32-bit integer interface is not independently confirmed by target metadata.
- **documented_only — Address arithmetic boundaries:** The supplied material does not establish whether base-plus-40 overflow wraps, traps, or lies outside valid execution. No acceptance example relies on overflow.
- **documented_only — Object layout:** Only the byte displacement and store width are established. No enclosing structure size, field name, or object type is known.
- **documented_only — Semantic meaning:** Values 5, 7, and 14 have no established symbolic meaning. Describing 14 as an error code would be inferred intent.
- **documented_only — Failure behavior:** Invalid-memory handling and its externally visible consequences are unspecified.
- **documented_only — Concurrency:** Atomicity, synchronization, and behavior under concurrent access are not established.

## Evidence

The supplied static body establishes two exact-selector writing paths, one fallback return, byte-wide destinations at displacement 0x28, and no explicit calls.

The three-outcome table follows from those conditions. The selector-1 exclusion does not create an additional behavioral case.

All behavior described here is based on static evidence. No runtime tests, memory-access observations, or ABI checks have been performed.
