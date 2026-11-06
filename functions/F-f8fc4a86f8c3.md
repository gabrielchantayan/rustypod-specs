<!-- Trusted provenance: F-f8fc4a86f8c3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f8fc4a86f8c3

## Purpose

F-f8fc4a86f8c3 attempts to produce a two-field extent description from a caller-supplied memory region and an object providing an indirect query operation.

Its primary path examines four fixed-position records in order. A qualifying record supplies a nonzero 32-bit starting value and a nonzero 32-bit length. The length must not exceed the unsigned result of the object’s query operation.

If no record qualifies, the function performs up to two calls to F-fca51b715561. A zero result from either call selects a fallback extent: starting value zero and length equal to a fresh query result.

A 16-bit marker gates both paths. If the marker does not match, or neither path succeeds, the function returns zero.

The layout and accepted type values suggest selection of a partition-like extent. That interpretation is inferred intent, not an established requirement. The supplied evidence does not establish units, a storage format name, or the meaning of the queried quantity.

## Interface

The interface has four argument roles:

| Argument role | Representation in the supplied evidence | Use |
|---|---|---|
| Query object | Pointer to an object whose first field supplies a dispatch-table address | Receiver of an indirect operation at dispatch-table byte offset 0x28 |
| Region address | 32-bit integer used as a memory address | Base for record access, fallback input addresses, and marker-address calculation |
| Starting-value destination | Pointer to a 32-bit integer | Receives the selected starting value, or zero on fallback |
| Length destination | Pointer to an unsigned 32-bit integer | Receives the selected length or fallback query result |

The return value is 32 bits:

- **1:** the function completed one of its success paths and issued both output stores.
- **0:** the function did not select an extent.

The function has no separate error-code output.

### Input and output widths

The marker comparison uses a 16-bit value. Its required signed value is −21,931, corresponding to the 16-bit pattern 0xAA55.

Record status and type fields are one byte each. The accepted status values have byte patterns 0x00 and 0x80.

The record starting value is loaded as a 32-bit integer. Only equality with zero is tested; a value with its high bit set is not rejected for being negative under signed interpretation. Its 32-bit value is copied to the starting-value destination.

The record length and query result are treated as unsigned 32-bit quantities. Their comparison is unsigned, including for values at or above 0x80000000.

### Preconditions

For defined operation, the caller must provide all memory and call targets actually reached by the chosen execution path:

- Readable memory for the marker access.
- Readable record fields for every record inspected.
- A valid query object, dispatch-table entry, and callable operation whenever a query is reached.
- Writable four-byte output locations whenever success is reached.
- Valid input memory under F-fca51b715561’s contract whenever fallback checking is reached.

These requirements are conditional. For example, a marker mismatch does not require usable output destinations or a usable query object, because the function does not explicitly access them on that path.

The evidence does not establish an enforced non-aliasing rule. Aliasing can affect results because output stores and dependency calls have a defined logical order. No blanket alias-safety guarantee is supported.

## Behavior

### 1. Gate all selection work on the marker

The function reads a 16-bit marker at an address formed by adding the region address to a separately supplied global value.

The value of that global is not included in the supplied evidence. Consequently, the marker cannot be assigned a concrete offset within the region.

If the marker is not 0xAA55, the function returns zero immediately. It does not explicitly inspect records, invoke either dependency operation, or write the output destinations.

If the marker matches, processing continues to the four-record scan.

### 2. Examine four records in increasing index order

Record indices are zero through three. Relative to the region address, the fields for record index \(i\) are:

| Field | Byte offset | Width |
|---|---:|---:|
| Status | 0x1BE + 0x10 × \(i\) | 1 byte |
| Type | 0x1C2 + 0x10 × \(i\) | 1 byte |
| Starting value | 0x1C6 + 0x10 × \(i\) | 4 bytes |
| Length | 0x1CA + 0x10 × \(i\) | 4 bytes |

The table defines accesses, not a claim that the entire surrounding region is validated or read.

For each record, the function applies these conditions in order:

1. **Status eligibility.** Only status byte patterns 0x00 and 0x80 permit further inspection of that record. Any other status skips directly to the next record.
2. **Type eligibility.** Only types 0x01, 0x04, 0x06, 0x0B, 0x0C, and 0x0E permit candidate-field evaluation. Other types skip the record.
3. **Nonzero fields.** The starting value and length are both loaded. Both must be nonzero.
4. **Query bound.** For a candidate satisfying the preceding conditions, the object query is invoked. The candidate qualifies if its unsigned length is less than or equal to the unsigned query result.

