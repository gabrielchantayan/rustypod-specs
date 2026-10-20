<!-- Trusted provenance: F-1a1ecaac8aad; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1a1ecaac8aad

## Purpose

F-1a1ecaac8aad conditionally copies one four-byte value from indexed storage into caller-designated storage and returns a four-byte status.

The established behavior has two paths:

- A null destination produces status `0xFFFFFFCE`, without accessing the indexed storage or writing an output value.
- A non-null destination causes an indexed four-byte read followed by a four-byte destination write. If those operations complete normally, the status is zero.

Interpreting the indexed storage as a table and the selector as an entry number is inferred intent. The supplied representation does not establish a table length, an application-level meaning for its entries, or a named error category.

## Interface

The function has two inputs and one return value.

| Element | Contract |
|---|---|
| Selector | An integer used to locate a four-byte entry relative to a base address obtained from global storage. |
| Destination | A pointer to storage receiving exactly one four-byte value on the non-null path. A null pointer selects the error-return path. |
| Return value | A four-byte status: `0xFFFFFFCE` for a null destination, or `0x00000000` after a completed copy. |

The recovered interface represents the selector as an integer. A 32-bit signed selector is the apparent intended interface convention, but the supplied material does not independently establish the original declaration or all machine-level integer semantics.

The returned bit pattern `0xFFFFFFCE` represents −50 when interpreted as a signed 32-bit two’s-complement integer, or 4,294,967,246 when interpreted as unsigned. The contract establishes the bit pattern, not an external error-code name.

The copied value is an uninterpreted 32-bit quantity. No evidence identifies it as a pointer, signed number, bit field, or application-specific object.

For a successful, defined memory access, the caller and surrounding environment must provide:

- A readable base-address value in the global storage consulted by the function.
- A readable four-byte region at the selected effective address.
- A writable four-byte region at the destination.
- Address calculations and alignment compatible with the actual execution environment.

Only destination nullness is explicitly checked. A non-null destination does not establish that any of these other conditions hold.

## Behavior

### Null destination

The destination is tested for nullness before the indexed read is attempted.

When it is null, the function returns `0xFFFFFFCE`. This path contains no access to the global base-address storage, no selected-entry read, and no destination write.

The selector does not affect this result. A selector that would otherwise designate inaccessible storage does not cause an indexed access on this path.

This is the only explicitly handled error condition.

### Non-null destination

For a non-null destination, the observable data-access sequence represented by the supplied material is:

1. Obtain the base address from global storage.
2. Read a four-byte value at the address selected relative to that base.
3. Write the retrieved four-byte value to the destination.
4. Return zero after the write completes normally.

The selected address is the base plus the selector multiplied by four. Thus, selector zero addresses the base itself; selector one addresses four bytes beyond it; selector two addresses eight bytes beyond it.

There is no explicit bounds check, sign check, destination-capacity check, source-validity check, or alignment check. There is also no explicit validation of the base address obtained from global storage.

The selected source value is read before the destination assignment. The function does not reread the selected entry after writing the output.

### Width and arithmetic

Each selected entry occupies four bytes for purposes of address stepping and transfer. The destination receives four bytes, not a variable-length entry or an entire table.

The supplied representation does not show selector normalization, saturation, or range limiting. It also does not establish a portable outcome for overflowing address calculations.

A negative selector is not explicitly rejected. Under a signed interpretation and ordinary representable address arithmetic, a selector of −1 designates the four-byte region immediately before the base. Whether that region is readable is an external memory-layout condition.

For selectors whose scaling or addition exceeds the applicable integer or address range, the outcome is **documented_only**. The representation is insufficient to choose among machine wraparound, an invalid effective address, or other consequences of the actual instruction sequence. No overflow-error return is established.

### Aliasing and exceptional accesses

No source/destination non-aliasing condition is checked.

If the destination identifies exactly the selected source location, the represented operation reads that location and then writes the retrieved value back to it. This still takes the non-null path and returns zero if both accesses complete normally.

Partial overlap is not explicitly rejected. The logical source read precedes the destination write, but the supplied representation does not establish bytewise transfer order, access atomicity, or fault behavior for overlapping or unaligned regions.

