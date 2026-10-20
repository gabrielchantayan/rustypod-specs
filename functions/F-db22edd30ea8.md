<!-- Trusted provenance: F-db22edd30ea8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-db22edd30ea8

## Purpose

F-db22edd30ea8 provides a bounded, first-match lookup over six integer-valued positions. The positions have fixed numeric address offsets of 0, 8, 16, 24, 32, and 40 from a starting-address value obtained indirectly through an input pointer.

Under stable, valid storage conditions, the result is the integer-encoded address of the earliest position whose integer value equals the search key. If none of the six values equals the key, the result is zero.

**Established static evidence:** six searchable positions, ascending search order, an eight-unit address increment, integer equality, immediate success termination, and zero on exhaustion.

**Inferred intent:** the searched storage is a small table with a key field at the beginning of each entry. Neither a complete record format nor the meaning of storage between compared integers is established.

## Interface

### Inputs

The routine accepts two inputs:

| Input | Required interpretation |
|---|---|
| Address-holder pointer | Identifies an integer object whose contents supply the numeric starting address. It is not itself the starting address of the searched storage. |
| Search key | An integer compared for equality with each examined candidate integer. |

The interface has no length, capacity, stride, comparator, output buffer, or separate status argument. The search bound and numeric spacing are fixed.

### Output

The result has the same signed integer type used for the key and the address-holder contents.

For an ordinary execution with an unchanged address holder:

- Success returns the starting-address integer plus the offset of the earliest matching position.
- Exhaustion returns zero.

The returned value is an integer address representation, not a pointer-typed result and not the matched key. Successful return does not copy a record or provide its size.

There is no independent success indicator. If the target permits a successful candidate to have integer address representation zero, success at that candidate and exhaustion have the same returned value.

### Widths, arithmetic, and layout

The supplied declaration establishes signed integer types but does not establish their bit width. A particular 32-bit ABI must not be assumed.

The fixed offsets and the internal search range are small enough to fit within the minimum required range of the source integer type. The potentially problematic arithmetic is addition of an offset to the starting-address integer. Required additions must remain representable for ordinary defined signed-arithmetic behavior.

The numeric offset is applied before the resulting integer is interpreted as an address. Consequently:

- The spacing is not scaled by the size of an integer.
- An offset of eight does not mean eight integer objects.
- The source does not independently establish that an increment of eight in the integer address representation corresponds to eight bytes.
- Integer-to-address conversion, alignment, and address validity depend on the target implementation.

Each examined position contributes one integer value beginning at its derived address. No other field is inspected. There is no established record header, length field, terminator, or metadata layout.

Conditionally, on a target with four-byte integers and ordinary byte-addressed integer address representations, the compared fields occupy four bytes beginning at byte offsets 0, 8, 16, 24, 32, and 40. The final field then extends through byte offset 43. This is an example target interpretation, not an unconditional ABI requirement.

### Preconditions

For defined operation:

- The address-holder pointer must identify a live, readable integer object whenever the routine consults it.
- Every integer address actually used for a candidate read must convert to a valid, suitably aligned address for that read.
- Required address additions must be representable.
- The memory accesses must satisfy the target’s applicable lifetime, access, and concurrency rules.

Only reached positions require candidate reads. A successful early comparison does not require later positions to be readable.

A stable address holder is additionally required to interpret the operation as a search over one fixed six-position region. The body does not validate or enforce that stability.

## Behavior

### Selection contract

With a stable address holder and stable candidate values, success selects the lowest-numbered matching position among the six fixed positions.

Matching is ordinary integer equality. Negative values and zero are not special cases. There is no ordering comparison, range comparison, masking, string interpretation, or sentinel termination.

Duplicate keys do not create ambiguity in selection: the earliest matching position wins. Later positions are not examined after success.

A complete miss requires six unsuccessful comparisons. It does not cause an additional read beyond offset 40.

### Address-holder observations

The starting-address integer is not contractually captured once for the entire operation. Each attempted comparison obtains its address basis through the address-holder pointer. Successful result construction consults the holder again.

This distinction affects what a successful result means:

- If the holder remains unchanged, the result identifies the location whose value matched.
- If a supported execution environment allows the holder’s value to change between observations, successive comparisons may use different address bases.
- If such a change occurs between a successful comparison and result construction, the result uses the later address-holder value and need not identify the location just compared.

The last two cases describe dependencies visible in the supplied expressions. They do not establish that concurrent modification, asynchronous mutation, or device-backed storage is supported or well-defined. That requires an external memory and execution contract.

### Observable read ordering

The source establishes these ordering constraints:

| Event | Established relationship |
|---|---|
| Candidate address formation | Depends on obtaining the address-holder value for that comparison. |
| Candidate value read | Uses the address derived for that comparison. |
| Later candidate comparison | Occurs only after all earlier attempted comparisons have failed. |
| Successful result construction | Follows the successful comparison and includes another address-holder evaluation. |
| Exhaustion result | Returns zero without an additional address-holder evaluation for result construction. |

These are source-level data dependencies and control-flow relationships, not guarantees about individual machine instructions or physical memory transactions. The supplied accesses are not declared volatile or atomic. Compiler reuse of values and other permitted transformations prevent a portable promise of exact hardware load counts.

### Invalid inputs and failures