A candidate with a zero starting value or zero length does not cause a query call.

A candidate whose length exceeds the query result is rejected, and scanning continues. The next eligible candidate causes a new query call. The function does not cache one query result for the complete scan.

### 3. Return the first qualifying record

On the first qualifying record:

1. The already-loaded starting value is written to the starting-value destination.
2. The already-loaded length is written to the length destination.
3. The function returns one.

Later records are not inspected, and fallback checking is not performed.

The candidate fields are loaded before the query call. If the query modifies those fields, this invocation still uses the previously loaded values for its comparison and successful output stores.

The function does not check:

- Whether the starting value is below the query result.
- Whether starting value plus length fits within the queried quantity.
- Whether that sum would overflow 32 bits.
- Whether either field satisfies alignment or unit constraints.
- Whether another record would be a better match.

Thus the established qualification rule is a bound on **length alone**, not on the end of an extent.

### 4. Check fallback eligibility after all four records fail

If none of the four records succeeds, the function calls F-fca51b715561 with:

- First input address: region address plus 0x34.
- Second input address: a fixed global data address.
- Third argument: integer value three.

If this call returns zero, fallback is selected immediately.

If it returns nonzero, the function calls the same dependency again with:

- First input address: region address plus 0x2B.
- The same fixed global data address as the second input.
- Third argument: integer value three.

A zero result from this second call also selects fallback.

If both results are nonzero, the function returns zero without issuing either explicit output store.

Only the zero/nonzero distinction is used. The evidence does not establish that F-fca51b715561 is a comparison routine, that its inputs are strings, or that the value three means a byte count.

### 5. Produce the fallback outputs

Once fallback is selected:

1. The function writes zero to the starting-value destination.
2. It invokes the object query.
3. It writes that query result to the length destination.
4. It returns one.

Unlike record selection, fallback accepts a zero query result. It applies no additional bound or nonzero test.

The store of zero occurs **before** the query call. This matters if the query observes or modifies the starting-value destination, or if that destination aliases memory used to resolve the query.

The function does not restore the starting-value destination after the query. Consequently, although it explicitly writes zero there, a query-side modification can determine its value at return.

## Side effects

### Explicit writes

The function’s own explicit writes are limited to the two output locations on a success path.

Their order is always:

1. Starting-value destination.
2. Length destination.

On record success, both writes occur after the successful query. On fallback success, the query occurs between the writes.

If both destinations denote the same four-byte location, the length write is last and determines that location’s final value, provided intervening dependency behavior does not invalidate the access.

If the function returns zero, it has issued no explicit output stores. This is not a guarantee that memory remains unchanged: query calls made while rejecting candidates, or fallback dependency calls, may have side effects.

### Reads and call ordering

The established logical sequence is marker gating, increasing-index record inspection, and then fallback checks only if necessary.

Within an eligible record, the starting value is loaded before the length, and both are loaded before a query is invoked. A failed status or type check avoids the later fields for that record.

The status expression permits a second status evaluation when the first evaluation is nonzero. The supplied representation does not establish the exact number of physical machine reads, volatile semantics, or behavior under concurrent modification.

Each query invocation obtains its target through the query object and dispatch-table slot. There is no established guarantee that a previous dependency call preserves either the object or its dispatch table.

### Ownership and lifetime

The function contains no explicit allocation, deallocation, ownership transfer, or reference-count operation.

This does not establish that its dependencies allocate nothing, release nothing, or preserve referenced memory. Their ownership and lifetime behavior is undocumented.

The caller must keep memory valid for later accesses even across dependency calls. The function provides no recovery path if a dependency invalidates memory subsequently needed by this invocation.

## Dependencies

### F-fca51b715561

This dependency is called only after a successful marker check and an unsuccessful four-record scan.

Established call behavior:

- At most two calls.
- The first uses the region-relative address 0x34.
- The second, if needed, uses the region-relative address 0x2B.
- Both receive the same fixed global data address and integer argument three.
- A zero result selects fallback.
- A nonzero result from the first call permits the second call.

