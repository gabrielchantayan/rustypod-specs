<!-- Trusted provenance: F-7c170ecfc86e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7c170ecfc86e

## Purpose

F-7c170ecfc86e conditionally stores the byte value **1** at an address derived from its two integer inputs.

The condition is an **upper-bound-only signed comparison**: the store occurs whenever the second input is less than 3. Negative values therefore select the store path as well. Values of 3 or greater cause the function to return without an explicit memory access.

A plausible intent is to mark one of three adjacent byte-sized fields relative to a base address. That interpretation is not the contract: the supplied source does not establish an object layout, a three-element allocation, or a requirement that the second input be nonnegative.

## Interface

The function accepts two integer inputs and produces no return value.

| Input | Contract role | Established properties |
|---|---|---|
| First input | Base-address integer | Used in address arithmetic only when the store condition is satisfied. It is not checked for validity. |
| Second input | Signed selector and address displacement | Compared against 3. If less than 3, its full signed value participates in the destination-address calculation. |

The inputs are represented as signed integers in the supplied source. Their exact width, the address width, and the calling convention are not established by the supplied material. A 32-bit representation may be plausible for the originating environment, but it is **documented_only**, not a verified interface requirement.

The stored datum is exactly one byte with numeric value 1. The destination is accessed as byte-sized storage, not as a multi-byte integer field. Endianness does not affect the value of this individual byte.

For ordinary arithmetic that is representable in the execution environment, the destination is:

**base-address integer + signed selector + 228**

The constant 228 is an address displacement. The selector is added directly; it is not multiplied by an element size, masked, clamped, or reduced modulo three.

### Preconditions

For a normal completion on the store path:

- The integer address calculations must have valid semantics in the originating execution environment.
- Conversion of the calculated integer to a destination address must be valid there.
- The resulting address must identify writable storage for one byte.

The function does not establish these conditions itself. In particular, it does not check allocation boundaries, object lifetime, address accessibility, or whether the selector is nonnegative.

When the selector is 3 or greater, the supplied source does not evaluate the destination-address expression. A writable destination is therefore not required by this function’s explicit operations on that path.

## Behavior

### Conditional access

The function first determines whether the signed selector is less than 3.

If the selector is 3 or greater:

- No destination address is calculated by the conditional body.
- No explicit memory read or write occurs.
- The function returns without a result.

If the selector is less than 3:

- The base-address integer and the signed selector are added.
- The fixed displacement 228 is added to that sum.
- The resulting integer is used as the address of a byte-sized destination.
- The function stores the byte value 1 at that destination.
- The function returns without a result if the access completes normally.

This is a conditional assignment, not a conditional bit operation. The destination’s previous contents are not read by the supplied function. A previous value of 0, 1, or any other byte value does not alter the value written.

### Address layout and negative selectors

For the nonnegative selectors admitted by the comparison, the destinations are adjacent:

| Selector | Destination relative to the base-address integer |
|---:|---:|
| 0 | +228 |
| 1 | +229 |
| 2 | +230 |

Those offsets do not establish that three bytes are allocated or that they constitute an array.

Negative selectors are also admitted:

| Selector | Destination relative to the base-address integer |
|---:|---:|
| −1 | +227 |
| −228 | +0 |
| −229 | −1 |

These relationships apply when the intermediate arithmetic and address conversion are valid. They do not authorize accesses outside any actual allocation.

There is no minimum accepted selector in the supplied comparison. Any representable negative selector takes the store path, although arithmetic validity and destination accessibility may prevent normal completion.

### Arithmetic boundaries

The supplied expression groups the address calculation as the sum of the two inputs, followed by addition of 228. The material does not establish a portable wraparound rule for either addition.

Consequently, the contract does not replace this calculation with unsigned modular arithmetic. It also does not guarantee that a mathematically valid final sum makes an overflowing intermediate sum acceptable. Behavior at signed-overflow boundaries, and behavior when an integer cannot validly designate an address, remain environment-dependent or otherwise unspecified by the supplied evidence.

### Completion and errors

There is no explicit error return, exception handler, retry, fallback, or failure notification.

