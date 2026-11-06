<!-- Trusted provenance: F-dc9e9ffc7266; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dc9e9ffc7266

## Purpose

F-dc9e9ffc7266 coordinates a two-level traversal using two independently derived strides, two mutable 32-bit cursors, and three four-byte scratch regions. Each inner iteration invokes F-8b2bcc916975 and then F-94989df6c6d2. The second invocation receives one scratch region also supplied to the first, providing a possible data-transfer path between the dependencies.

The established behavior is the sequencing of these calls, the initialization and adjustment of the cursors, and the decrement-controlled traversal. The supplied source does not establish the data format processed by either dependency, their memory-access widths, or their effects on the cursor values.

A plausible intent is processing a rectangular region whose first side uses two-byte horizontal spacing and whose second side uses four-byte horizontal spacing. This is an inference from the arithmetic, not an established conversion, copying, or image-processing contract.

## Interface

### Arguments

The interface contains sixteen positional arguments. The recovered representation uses 32-bit integer-like values and opaque four-byte values. Address-like arguments and cursors are represented as integers rather than typed pointers.

| Position | Contract role |
|---|---|
| 1 | First base value. Used in the initial first cursor and copied into a separate local word. |
| 2 | First stride input. Shifted left by one bit before the first F-082b668da333 call; also copied into a separate local word. |
| 3 | Opaque four-byte value copied into local storage. No subsequent explicit use is shown. |
| 4 | First horizontal offset. Multiplied by two in the initial first cursor; also copied into a separate local word. |
| 5 | First vertical offset. Multiplied by the first returned stride. |
| 6 | Inner-loop count and row-correction input. Its original value is reused for every row. |
| 7 | Outer-loop count. Decremented after each completed inner loop. |
| 8 | Second base value. Used in the initial second cursor. |
| 9 | Second stride input. Shifted left by two bits before the second F-082b668da333 call. |
| 10 | Opaque four-byte value with no explicit use in the shown body. |
| 11 | Second horizontal offset. Multiplied by four in the initial second cursor. |
| 12 | Second vertical offset. Multiplied by the second returned stride. |
| 13 | Opaque four-byte value with no explicit use in the shown body. |
| 14 | Opaque four-byte value with no explicit use in the shown body. |
| 15 | Opaque four-byte value forwarded to the first F-082b668da333 call. |
| 16 | Opaque four-byte value forwarded to the second F-082b668da333 call. |

Roles such as “base,” “horizontal offset,” and “vertical offset” describe the arithmetic, not independently established object types or coordinate systems.

### Result

The function returns no value. On normal completion, observable results can arise through dependency side effects, including changes to memory addressed through the cursors or changes to external state.

No success flag, processed-element count, final cursor, or error code is returned.

### Local storage exposed to dependencies

The function creates:

- Two mutable 32-bit cursor cells.
- Three distinct four-byte scratch regions, called **transfer scratch**, **first auxiliary scratch**, and **second auxiliary scratch** below.

Each scratch region is supplied by address to at least one dependency. None is explicitly initialized before its first use.

Four additional local words receive arguments 1 through 4 before the stride calls. Their addresses are not explicitly passed to any dependency in the shown body. Their physical adjacency to other storage, and any use through an undocumented calling convention, are not established contractual properties.

### Preconditions

For an ordinary finite traversal:

- Both loop counts should be positive.
- Each dependency must accept the supplied argument representations.
- Any memory actually accessed by a dependency must be valid for that dependency’s operation.
- Dependencies must not access beyond the local objects supplied to them unless a separate, valid contract authorizes that access.
- Any required initialization of scratch storage must occur within the dependencies before they read it.

These are safe-use conditions, not checks performed by F-dc9e9ffc7266. This function performs no explicit count, pointer, range, alignment, or allocation-size validation.

## Behavior

### Setup and dependency order

The function first copies arguments 1 through 4 into four local words.

It then calls F-082b668da333 twice, in this order:

1. With argument 2 shifted left by one bit, followed by argument 15.
2. With argument 9 shifted left by two bits, followed by argument 16.

The first return value becomes the **first stride**. The second becomes the **second stride**. The second call occurs after the first returns; no independence or absence of shared-state effects is guaranteed.

After both calls return, the two cursor cells are initialized:

- First cursor: argument 1, plus twice argument 4, plus argument 5 multiplied by the first stride.
- Second cursor: argument 8, plus four times argument 11, plus argument 12 multiplied by the second stride.

No dependency receives a pointer to either cursor before these initializations.

### Inner iteration

Each inner iteration performs exactly these dependency invocations, in order:

