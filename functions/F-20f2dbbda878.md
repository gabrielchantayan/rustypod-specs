<!-- Trusted provenance: F-20f2dbbda878; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-20f2dbbda878

## Purpose

F-20f2dbbda878 conditionally retrieves a four-byte value from an indexed table.

The input identifies an object containing a link at byte offset 80. If that link is zero when tested, the function returns a four-byte zero. Otherwise, it obtains an index from a linked object and returns the four-byte value located at byte offset 4 within the corresponding 12-byte table entry.

This is a lookup, not an allocation, update, or ownership-transfer operation. The static representation contains no explicit writes to the objects or table and no calls to other functions.

The meaning of the returned bits is unknown. They must not be described as a status, pointer, identifier, or Boolean without additional evidence.

## Interface

### Input

There is one argument: an integer-valued address identifying the primary object.

The supplied representation uses integer-to-address conversions for memory access. It models the relevant link and index as 32-bit integer quantities, and the result as a four-byte value. The actual calling convention, pointer width, alignment rules, and byte order are not independently established.

The minimum relevant layouts are:

| Region | Byte offset | Accessed quantity |
|---|---:|---|
| Primary object | 80 | 32-bit link used first for a zero test, then read again as an address |
| Linked object | 16 | 32-bit integer index |
| Selected table entry | 4 | Four-byte return value |
| Table | Every 12 bytes | Start of the next indexed entry |

These are access requirements, not complete structure definitions. No meaning is established for bytes outside those fields.

### Output

The function returns exactly four bytes:

- Four-byte zero when the initial link read is zero.
- Otherwise, the four bytes read from the selected table location.

There is no separate success indicator. A returned zero can mean either that the initial link was zero or that the selected table value itself was zero.

### Preconditions

For a normal return:

- The primary-object address must permit the four-byte read at offset 80.
- On the nonzero branch, the second link read must yield an address permitting the index read at offset 16.
- The table-base value, index arithmetic, and final effective address must permit the four-byte table read.

The function performs no explicit validation of these conditions. A null primary-object address is not treated as an absent object: the function attempts the offset-80 read before it can make its conditional decision.

## Behavior

### Zero-link case

The function first reads the 32-bit field at byte offset 80 from the primary-object address and compares that value with zero.

If the value is zero, the result is four-byte zero. This path does not evaluate the table lookup. It does not require a readable linked object or a valid table entry.

The zero test concerns the field inside the primary object, not the input argument itself.

### Nonzero-link case

If the first link read is nonzero, the supplied representation reads the same primary-object field again. The second value supplies the linked-object address used by the lookup.

From that address, the function reads the 32-bit index at byte offset 16. The selected return location is the table-base address plus twelve times that index, plus four bytes.

For example, index zero selects byte offset 4 from the table base, index one selects byte offset 16, and index two selects byte offset 28. The function reads four bytes from the selected location and returns those bits without an explicit conversion or interpretation.

The 12-byte stride does not mean the entire entry is read. Only the four-byte field at offset 4 is explicitly accessed.

### Read ordering and repeated access

The established source-level ordering is:

- The initial primary-object link read precedes the branch decision.
- The second link read occurs only on the nonzero branch.
- That second read supplies the address for the linked-object index read.
- The index and table-base value are both needed before the final table access.

The relative evaluation order of the global table-base value and the expression obtaining the index is not established. The contract therefore does not require the table-base value to be obtained before or after the linked-object reads.

The two occurrences of the primary-object link access are distinct in the supplied representation. They must not be silently interpreted as a single captured value when describing that representation. However, actual machine-level load count and behavior under concurrent mutation are not established.

In particular, the first nonzero test does not establish that the second link value remains nonzero or points to the same object. There is no second zero check.

### Integer arithmetic and bounds

The index is presented as a signed integer rather than an explicitly unsigned quantity. There is no explicit rejection of negative values.

For values whose arithmetic and addressing are valid, a negative index selects a location before the table base. For example, index −1 selects a four-byte value beginning eight bytes before that base: negative twelve bytes for the stride, followed by the positive four-byte field offset.

No entry count, lower bound, upper bound, or overflow check appears. The function does not clamp, normalize, or otherwise constrain the index.