There is no explicit argument validation, recoverable error branch, exception translation, or error-status update.

A null address-holder pointer is not an empty-search request. A zero stored in a valid address holder is likewise not checked as a special empty-table marker; it participates in address formation.

Invalid candidate storage, invalid integer-to-address conversion, or unrepresentable address arithmetic is outside the defined-operation preconditions. Such failures must not be specified as returning zero. Zero is the ordinary exhaustion result, not a general failure code.

No target-specific trap, signal, exception, or recovery behavior is established.

## Side effects

The routine contains no explicit writes to caller-visible memory. It does not modify the address holder, candidate integers, or an output object.

It also contains no explicit allocation, deallocation, ownership transfer, reference-count operation, callback, or callable dependency.

Ownership and lifetime remain the caller’s responsibility:

- Supplying an address does not transfer ownership.
- Returning an encoded address does not extend the lifetime of the referenced storage.
- A successful result does not establish permission to write through a subsequently reconstructed pointer.
- The routine does not retain either input for later use.

The absence of explicit writes does not establish that every permitted memory read is externally inert. Reads from device-backed or otherwise special storage require a separate platform contract. Nor does the body establish synchronization with another execution context.

Storage reached through either input may belong to a global object, heap object, or another storage class. The body does not identify or distinguish those ownership arrangements.

## Dependencies

There are no callable dependencies in the supplied body, so no additional opaque function IDs apply.

The behavior nevertheless depends on external data and implementation rules:

- The current contents and validity of the address-holder object.
- The integer values at the candidate addresses actually reached.
- Signed integer width and representation.
- Integer-to-address conversion and address validity rules.
- Alignment, object lifetime, and permitted memory access.
- Any externally imposed storage-stability or synchronization guarantees.

There is no evidence of a helper that validates storage, pins a table, preserves global state, or acquires or releases memory. None of those responsibilities may be attributed to an unknown callee.

## Acceptance examples

These are conditional expectations derived from static evidence, not executed tests. Unless an example states otherwise, assume a stable address holder, stable candidate values, representable arithmetic, and valid aligned candidate reads.

For the numeric examples, additionally assume that integer address representation 4096 is usable as the starting address and that the listed numeric offsets map to the intended candidate locations.

### Earliest position matches

If the integer at address representation 4096 equals the key, the result is 4096.

No candidate at offsets 8 through 40 is read. Those later locations need not be accessible for this execution. The address holder must remain readable for successful result construction.

### Interior match

If the values at address representations 4096 and 4104 differ from the key, while the value at 4112 equals it, the result is 4112.

No candidate beyond 4112 is examined. The integer stored at 4112 is not itself the returned value unless it happens numerically to equal that address representation.

### Duplicate matches

Suppose positions one and four match, while position zero does not. The result is 4104, identifying position one.

The later duplicate does not affect selection and is not examined.

### Final-position match

If the first five candidate values differ from the key and the value at address representation 4136 equals it, the result is 4136.

All six candidate reads are required on this path.

### Complete miss

If all six candidate values differ from the key, the result is zero.

There is no candidate read at address representation 4144, corresponding to offset 48.

### Zero and negative keys

A key of zero matches a candidate integer containing zero. A negative key matches an equal negative candidate integer.

Neither case shortens the bound, introduces a terminator rule, or changes the first-match policy.

### Inaccessible later storage

If position zero matches and all later candidate locations are inaccessible, the later locations are not read.

If position zero fails and the next required candidate is inaccessible, there is no established ordinary result. The inaccessible read is not converted into a miss.

### Successful return after a permitted holder change

Conditionally, suppose an external execution model makes a holder change between observations valid and observable. If the comparison at offset 16 succeeds using starting-address value 4096, but successful result construction observes starting-address value 8192, the return expression yields 8208, provided that addition is representable.

This example does not authorize unsynchronized concurrent modification. It demonstrates why a promise that the result identifies the compared location requires holder stability.

### Zero-valued successful result

If the target permits a valid integer read at an address represented by zero and that first candidate matches, the result is zero.

The interface cannot distinguish that success from exhaustion. Whether such a readable address exists remains unresolved.

## Uncertainties

The following are **documented_only**:

- Exact integer and pointer widths, including their relative sizes.
- Byte order, address-unit size, and the mapping from integer address values to locations.
- Complete storage layout beyond the compared integer at each numeric offset.
- Whether callers guarantee holder and candidate stability.
- Whether special memory or an external concurrency model applies.
- Platform behavior after invalid access, invalid conversion, or signed arithmetic overflow.
- Whether caller or platform constraints exclude zero from successful address results.

The fixed-table interpretation is inferred intent. It must not be expanded into an undocumented record type, ownership protocol, or validation guarantee.

The six-position bound, comparison order, equality predicate, successful return dependency, exhaustion result, and absence of explicit memory writes or calls are established static evidence.

## Evidence

The supplied body directly supports the bounded search, fixed numeric spacing, first-match selection, repeated address-holder dependencies, and zero-on-exhaustion behavior described above.

The supplied declaration establishes signed integer inputs and result, with a pointer to an integer used for the address holder. It does not establish a target ABI or a complete address-representation contract.

No caller implementation, platform memory contract, compiler output, or runtime observations were supplied. No runtime tests have been performed. The acceptance examples are conditional behavioral requirements, not reported measurements.
