<!-- Trusted provenance: F-e3b3a13bf9d9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e3b3a13bf9d9

## Purpose

F-e3b3a13bf9d9 retrieves a four-byte value through an indexed, two-stage memory lookup.

The supplied index selects a location in a table using a stride of 396 bytes. A four-byte value read from that location supplies the address basis for a second read, at a displacement of 3,200 bytes. The second four-byte value is returned without an explicit conversion, validation step, or intervening operation.

The established behavior is an indirect read, not a checked table-access service. The visible implementation does not establish the table’s length, the meaning of an entry, the type of the returned value, or a failure-reporting protocol.

## Interface

### Input

The function accepts one signed 16-bit integer index.

Its representable range is −32,768 through 32,767. Negative values participate in address calculation as negative indices; the visible implementation neither rejects them nor converts them into nonnegative indices.

The contract begins with the value received as this parameter. How a caller converts a wider integer into this parameter, including any truncation or out-of-range conversion, is outside the supplied evidence.

The function also depends on externally established memory state:

- A table-base value available through a global reference.
- Readable memory at the index-selected location.
- A usable address value stored at that location.
- Four readable bytes at the final address derived from that value.

There is no explicit length, capacity, status, or ownership argument.

### Output

The result is a four-byte value.

The visible return type does not establish whether these bits represent an integer, floating-point value, pointer, flags, or another payload. No signedness should be assigned to the result solely from this implementation.

On normal completion, the result is the four-byte value obtained by the final read. There is no separate success indicator.

### Widths and layout

The supplied representation establishes the following access widths and displacements:

| Property | Contract |
|---|---|
| Index | Signed 16-bit value |
| Table stride | 396 bytes |
| Selected table access | Four bytes at the beginning of the selected stride |
| Intermediate value | Four-byte integer used in subsequent address formation |
| Final displacement | 3,200 bytes |
| Final access | Four bytes |
| Return value | Four bytes |

The stride does not prove that each table element is a complete 396-byte record. Only the four-byte field selected at that spacing is accessed here.

The integer-to-address operations are consistent with a 32-bit address model. The supplied fragment does not independently specify the complete ABI, byte order, alignment requirements, or memory-protection rules.

## Behavior

### Address selection

Let \(B\) denote the table-base value obtained through the global reference, and let \(i\) denote the signed input index.

The selected table location is:

\[
S = B + 396i
\]

The displacement is measured in bytes. Consequently:

- Index zero selects the location at \(B\).
- Index one selects the location 396 bytes after \(B\).
- Index minus one selects the location 396 bytes before \(B\).

The function reads a four-byte value from \(S\). Let \(P\) denote the value obtained by that read, interpreted as the integer/address basis used by the implementation.

The final read location is:

\[
T = P + 3200
\]

The function reads four bytes from \(T\) and returns that value.

These equations describe the intended address relationships when the relevant arithmetic and address conversions are valid. They do not prescribe behavior for arithmetic overflow or invalid addresses.

### Observable read order

The table-base value must be available before the selected table location can be determined. The selected four-byte read must then supply the value used to determine the final read location.

The two explicit dereferences therefore have this dependency order:

1. Read the four-byte address-bearing value from the selected table location.
2. Read the four-byte result from the derived final location.

The supplied fragment does not resolve whether obtaining the table-base value itself requires a separate runtime memory load or whether that value is represented through some other established mechanism.

There is no visible call, write, check, or recovery operation between the two dereferences.

This is a semantic ordering of dependent accesses. The evidence does not establish machine-level transaction counts, speculative execution behavior, atomicity, or memory-barrier semantics.

### Index arithmetic

The index multiplication has a bounded mathematical range:

\[
-12{,}976{,}128 \leq 396i \leq 12{,}975{,}732
\]

That range fits within a signed 32-bit integer. Thus, under ordinary signed 32-bit promotion of the 16-bit input, multiplication by the stride does not itself overflow.

This does not establish that adding the displacement to the table base is valid. It also does not establish that adding 3,200 to the intermediate value is valid. Those operations depend on the actual values and the applicable arithmetic and address model.

No masking, clamping, modulo operation, bounds check, or special handling of either endpoint is visible.

### Preconditions for a normal return

A normal return requires all necessary address formation and memory reads to be valid in the execution environment. In particular:

- The table-base value must be usable with the supplied index.
- The selected four-byte region must be readable.
- The value read there must support formation of a readable final location at a displacement of 3,200 bytes.
- The final four-byte region must be readable.
- Any platform-specific alignment or address-conversion requirements must be satisfied.