1. F-8b2bcc916975 receives:
   - The address of transfer scratch.
   - The address of first auxiliary scratch.
   - The address of the first cursor cell.
2. After that call returns, F-94989df6c6d2 receives:
   - The address of second auxiliary scratch.
   - The address of the second cursor cell.
   - The address of transfer scratch.

Only after both calls return does the inner counter decrement and undergo a zero test.

Passing a cursor cell by address allows its dependency to modify the cursor. The outer function does not explicitly advance either cursor inside the inner loop. In particular, the visible body does not prove that the first dependency adds two or that the second adds four.

The transfer scratch region is shared across the two calls in an iteration. This establishes an opportunity for the first dependency to produce data consumed by the second, but does not establish which bytes are written or read.

The two auxiliary scratch regions are distinct. They are not directly passed from one dependency to the other.

### Row completion

After the inner counter reaches zero:

1. The outer counter is decremented.
2. The first cursor is adjusted by adding the first stride and subtracting twice the original inner-loop count.
3. The second cursor is adjusted by adding the second stride and subtracting four times the original inner-loop count.
4. The working inner counter is restored from the original inner-loop count.
5. The decremented outer counter is tested.

These cursor adjustments occur even after the final row. They therefore remain part of normal execution even though no later dependency call necessarily observes the resulting cursor values.

The row corrections use the current cursor contents, including any changes made by the dependencies. They do not reconstruct the cursors from their original bases.

For example, if—and only if—the first dependency advances its cursor by two on every invocation, the first cursor’s net change over one completed row is the first stride. The corresponding result holds for the second cursor if its dependency advances it by four. Those advancement rules are not supplied here.

### Loop boundaries

Both loops test their counters after executing their bodies.

Consequently:

- No count value causes an initial zero-iteration exit.
- Every normally entered invocation reaches at least one inner dependency pair after setup.
- With positive inner count \(N\) and positive outer count \(M\), normal execution performs \(N \times M\) calls to each inner dependency.
- The inner count is restored for every outer iteration.
- The original inner count also controls both row corrections.

Zero and negative counts are not rejected. Under 32-bit wraparound decrement semantics, an initial zero counter reaches zero again only after \(2^{32}\) decrements. An initially negative counter likewise requires a very large number of decrements before returning to zero. Actual completion additionally depends on all intervening calls and memory accesses succeeding.

The exact behavior of overflow cannot be guaranteed from the supplied source representation alone. Such counts must not be interpreted as a supported empty-region convention.

### Integer widths and arithmetic

The relevant scalar slots and cursor cells are represented as 32-bit quantities. Each scratch region is exactly four bytes.

Potentially significant arithmetic includes:

- Left shifts of arguments 2 and 9 by one and two bits.
- Multiplication of offsets by two or four.
- Multiplication of vertical offsets by returned strides.
- Addition of address-like values and offsets.
- Multiplication of the inner count by negative two or negative four.
- Counter decrements and row adjustments.

No overflow checks or widening operations are shown. The source representation does not establish portable signed-overflow behavior, nor does it fully resolve signed left shifts for negative or out-of-range inputs. A binary-compatible specification would need the target’s actual arithmetic semantics to define these cases precisely.

For inputs whose intermediate results are representable, the formulas above describe the arithmetic directly. Outside that range, exact results remain documented_only rather than assuming either mathematical integers or a portable wraparound guarantee.

## Side effects

### Established local effects

The function initializes and subsequently updates its cursor cells and loop-control state. It repeatedly exposes the same three scratch regions and the same cursor cells to the dependencies.

Scratch storage is not cleared between iterations or rows. Bytes written during an earlier invocation remain available in later invocations unless subsequently overwritten. There is no explicit per-row scratch reset.

### Dependency-mediated effects

All processing of externally addressed data is delegated. The shown body does not itself explicitly dereference either integer cursor as a data pointer.

The dependencies may read or write external memory or global state according to their own contracts. Their names, argument positions, and the surrounding arithmetic do not establish whether they copy, convert, allocate, release, validate, or advance through data.

No dependency can be assumed to preserve globals merely because this caller does not explicitly reference them.

### Ownership and lifetime

The caller supplies values, including values that may designate external storage. F-dc9e9ffc7266 contains no explicit allocation, release, ownership-transfer declaration, or cleanup call.

The scratch regions and cursor cells have invocation-local lifetime. Their addresses are valid during the synchronous dependency calls. The supplied body provides no lifetime support for a dependency retaining those addresses after this invocation returns.

Whether dependencies allocate or release external resources is unknown. Absence of an explicit release in this function is not evidence that no resource release occurs.

### Errors and abnormal termination

