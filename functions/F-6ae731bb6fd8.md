<!-- Trusted provenance: F-6ae731bb6fd8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6ae731bb6fd8

## Purpose

F-6ae731bb6fd8 converts a byte-counted payload into a temporary sequence of 16-bit units, submits that sequence to an opaque dependency, and conditionally submits it to a second dependency. On normally completed nonempty-input paths, it writes a 32-bit result to caller-provided storage and returns the first dependency’s status.

Payload bytes below 128 convert directly. Bytes at or above 128 select values through globally obtained table bases and may produce either one or two units.

**Inferred intent:** The arrangement resembles character conversion followed by primary processing and conditional fallback. Neither the intended encoding nor the application meaning of the result is established. This contract treats table entries, dependency results, and statuses as opaque numeric values.

## Interface

The function accepts four arguments, in order:

1. **Input address:** Address of an unsigned 8-bit count followed by the counted payload.
2. **Opaque argument A:** A 32-bit value forwarded unchanged to F-8a6756c94146.
3. **Result destination:** Address receiving one 32-bit value on a normally completed nonempty-input path.
4. **Opaque argument B:** A 32-bit value forwarded unchanged to F-8a6756c94146.

The return value is a 32-bit integer status:

- Empty input returns **22**.
- Nonempty input returns the status from F-8a6756c94146.
- Calling F-54585861452f does not replace that status.

The return status and destination value are distinct outputs. Empty input does not cause a destination write.

### Input layout and access requirements

The first byte gives the payload length \(N\), from 0 through 255. The function consumes exactly the next \(N\) bytes in increasing address order.

There is no terminator scan. A zero-valued payload byte is processed normally and emits a zero-valued unit.

The initial count byte must be readable. For nonempty input, all selected payload bytes must also be readable. The interface supplies no independent storage length and performs no bounds validation.

### Temporary representation

The conversion buffer contains **76 slots of 16 bits each**, totaling **152 bytes**. Dependencies receive its starting address and a byte length.

Let:

- \(N\) be the payload-byte count.
- \(E\) be the number of payload bytes that trigger two-unit expansion.
- \(M=N+E\) be the emitted-unit count.

Conversion fits the declared buffer only when \(M\leq76\). This condition is not checked.

The function does not append a terminator or initialize unused slots. The supplied length covers only emitted units.

A separate local 32-bit result slot is passed to the first dependency. The function does not initialize that slot before the call.

## Behavior

### Count acquisition and traversal

The function reads the initial count byte once. Its unsigned value initializes both the payload traversal count and the emitted-unit count.

The traversal condition uses the remaining count before decrementing it. Each decrement is masked to 16 bits. The final condition observes zero and terminates traversal even though the accompanying decrement wraps to 65,535. That wrap does not cause another payload read.

The original count byte is not reread.

### Direct conversion

For each payload byte from 0 through 127, the function writes exactly one 16-bit unit containing the same unsigned numeric value.

Neither table-base global nor either table entry is read for this conversion.

### Table-based conversion

For each payload byte from 128 through 255:

1. The lower seven bits determine an index from 0 through 127.
2. The function reads the global value supplying the primary-table base.
3. It reads a 16-bit primary entry at a byte displacement of twice the index from that base.
4. It writes that entry as the next temporary unit.
5. It evaluates whether the primary value requires expansion.
6. If expansion applies, it increases the emitted-unit count by one, then reads the global value supplying the secondary-table base.
7. It reads the selected 16-bit secondary entry at the same index and writes it immediately after the primary unit.

The secondary-table base read therefore occurs **after the primary unit has been written and after the emitted-unit count has increased**.

These are access-time base acquisitions. The supplied body does not establish that either base is captured once for the whole invocation, that repeated acquisitions return the same address, or that the pointed-to data is immutable. A high-bit byte that does not expand does not cause a secondary-base read.

Payload order is preserved. An expansion’s primary and secondary units are adjacent and precede units emitted for later payload bytes.

### Expansion condition and arithmetic

Under the displayed types and ordinary integer-promotion rules, expansion occurs when the unsigned 16-bit primary value is **less than 123**.

The primary value promotes to a signed 32-bit integer before subtracting 65. Values below 65 consequently produce a negative difference, which still compares as less than 58.

Thus:

- Primary values **0 through 122** trigger expansion.
- Primary values **123 through 65,535** do not.

The expansion range is not limited to 65 through 122. This rule applies only to table-based conversion: a directly converted payload byte of 122 still emits one unit.

This is a conclusion from the supplied expression and types. Whether those types reproduce the original machine-level comparison exactly remains **documented_only**.

