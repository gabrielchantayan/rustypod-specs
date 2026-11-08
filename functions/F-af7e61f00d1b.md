<!-- Trusted provenance: F-af7e61f00d1b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-af7e61f00d1b

## Purpose

F-af7e61f00d1b derives a 16-bit result from an unsigned query and a memory-resident table reached through a caller-supplied object. It first invokes F-933d555a9096, then, if that dependency returns nonzero, locates a table record and applies either a direct additive mapping or an indirect 16-bit lookup.

The established behavior is a table lookup with optional search acceleration. Interpreting the table as a character-to-glyph map is plausible but not established by the supplied evidence. This contract therefore uses neutral terms such as “query,” “boundary,” and “result.”

The routine has no explicit memory writes. Its dependency may have side effects, including changes to the object or other state; those effects are not specified here.

## Interface

### Inputs

- **Object reference:** An address-like argument used by F-933d555a9096 and subsequently as the base of an object containing a table pointer at byte offset `0x74`.
- **Query:** An unsigned 32-bit value. The routine compares this full-width value against zero-extended 16-bit table values. It does not truncate the query before searching.

The supplied representation uses 16-bit unsigned table elements and 32-bit unsigned arithmetic. Address calculations are consistent with a 32-bit address model. The exact machine ABI, byte order, and alignment requirements are not independently established.

### Output

The return value is unsigned 32-bit, but its upper 16 bits are always zero on any normally returning path.

A zero return can mean any of the following:

- F-933d555a9096 returned zero.
- The selected record’s lower boundary exceeds the query.
- An indirect table entry was zero.
- A direct or indirect calculation produced zero after reduction to 16 bits.

There is no separate error indicator distinguishing these outcomes.

### Referenced layout

After a nonzero dependency return, the routine reads the table pointer from object byte offset `0x74`. Relative to the resulting table address:

| Byte offset | Width | Use |
|---|---:|---|
| `0x06` | 16 bits | Byte displacement, called **S** below |
| `0x08` | 16 bits | Initial search displacement, read only when S exceeds 15 |
| `0x0A` | 16 bits | Search-control value, read only when S exceeds 15 |
| `0x0C` | 16 bits | Alternate-start displacement, read only when S exceeds 15 |
| `0x0E` | — | Base of the searchable boundary region, called **B** below |

All three displacements are byte offsets, not counts of 16-bit elements. In particular, S is added directly to addresses without multiplication by two.

No length, allocation size, ownership descriptor, or independently checked table terminator is provided through this interface.

## Behavior

### 1. Invoke the dependency before examining the table

The routine calls F-933d555a9096 with the object argument and a second argument of zero.

If that call returns zero, the routine immediately returns zero. On this path, F-af7e61f00d1b itself does not read the table pointer or table contents. This does not constrain what the dependency reads or writes.

If the call returns nonzero, processing continues using the object’s table pointer as read **after** the call. A table-pointer change made by the dependency can therefore affect the lookup.

The dependency’s precise nonzero return value is otherwise unused.

### 2. Read the layout and establish the starting address

The routine reads the table pointer, then reads S as an unsigned 16-bit value at table byte offset `0x06`.

The ordinary scan starts at B, the table address plus `0x0E`.

When S is at most 15:

- The acceleration fields at offsets `0x08`, `0x0A`, and `0x0C` are not read.
- No pivot value is read.
- The routine proceeds directly to the forward scan from B.

When S exceeds 15, the routine:

1. Reads the initial search displacement from offset `0x08`.
2. Reads the search-control value from offset `0x0A`.
3. Reads the alternate-start displacement from offset `0x0C`.
4. Reads a 16-bit pivot at B plus the initial search displacement.
5. Selects the alternate start, B plus the alternate-start displacement, if the pivot is less than or equal to the query. Otherwise, the current start remains B.

The pivot read occurs on every S-greater-than-15 path, regardless of the query or search-control value.

### 3. Apply the optional search-acceleration loop

For the S-greater-than-15 case only, the routine repeats an acceleration step while the search-control value exceeds 3.

Each step, in order:

1. Halves the current search displacement using an unsigned right shift.
2. Decrements the search-control value by one.
3. Reads a 16-bit candidate boundary at the current start plus the halved displacement.
4. Advances the current start to that candidate address only if the candidate boundary is **strictly less than** the query.