Its actual processing, read footprint, write footprint, return width, ownership effects, and global-state effects are **documented_only** unknowns.

In particular, a first nonzero-returning call is not assumed to leave the second call’s inputs unchanged.

### Unidentified query dependency

The query operation is an indirect dependency with no opaque function ID supplied in the dependency map.

Its target is obtained from dispatch-table byte offset 0x28, and its receiver is the query object. The returned value is consumed as an unsigned 32-bit quantity.

Possible invocation counts are:

- Zero when the marker fails, or when no eligible candidate reaches the bound check and fallback is rejected.
- One per record reaching its bound check, up to four.
- One additional invocation if fallback is selected.

The maximum is therefore five query invocations: four rejected candidate checks followed by fallback.

No purity, stability, ownership, or global-preservation property is established. Different calls may return different values.

## Acceptance examples

These examples are conditional static expectations, not runtime test results. Unless stated otherwise, memory remains valid and dependencies do not modify relevant memory.

1. **Marker mismatch.**  
   If the marker differs from 0xAA55, expect return zero, no dependency calls, and no explicit output writes.

2. **First qualifying record wins.**  
   With a matching marker, an accepted status and type in record zero, starting value 100, length 40, and query result 40, expect outputs 100 and 40 and return one. Equality at the bound is accepted. No later record or fallback check is reached.

3. **Rejected status avoids further record processing.**  
   A record with status 0x01 is skipped regardless of what its type, starting value, and length would contain. It causes no query call.

4. **Zero candidate field avoids querying.**  
   An otherwise eligible record with starting value zero and length 25 is rejected without a query call. The same applies to nonzero starting value and zero length.

5. **Oversized candidate permits a later success.**  
   If one eligible record has length 80 and receives query result 79, it is rejected. A later record with length 30 may succeed against a fresh query result of 30. The first result is not reused.

6. **No end-position validation.**  
   With accepted status and type, starting value 0xFFFFFFF0, length 32, and query result 32, the candidate qualifies. There is no rejection based on signed starting-value interpretation or starting-value-plus-length overflow.

7. **First fallback check succeeds.**  
   If all records fail and the first F-fca51b715561 call returns zero, the second call is omitted. A fallback query result of 500 produces explicit writes of zero and 500, followed by return one.

8. **Fallback accepts zero length.**  
   If fallback is selected and the query returns zero, expect return one and explicit writes of zero to both destinations.

9. **Both fallback checks fail.**  
   If all records fail and both F-fca51b715561 calls return nonzero, expect return zero and no explicit output stores. Earlier dependencies may nevertheless have modified state.

10. **Aliased destinations.**  
    If both destinations identify the same valid four-byte location and a record succeeds with starting value 12 and length 7, the location ends with 7.

11. **Fallback query observes the first output store.**  
    If fallback is selected and the query can observe the starting-value destination, it observes the zero store before its invocation, subject to any other permitted modifications. The function does not defer that store until after querying.

## Uncertainties

The following are **documented_only** unknowns, not implementation promises:

- The global value used to locate the marker.
- The contents and extent of the fixed global data supplied to F-fca51b715561.
- The identity and complete contract of the indirect query operation.
- The semantics of F-fca51b715561 beyond the observed zero/nonzero branch use.
- The units and domain meaning of the starting value, length, and query result.
- Target byte order, alignment requirements, and machine behavior for misaligned accesses.
- Defined behavior for invalid pointers, inaccessible memory, address wraparound, dependency faults, or concurrent mutation.
- Any stronger aliasing restrictions imposed outside this function.
- Any dependency guarantees concerning memory ownership, global state, or object lifetime.

The evidence uses 32-bit address-bearing arithmetic, with record stride 16 and fixed byte offsets. It contains no bounds or overflow checks for those address calculations. That absence does not establish that wrapped addresses are valid or that overflow behavior is portable.

The partition-like interpretation is **inferred intent**. It must not add validation rules or named format semantics to the established contract.

## Evidence

This contract is based solely on the supplied static function representation and the dependency mapping identifying F-fca51b715561.

Established static evidence covers the marker gate, record offsets and widths, accepted status and type values, short-circuit call conditions, unsigned length comparison, output-store order, fallback call order, and return values.

No dependency implementations, global-data contents, target ABI documentation, or execution traces were supplied. No runtime tests have been performed.