Each payload byte emits at most two units. Consequently, \(M\leq510\), and the calculated byte length \(2M\leq1{,}020\). The emitted-count increment and length calculation fit within 32 bits. Buffer capacity, not arithmetic overflow, limits supported conversion.

### Empty input

When \(N=0\):

- No payload byte is read.
- Neither table-base global nor any table entry is read.
- Neither dependency is called.
- The result destination is not written.
- The function returns 22.

The result destination need not be accessible for this path, because the function does not dereference it.

### First dependency call

For nonempty input, conversion completes before F-8a6756c94146 is called. Its arguments, in order, are:

1. Address of the temporary sequence.
2. Sequence byte length, \(2M\).
3. Opaque argument B.
4. Address of the local 32-bit result slot.
5. Opaque argument A.
6. Numeric value 58.
7. Numeric value 0.

The function retains the returned status for its own return value.

No supplied evidence establishes that the dependency leaves the buffer unchanged, initializes the result slot on every status, preserves global state, or refrains from retaining addresses.

### Conditional second dependency call

If, and only if, the first dependency returns exactly **63**, the function calls F-54585861452f with:

1. The same temporary-buffer address.
2. The same sequence byte length.
3. Numeric value 58.
4. Numeric value 0.

The second dependency’s 32-bit return value replaces the local result-slot value. The function still returns the first dependency’s status, namely 63.

The buffer is not reconstructed between calls. Any first-dependency modifications remain present for the second call. The function also performs no restoration of global or other memory state between dependencies.

For every first-dependency status other than 63, the second dependency is skipped.

### Result publication

On every nonempty path that reaches the final write, the function copies the local 32-bit result value to the result destination.

Publication is not conditioned on a success interpretation of the status. A non-63 status still leads to publication, even if the first dependency did not initialize its output slot. In that case, no meaningful destination value or defined completion guarantee is established by the supplied body.

If a dependency does not return normally, the body establishes no guarantee that the final destination write occurs.

## Side effects

### Ordered direct accesses

The direct data-access order is:

1. Read the input count.
2. For each payload byte, in increasing address order:
   - Read the payload byte.
   - For a low-bit byte, write its directly converted unit.
   - For a high-bit byte:
     - Read the primary-table-base global.
     - Read the selected 16-bit primary entry.
     - Write the primary unit.
     - If expansion applies, increase the emitted-unit count, read the secondary-table-base global, read the selected 16-bit secondary entry, and write the secondary unit.
3. For nonempty input, call F-8a6756c94146.
4. If its status is 63, call F-54585861452f and use its return value as the result.
5. Read the local result value and write one 32-bit value through the result destination.

This ordering describes accesses expressed by the supplied body, not a concurrency or synchronization guarantee.

### Direct writes and aliasing

There are no explicit assignments through the input address or lookup-table access expressions. That fact does **not** establish that input, tables, or global storage remain unchanged.

The caller-provided result destination is not checked for overlap. Its final 32-bit write can modify:

- Input storage.
- Primary- or secondary-table storage.
- Storage holding either table-base global.
- Other global or caller-visible storage.

Those effects occur when the destination aliases the corresponding region. A write overlapping only part of a larger object can modify that part; object sizes and pointer widths beyond the stated value widths are not established.

All direct input, table-base, and table-entry reads finish before result publication. Therefore, destination aliasing alone does not retroactively alter the values already used during conversion. It can still change state visible after the call, including lookup configuration used by later invocations.

Dependencies execute before publication and may independently change memory or global state. Their effects are not restricted by the absence of explicit assignments in this function.

### Ownership and lifetime

Input and result storage are caller-provided. The body contains no explicit allocation, deallocation, or release operation.

The conversion buffer and result slot have invocation-local lifetimes. Their addresses are exposed to F-8a6756c94146; the buffer address may also be exposed to F-54585861452f.

Passing these addresses does not establish ownership transfer. Safe use requires dependencies not to access local storage after its lifetime ends. Whether they actually retain pointers, release memory, or assume ownership is unknown.

### Invalid access and capacity violations

The body does not validate:

- Null, inaccessible, or insufficient input storage.
- Table-base values or selected table-entry accessibility.
- Alignment requirements.
- The 76-unit capacity.
- Result-destination accessibility on a nonempty path.

Emitting a 77th unit writes outside the declared array before dependency processing. No supported output, return status, or subsequent access sequence is guaranteed after that violation.

There is no explicit conversion of invalid addresses or capacity overflow into status 22 or any other error status.

## Dependencies

### F-8a6756c94146

Called once after nonempty conversion, provided execution reaches the call. It receives the temporary sequence, byte length, both opaque arguments, a 32-bit output-slot address, and constants 58 and 0.

Its status becomes this function’s returned status. Exactly 63 selects the second dependency.