No dependency result is tested for failure. Both stride results are used directly. The inner dependency calls have no inspected result.

There is no explicit retry, rollback, exception handler, or recovery path. If a dependency fails to return normally, later calls and row adjustments need not occur. Earlier external writes may already have happened; no atomicity guarantee is established.

## Dependencies

| Opaque function ID | Invocation contract visible here | Unknown behavior |
|---|---|---|
| F-082b668da333 | Called twice during setup. Receives a shifted 32-bit input and an opaque four-byte argument. Its return is used as a stride. | Meaning of the opaque argument; validation; return range; failure signaling; external-state effects; ownership effects. |
| F-8b2bcc916975 | Called first in every inner iteration with transfer scratch, first auxiliary scratch, and the first cursor cell, all by address. | Read/write directions; scratch initialization; external access sizes; cursor advancement; alignment requirements; failure behavior. |
| F-94989df6c6d2 | Called second in every inner iteration with second auxiliary scratch, the second cursor cell, and transfer scratch, all by address. | Read/write directions; interpretation of transfer scratch; external access sizes; cursor advancement; failure behavior. |

Return values from the two inner dependencies, if any exist in their actual interfaces, are not consumed by this function.

## Acceptance examples

These are conditional contract examples, not executed tests. Unless otherwise stated, all dependencies return normally and all arithmetic remains representable.

### One element in one row

Given inner count 1 and outer count 1:

- Both stride calls occur before either inner dependency.
- F-8b2bcc916975 is called once.
- F-94989df6c6d2 is called once, after the first inner dependency returns.
- Both row corrections occur after that pair.
- The function returns without a value.

A dependency invocation after the final row corrections would contradict the visible control flow.

### Two elements in three rows

Given inner count 2 and outer count 3:

- Each inner dependency is invoked six times.
- Calls alternate between F-8b2bcc916975 and F-94989df6c6d2.
- Each pair completes before the next pair begins.
- Three sets of row corrections occur.
- There are still only two F-082b668da333 calls.

### Initial cursor calculation

Suppose the first stride result is 100, argument 1 is 1,000, argument 4 is 3, and argument 5 is 2. The first cursor is initialized to 1,206.

Suppose the second stride result is 200, argument 8 is 5,000, argument 11 is 4, and argument 12 is 3. The second cursor is initialized to 5,616.

These examples establish cursor arithmetic only. They do not imply those cursor values identify valid memory.

### Conditional row progression

Given inner count 3:

- If F-8b2bcc916975 advances its cursor by two per call, the three inner calls advance that cursor by six; the row correction then removes six and adds the first stride.
- If F-94989df6c6d2 advances its cursor by four per call, its corresponding net row change is the second stride.

If either dependency leaves its cursor unchanged instead, the row correction still subtracts six or twelve respectively. The caller does not compensate based on observed progress.

### Zero counts

With inner count zero, the first inner dependency pair still executes. With outer count zero, the first row still begins.

An implementation that immediately returns for either zero count would introduce behavior absent from the supplied body. The exact eventual outcome for these inputs depends on arithmetic semantics and dependency behavior.

### Unused arguments

Changing only argument 10, 13, or 14 does not change any explicit computation or dependency argument in the shown body. This establishes lack of direct use, not a guarantee about undocumented calling conventions or arbitrary inspection of surrounding machine state.

## Uncertainties

The following remain documented_only:

- The processed data types, formats, and conversion semantics.
- Whether either base is a readable or writable address.
- The semantic units of returned strides.
- Exact target overflow and signed-shift behavior.
- The dependencies’ scratch read/write directions and initialization guarantees.
- Whether dependencies advance cursors, and by how much.
- Required alignment, accessible memory extents, and aliasing restrictions.
- Whether overlapping external regions are safe.
- Whether dependency side effects include allocation, release, or global-state mutation.
- Error signaling and behavior after a dependency fails.
- Any significance of the four local copies beyond their explicit initialization.
- Any undocumented ABI-level use of otherwise unused arguments.

The two-byte and four-byte row corrections support an inferred traversal intent. They do not resolve these unknowns.

## Evidence

This contract is based solely on static inspection of the supplied function body and the supplied opaque dependency mapping.

Established evidence includes the sixteen-argument interface, two ordered stride calls, cursor initialization formulas, three four-byte scratch regions, ordered inner dependency calls, post-tested loops, and unconditional row corrections after every completed row.

Rectangular traversal and two-byte-to-four-byte processing are inferred intent only. Dependency internals and target-specific overflow behavior were not supplied.

No runtime tests have been performed. No processed-data result, memory-safety property, or dependency behavior has been dynamically verified.