A destination can also alias storage with significance beyond this call, including the global base-address storage if the surrounding layout permits it. In that case, the destination write can change the base used by later calls. The current source read has already occurred before that write.

If an access faults or otherwise fails outside the explicitly represented null check, no recoverable status, rollback, or completed return is established.

## Side effects

On normal completion with a non-null destination, the function performs one logical four-byte destination write.

There is no separately represented update to the selector, global base-address storage, or selected source entry. Nevertheless, any of those storage locations that overlap the destination can be changed by the output write.

The supplied representation contains no allocation, deallocation, ownership transfer, resource acquisition, logging, callback, or explicit synchronization. It does not establish any requirement that the destination be newly allocated or exclusively owned.

The caller remains responsible for the validity and lifetime of destination storage. The function’s write does not itself establish ownership of that storage or of any object whose representation happens to be copied.

The logical accesses must not be assumed atomic merely because they are four bytes wide. Thread safety, memory ordering, concurrent modification behavior, and suitability for memory-mapped devices are **documented_only**.

## Dependencies

No called functions are visible, so there are no opaque function dependencies to enumerate.

The visible data dependencies are:

- Global storage containing the base address.
- The four-byte region selected relative to that base.
- The caller-designated four-byte destination.

The supplied material does not identify who initializes the base, how long the underlying storage remains valid, or whether another actor can change either the base or an entry during execution.

There are no visible callees whose preservation of globals, ownership behavior, or cleanup behavior can be assessed or relied upon.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence. They are not reports of executed tests.

### Null destination bypasses indexed storage

Given a null destination, the expected result is the four-byte status `0xFFFFFFCE`. No indexed-storage access or destination write occurs.

This criterion applies regardless of the selector and does not require the global base or selected region to be usable, because this path does not consult them.

### Selector zero copies the base entry

Given a non-null writable destination, a readable base-address value, and a readable four-byte value `0x12345678` at that base, selector zero causes the destination to receive `0x12345678` and the function to return zero.

This specifies the transferred 32-bit value without asserting a byte order.

### Consecutive selectors use four-byte spacing

Given readable four-byte entries at the base, base plus four, and base plus eight, selectors zero, one, and two select those respective locations.

This criterion applies only when the calculations and accesses are valid in the execution environment. It does not establish that any further entry exists.

### Negative selector is not rejected explicitly

Given a signed selector of −1, representable effective-address arithmetic, a readable four-byte region immediately before the base, and a valid destination, the represented operation copies that preceding value and returns zero.

Without those conditions, no specific fault or status is established.

### Identical source and destination retain the value

Given a valid selected location that is also the destination, the function reads its four-byte value and writes that value back. Normal completion returns zero.

This does not establish that the write is omitted or unobservable to instrumentation or special-purpose memory.

### Invalid non-null destination has no specified error status

Given a non-null destination that is not writable, the null check does not reject it. The represented path attempts the indexed read and subsequent destination write. No particular exception, recovery behavior, or return value is promised if that write cannot complete.

## Uncertainties

The following remain **documented_only**:

- Original selector declaration and exact machine-level arithmetic rules.
- Address width, endianness, alignment requirements, and instruction-level access granularity.
- Table extent, permitted selector range, and entry meaning.
- Initialization, ownership, lifetime, and mutability of the base and indexed storage.
- Overflow and invalid-address behavior.
- Atomicity, concurrency guarantees, and interaction with memory-mapped devices.
- External meaning of status `0xFFFFFFCE`.
- Fault handling and whether a failed destination write can partially modify storage.

Inferred intent is limited to a status-returning indexed-value accessor. Bounds safety, immutability, synchronization, and ownership conventions are not established.

## Evidence

The sole evidence is the supplied recovered function representation for F-1a1ecaac8aad.

It directly supports the destination-null branch, the two return bit patterns, the globally based indexed read with four-byte spacing, the four-byte output assignment, and the absence of visible calls or additional validation.

It does not provide the underlying instructions, surrounding declarations, storage definitions, callers, or runtime observations. No runtime tests have been performed.