The dependency’s status meanings, output-initialization guarantees, buffer mutation, global effects, allocation behavior, ownership assumptions, and pointer retention are unknown.

### F-54585861452f

Called once only when the first dependency returns 63. It receives the same buffer and length plus constants 58 and 0.

Its 32-bit return value becomes the published result, not the returned status. Its internal processing, memory effects, and ownership behavior are unknown.

### Lookup data and global bases

Two global values supply bases for indexed 16-bit lookup data. Each selected index addresses a two-byte entry.

The primary base is obtained for every high-bit payload byte. The secondary base is obtained only for an expanding byte, after primary-unit publication into the temporary buffer and the emitted-count increase.

The source does not establish base-value immutability, table immutability, initialization, synchronization, contents, encoding, or disjointness from caller-provided storage. It also does not establish a portable byte order for the 16-bit values.

## Acceptance examples

These are conditional static acceptance cases, not runtime observations. Unless stated otherwise, assume valid addresses, compatible alignment, sufficient temporary capacity, normally returning dependencies, and a defined result supplied by the applicable dependency.

1. **Empty input.**  
   A count of zero returns 22. No payload, table-base global, or table entry is read. Neither dependency runs, and the destination is untouched.

2. **Direct values, including zero.**  
   A count of three with payload values 0, 65, and 127 emits those three numeric units in order. No table bases are read. The first dependency receives byte length six.

3. **Primary-only lookup.**  
   A single high-bit payload byte selects a primary entry of 123. The primary base is read before that entry. One unit, 123, is emitted; no secondary base or entry is read. The submitted byte length is two.

4. **Expansion below 65.**  
   Under the displayed promotion semantics, a selected primary value of 64 expands. After writing 64 and increasing the emitted count, the function obtains the secondary base. If its selected entry is 900, the sequence is 64 followed by 900, with byte length four.

5. **Expansion boundary.**  
   Primary value 122 expands; primary value 123 does not. A direct payload byte of 122 emits one unit without any lookup.

6. **Repeated base acquisition.**  
   Two high-bit bytes each require a primary-base read. If the base values obtained at those accesses differ, each entry is selected relative to the base obtained for that byte. No single invocation-wide snapshot is established.

7. **Non-63 status.**  
   If F-8a6756c94146 writes result \(R\) and returns 7, the second dependency is skipped, \(R\) is published, and the function returns 7. This example assigns no success or failure meaning to 7.

8. **Status-63 branch.**  
   If the first dependency returns 63 and the second returns \(R\), the destination receives \(R\), and the function returns 63. A result previously written by the first dependency is replaced.

9. **Missing result initialization.**  
   If the first dependency returns a non-63 status without initializing its result slot, the body still reaches the publication operation. No particular destination value or defined completion is guaranteed.

10. **Aliased destination.**  
    If the destination overlaps writable input, table data, or table-base storage, the final 32-bit publication modifies that overlapping storage. Conversion’s direct reads have already completed; the function does not preserve the overwritten state.

11. **Capacity boundary.**  
    Exactly 76 emitted units fit the declared array. A 77th unit exceeds it. The latter case has no supported final result or status guarantee.

## Uncertainties

The following remain **documented_only**:

- Application meaning, encoding, and status interpretation.
- Table contents and initialization.
- Exact representation and width of table-base values.
- Whether bases or entries remain stable between accesses.
- Concurrency behavior and synchronization requirements.
- Correspondence between displayed promotion semantics and the original machine-level comparison.
- Storage byte order and precise alignment requirements.
- Dependency mutation, global effects, pointer retention, allocation, release, and ownership assumptions.
- First-dependency result initialization on non-63 statuses.
- Outcomes after invalid accesses, indeterminate-result use, or capacity violations.

These unknowns do not justify assuming immutable globals, nonaliasing storage, preserved buffers, or successful result initialization.

## Evidence

**Established static evidence:** The supplied body shows one unsigned count-byte read, ordered payload traversal, direct and table-based 16-bit emission, access-time reads of global table bases, a 76-unit temporary array, empty-input status 22, the two dependency call conditions, and final 32-bit result publication on normally completed nonempty paths.

**Width conclusions:** Payload count is at most 255. At most one additional unit is emitted per byte, giving at most 510 calculated units and 1,020 calculated bytes. These arithmetic bounds do not make oversized conversion safe.

**Aliasing conclusion:** The destination write is unrestricted by an overlap check. Absence of assignments through input or table expressions is therefore insufficient to claim those regions or globals remain unchanged.

**Inferred intent:** Conversion followed by primary processing and conditional fallback is a plausible interpretation, not an additional guarantee.

**Verification limit:** No runtime tests have been performed. All acceptance examples derive from the supplied static evidence and their stated conditions.