A candidate equal to the query does not advance the current start.

The displacement is halved cumulatively. It is not recomputed from the original displacement, and the candidate address is relative to the potentially updated current start.

The loop is controlled by the 16-bit value originally read from offset `0x0A`, after promotion to unsigned 32-bit arithmetic. If that value is C, the number of acceleration iterations is zero for C at most 3 and C − 3 otherwise. The routine does not stop early when the displacement becomes zero.

### 4. Scan forward for the first qualifying boundary

Starting at the address selected above, the routine reads successive unsigned 16-bit boundary values at two-byte intervals.

It stops at the first boundary whose value is greater than or equal to the full 32-bit query.

The scan is entered at least once. It has no explicit iteration bound and performs no allocation-boundary check.

Let **E** denote the address of the boundary that stops the scan. The scan position after that read is E plus two bytes.

### 5. Read the corresponding lower boundary

The routine reads a 16-bit lower boundary at:

**E + 2 + S**

Call its zero-extended value **L**.

If L exceeds the query, the routine returns zero without reading the selected record’s delta or indirect-offset field.

Otherwise, it continues.

The boundary found by the scan and L play different roles: the scan accepts a boundary at least as large as the query, while this later check requires L to be no larger than the query. The routine does not separately verify that these fields form a consistent range.

### 6. Read mapping fields in their observable order

For an accepted lower boundary, the mapping fields are located at:

- **Delta:** E + 2 + 2S
- **Indirect-offset field:** E + 2 + 3S

Both are unsigned 16-bit values.

The routine reads the **indirect-offset field first**, then the delta. Call the indirect-offset field’s address **R**, its value **O**, and the delta **D**.

This order matters where memory is observable or mutable. It also means that both fields are read before the routine chooses between direct and indirect mapping.

### 7. Choose direct or indirect mapping

The routine uses direct mapping if either:

- O is zero; or
- the query is exactly `0x0000FFFF`.

For direct mapping, the result is the low 16 bits of the query plus D.

The exact `0x0000FFFF` exception is established by a 32-bit subtraction-and-comparison condition. It does not apply to every query whose low 16 bits are `0xFFFF`.

For all other queries with nonzero O, the routine uses indirect mapping:

1. Compute the low 16 bits of the query minus L.
2. Multiply that value by two to obtain a byte displacement.
3. Read a 16-bit entry at R plus O plus that byte displacement.
4. If the entry is zero, return zero.
5. Otherwise, add D to the entry and return the low 16 bits of that sum.

The zero-entry rule precedes delta addition. A zero indirect entry does not become D.

### 8. Normalize the return width

Every normally returning path yields a result masked to 16 bits, carried in the unsigned 32-bit return value.

Both direct and nonzero indirect additions therefore wrap modulo 65,536 in the returned result.

## Side effects

### Established local effects

F-af7e61f00d1b explicitly performs:

- One call to F-933d555a9096.
- Reads of the object and table as described above.
- No explicit stores to the object, table, or other memory.
- No explicit allocation, deallocation, or ownership transfer.

The absence of explicit writes does not make the overall invocation side-effect-free: dependency effects and platform-specific read effects remain possible.

### Memory validity requirements

To obtain a defined normal return, the dependency must return normally, and every address actually read afterward must be readable under the target machine’s rules.

The routine does not validate:

- The object reference.
- The table pointer.
- Table allocation bounds.
- Alignment of computed 16-bit read addresses.
- The consistency of S or acceleration metadata.
- The presence of a scan boundary sufficient to terminate the search.
- The range of an indirect entry address.

Odd values of S or other byte displacements can produce unaligned 16-bit reads. Whether those reads work, fault, or have other platform-specific behavior is documented_only.

The routine neither establishes nor transfers ownership of the object or table. Whether F-933d555a9096 changes ownership or lifetime is unknown.

### Query-range consequences

For queries from zero through `0xFFFF`, a suitably placed boundary can terminate the scan.

For queries above `0xFFFF`, no zero-extended 16-bit boundary can satisfy the stopping comparison. Thus the supplied logic cannot normally reach mapping for such a query through a finite sequence of valid boundary values. It continues scanning until some external event, invalid access, or platform-specific condition intervenes.