These are preconditions, not checks performed by the function.

Neither a null table base nor a zero intermediate value receives special treatment. A zero intermediate value leads toward a final location corresponding to address 3,200; it does not explicitly cause a zero return or an early exit.

### Failure behavior

The fragment contains no explicit error branch, exception handler, fallback value, retry, or diagnostic.

If an address is invalid or an access violates platform requirements, the function provides no visible recovery path. The resulting fault, termination, exception, or other behavior is environment-dependent and documented_only here.

## Side effects

No memory writes are visible. The function does not explicitly update the table, the selected entry, the final value, or another global.

There are no visible allocations, deallocations, reference-count operations, resource acquisitions, or resource releases. No ownership transfer is established. Returning the four-byte payload does not, by itself, imply ownership of anything those bits might identify.

The memory reads are the relevant visible accesses. If either location belongs to memory-mapped hardware or another read-sensitive region, reading it could have external effects; the fragment does not identify either region as ordinary memory or device memory.

No synchronization is visible. In particular, the function does not establish a consistent snapshot across the table entry and the final read. If external activity changes relevant memory, the implementation supplies no visible mechanism to prevent or reconcile that change.

These observations do not establish a general purity guarantee: memory-access faults, concurrent mutation, and read-sensitive mappings remain outside the fragment’s guarantees.

## Dependencies

No other function is called in the supplied body. There are therefore no callee dependencies to identify with opaque function IDs.

The function’s dependencies are data and execution-environment dependencies:

- An externally supplied table-base value.
- The four-byte address-bearing field selected by the index and stride.
- The four-byte payload at the derived final location.
- The platform’s integer, pointer, alignment, and memory-access rules.

The supplied mapping identifies the analyzed function itself; it does not establish an additional dependency.

## Acceptance examples

These examples are conditional behavioral criteria, not reports of executed tests. Each normal-return example assumes valid arithmetic, valid address conversion, and readable memory.

### Zero index

If the index is zero, the selected four-byte read is at the table base. If that read supplies \(P\), and the four bytes at \(P + 3200\) represent payload \(V\), the function returns \(V\).

The table-entry value \(P\) is not itself returned.

### Positive index

If the index is two, the selected table location is 792 bytes after the table base. The function uses the four-byte value at that location to derive the final read address.

It does not visibly inspect the intervening table entries or scan for a matching record.

### Negative index

If the index is minus one, the selected table location is 396 bytes before the table base.

If the required memory and address operations are valid, the same two-stage lookup completes normally. Rejecting this input solely because it is negative would add behavior not present in the supplied implementation.

### Payload independence

If a valid final read yields four zero bytes, those bytes are returned. If it yields another four-byte pattern, that pattern is returned.

The implementation contains no visible payload-dependent branch or sentinel interpretation.

### Invalid selected location

If the selected table location is unreadable, a normal return is not guaranteed. A contract requiring an explicit error value, a retry, or a fallback table entry would not be supported by the evidence.

### Shared destination

If two selected table fields contain the same intermediate value, and the final memory remains unchanged, both lookups return the same final four-byte payload. The function visibly applies no index-dependent transformation after reading the selected field.

## Uncertainties

The following matters remain documented_only:

- The semantic meanings of the table, index, intermediate value, and returned payload.
- The number of valid table entries and whether negative indices are intentionally supported.
- The initialization, storage, and lifetime of the table-base value.
- Whether the intermediate value is formally a pointer, an integer encoding an address, or another address representation.
- The complete layout surrounding either accessed field.
- The target byte order, calling convention, and exact address-conversion rules.
- Overflow behavior in base addition or final-address formation.
- Alignment constraints and unaligned-access behavior.
- Whether the reads are atomic, externally mutable, volatile, or device-sensitive.
- The environment’s response to invalid memory access.

An array-like table and a fixed field within a pointed-to object are plausible inferred intent. Neither interpretation is required to describe the established lookup behavior.

## Evidence

The supplied static body supports the following conclusions:

- One signed 16-bit parameter is scaled by a 396-byte stride.
- A global-derived base participates in selecting a four-byte read.
- That read supplies the value used to form a second address with a 3,200-byte displacement.
- The second four-byte read supplies the returned value.
- No explicit branches, calls, writes, validation, or recovery operations appear.

The displacement range above is derived mathematically from the signed 16-bit input range and the supplied stride.

No runtime tests have been performed. The acceptance examples state conditional requirements; they are not execution evidence.