The supplied representation does not establish what happens when multiplication by 12 or subsequent address arithmetic exceeds its valid range. Neither modulo wraparound nor a particular failure mode should be promised. The offset additions used to reach the primary and linked fields are subject to the same unresolved address-arithmetic constraints.

## Side effects

The explicit operations are memory reads and returning a four-byte result. There are no explicit writes to:

- The primary object.
- The linked object.
- The table.
- The table-base state.

There are no explicit allocation, deallocation, reference-count, locking, logging, or callback operations.

The function does not visibly acquire ownership of any region, release any region, or transfer ownership through an explicit mechanism. If the returned bits happen to encode an address, that coincidence does not establish an ownership contract.

Memory access may itself have externally visible consequences on a platform with memory-mapped devices or access faults. Such consequences are not characterized by the supplied material. The absence of writes is not a guarantee that arbitrary addresses are safe to read.

No synchronization is shown. A coherent snapshot of mutable shared state is therefore not established.

## Dependencies

No called-function dependencies are present. There are no additional opaque function IDs to list.

The lookup depends on three categories of externally maintained data:

- The primary object and its offset-80 link.
- The linked object and its offset-16 index.
- A global table-base value and the addressed table storage.

The table is not supplied as an argument. Its initialization, extent, lifetime, mutability, and ownership are unknown.

The linked object’s index is used directly. No evidence establishes how that index was produced or whether another component guarantees its validity.

## Acceptance examples

These examples are conditional acceptance criteria derived from the static representation. No runtime tests have been performed. Unless otherwise stated, all required reads and arithmetic are assumed valid and relevant state remains stable during the call.

### Initial link is zero

Given a readable primary object whose offset-80 field is zero:

- The return value is four-byte zero.
- No linked-object index read is required by this path.
- No table-value read is performed by this path.

The validity of unrelated table storage does not affect this branch.

### Index zero

Given a stable nonzero link and a linked-object index of zero, with the four bytes at table-base offset 4 equal to hexadecimal 89ABCDEF as a 32-bit value:

- The function returns those same four-byte bits.
- It does not interpret the high bit as an error.
- It does not substitute the table base or return the index.

### A later entry

Given a stable link and an index of three:

- The selected four-byte value begins 40 bytes after the table base.
- Four bytes at offset 36, the start of that entry, are not the selected return field.
- Other fields in the entry do not contribute to the result.

### A selected value of zero

Given a nonzero link, a valid index, and a selected table value of zero:

- The function returns zero.
- The return alone cannot distinguish this case from an initially absent link.

### A negative index

Given an index of −1, and assuming the resulting arithmetic and the read eight bytes before the table base are valid:

- The selected value is read from that preceding location.
- No explicit negative-index error is returned.

This is not a guarantee that such a location belongs to an intended table allocation.

### Link changes between accesses

If the initial link read is nonzero but the second read yields a different address, the supplied expression uses the second address to obtain the index. It does not explicitly fall back to the first value or repeat the zero test.

This is a source-level dependency observation, not a concurrency guarantee. Whether such a change can be observed in an executable is documented_only.

## Uncertainties

The following remain documented_only:

- The semantic names and purposes of the objects, index, table entries, and return value.
- The complete object layouts and table capacity.
- The ABI, byte order, pointer width, and alignment requirements.
- Actual instruction ordering and whether repeated ordinary reads are combined.
- Behavior under concurrent mutation.
- Arithmetic-overflow behavior and invalid-address failure behavior.
- Initialization, lifetime, and ownership of all referenced storage.

A likely intent is to retrieve metadata associated with a linked object, with zero representing absence. That interpretation is inferred, not established. The established behavior is only the conditional indexed four-byte lookup.

## Evidence

The supplied static representation establishes:

- A four-byte result initialized to zero.
- A zero test of the primary-object field at byte offset 80.
- A second occurrence of that field access on the nonzero branch.
- An index read at byte offset 16 from the resulting linked address.
- A table address formed using a 12-byte stride and a four-byte field offset.
- A four-byte read returned directly on that branch.
- No explicit calls, writes, bounds checks, or error-handling path.

No implementation execution, runtime trace, or test result was supplied or obtained. The acceptance examples express expected behavior under their stated preconditions; they are not reports of observed execution.