There is no explicit “query out of range” return path.

## Dependencies

### F-933d555a9096

**Invocation:** The original object argument and a second argument of zero.

**Established use of its result:**

- Zero causes immediate return of zero.
- Nonzero permits table processing.
- No other property of the returned value affects the lookup.

**Unknown contract:**

- Meaning of the second argument.
- Meaning of zero beyond its observed gating role.
- Whether the dependency initializes, validates, replaces, or otherwise changes the table.
- Reads or writes to globals or object fields.
- Allocation, release, ownership, and lifetime effects.
- Failure mechanisms other than a normal zero return.

The lookup must not be specified as using a pre-call snapshot of the object or as preserving global state across this dependency.

## Acceptance examples

These examples are conditional behavioral checks, not executed tests. Unless stated otherwise, assume the dependency returns nonzero, memory is valid and stable, address arithmetic is representable in the target environment, and the search reaches the stated record.

### Dependency rejection

If F-933d555a9096 returns zero, the result is zero. F-af7e61f00d1b performs no subsequent table-pointer read, even if the object’s table pointer would be invalid.

### Small-layout path

If S is 12, the routine begins its forward scan at B without reading the three acceleration fields or the pivot. Mapping-field locations still use byte increments of 12, not 24.

### Search equality distinctions

If S exceeds 15 and the initial pivot equals the query, the alternate start is selected.

If an acceleration-loop candidate equals the query, that candidate does not advance the current start.

If a forward-scan boundary equals the query, the scan stops at that boundary.

### Lower-bound rejection

If the scan stops successfully for query 100 but the associated L is 101, the result is zero. The delta and indirect-offset fields are not read on that path.

### Direct mapping and wraparound

If the query is 100, O is zero, and D is 7, the result is 107.

If the query is `0xFFFE`, O is zero, and D is 3, the result is 1.

### Exact maximum-16-bit exception

If the query is `0x0000FFFF`, the lower-bound check passes, and D is 2, the result is 1 even when O is nonzero. The indirect-offset field and delta are still read, but no indirect entry is read.

A query of `0x0001FFFF` does not qualify for that exception. More importantly, it cannot satisfy the earlier 16-bit scan-boundary comparison.

### Indirect indexing and zero handling

Suppose the query is 100, L is 96, O is 20, and the query is not the direct-mapping exception. The indirect entry is read 28 bytes after R: 20 bytes from O plus 8 bytes for four 16-bit entries.

If that entry is zero, the result is zero regardless of D.

If that entry is `0xFFFE` and D is 5, the result is 3.

### Acceleration control with zero displacement

If the search-control value is 5 and the initial displacement is zero, two acceleration iterations still occur. Each reads at the current start; reaching a zero displacement does not terminate the loop.

## Uncertainties

### Inferred intent

The paired boundary checks, repeated S-byte spacing, additive delta, and optional indirect array suggest a segmented range mapping. The initial metadata appears intended to accelerate selection of a boundary region.

Neither the semantic domain of the query nor a standardized table format is established. Those interpretations are not acceptance requirements.

### documented_only

The following remain unspecified by the supplied evidence:

- Target byte order, precise ABI, and alignment behavior.
- Valid object and table allocation sizes.
- Required relationships among S, search displacements, and region sizes.
- Required boundary ordering and sentinel placement.
- Whether acceleration metadata always preserves correct range selection.
- Address-overflow and invalid-access behavior.
- Concurrency, synchronization, and behavior under changing memory.
- Dependency state changes and memory ownership.
- Semantic meaning of a returned zero or any nonzero result.

No validation, fallback, memory-preservation guarantee, or ownership guarantee should be added to fill these gaps.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping.

Established observations include the dependency gate, object-field offset, 16-bit table reads, byte-based displacement arithmetic, conditional acceleration, full-width query comparisons, mapping-field read order, exact `0xFFFF` exception, indirect zero-entry rule, and final 16-bit return mask.

The lookup-domain interpretation and acceleration intent are inferences, separated above from the required observable behavior.

No runtime tests have been performed. The acceptance examples describe conditional consequences of the supplied representation, not observed executions.