For a valid destination and valid arithmetic, completion consists of the single-byte store followed by return. For an invalid destination or invalid arithmetic, the supplied source does not establish a particular observable outcome. A memory fault, process termination, or other platform-specific outcome must not be presented as a guaranteed error protocol.

## Side effects

The only explicit memory side effect is one byte-sized store on the selector-less-than-3 path.

No explicit operation:

- Reads the destination’s old value.
- Reads neighboring bytes.
- Writes neighboring bytes.
- Allocates or releases memory.
- Calls another function.
- Updates a separate status field.
- Emits output or records an error.

The function neither acquires ownership of the destination storage nor visibly transfers ownership. The caller or surrounding environment must arrange the storage’s lifetime and accessibility.

The function is not necessarily limited to a private object. Its calculated destination could refer to shared storage, global storage, or memory-mapped storage if the environment permits that address interpretation. The supplied material does not identify the destination’s ownership or storage class.

No synchronization, atomicity guarantee, or memory-ordering protocol is established. A byte-sized assignment alone is insufficient evidence for any particular concurrent-access guarantee.

The store remains part of the specified behavior even if the destination already contains 1. In ordinary memory, the resulting byte would be unchanged; for an externally observable destination, the access itself may matter. Hardware-level transaction details are not established.

## Dependencies

No function dependencies are visible in the supplied body. There are therefore no dependency function IDs to enumerate.

Execution depends on environmental facilities rather than visible callees:

- Signed integer comparison and addition.
- Interpretation of an integer as a byte-storage address.
- Permission and ability to write that address.
- The execution environment’s treatment of invalid arithmetic and invalid memory accesses.

No assumptions about an unknown callee preserving globals or acquiring, retaining, or releasing memory are needed, because no call is present.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence. They are not reports of executed tests.

Unless stated otherwise, assume representable intermediate arithmetic, valid address conversion, writable ordinary memory at the selected destination, and no concurrent interference.

| Scenario | Required behavior |
|---|---|
| Selector is 0; the byte at base +228 initially contains 0 | Store 1 at base +228 and return without a result. |
| Selector is 1; the byte at base +229 initially contains 255 | Replace that byte with 1. This is not a bitwise OR that would preserve 255. |
| Selector is 2 | Store 1 at base +230. |
| Selector is −1 | Store 1 at base +227. The negative selector is not rejected. |
| Selector is −228 | Store 1 at the base address itself, provided the grouped arithmetic is valid. |
| Selector is −229 | Store 1 one byte before the base address, provided that destination and the arithmetic are valid. |
| Selector is 3 | Return without an explicit memory access. |
| Selector is a representable value greater than 3 | Return without an explicit memory access. |
| Selector is 2; the destination already contains 1 | Perform the byte assignment of 1; do not infer a skip based on the existing value. |
| Selector is 3; the first input would not designate usable storage | Do not dereference it through this function’s conditional body. |

For the store examples, surrounding ordinary-memory bytes are not written by this function. That statement concerns its explicit operations, not side effects that an unusual mapped device might associate with a byte access.

Two boundaries intentionally lack a required successful outcome:

- An accepted selector produces an inaccessible destination.
- Either grouped address addition exceeds the supported signed range.

Such cases cannot be assigned a portable fault type, wraparound destination, or successful return from the supplied evidence.

## Uncertainties

The following are **documented_only**:

- Exact integer widths and ranges.
- Address width and integer-to-address conversion rules.
- Calling convention and binary-level parameter placement.
- Concrete allocation size, object layout, field names, and storage ownership.
- Whether callers restrict the selector to 0 through 2.
- Whether negative selectors are intentional or expose a missing lower-bound check.
- Overflow behavior in the originating executable.
- Memory-access fault behavior.
- Concurrency guarantees and hardware-level access details.

The interpretation “set a flag in a three-byte region” is inferred intent only. The established behavior is broader: all signed selectors below 3 select a byte store.

## Evidence

The supplied static body establishes a signed comparison against 3, an address calculation using both inputs and the displacement 228, one byte-sized assignment of 1 on the accepted branch, and a return without a value.

It contains no lower-bound check, destination-content read, visible function call, allocation operation, or explicit error handling.

**No runtime tests have been performed.** The acceptance examples are static consequences subject to their stated arithmetic and memory preconditions.
