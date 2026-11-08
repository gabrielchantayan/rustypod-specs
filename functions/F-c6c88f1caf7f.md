<!-- Trusted provenance: F-c6c88f1caf7f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c6c88f1caf7f

## Purpose

F-c6c88f1caf7f validates a byte-encoded header followed by fixed-size records. Its visible checks concern:

- Availability of the fixed header relative to a boundary held in a context.
- Whether the declared object extent fits within that boundary.
- Whether the declared length can contain the stated number of records.
- Ordering and non-overlap of record ranges.
- An optional upper bound on a value derived from each record.

The function returns a 32-bit zero when execution reaches its normal return. It reports detected violations by calling F-7b0b9523800e; it does not return a distinct failure value itself.

The interpretation of the records as ranges is supported by the comparisons performed. The application-level meaning of their fields, and whether their endpoints represent addresses, character values, indices, or another domain, is unknown.

## Interface

### Inputs

The visible interface has four arguments:

| Argument | Representation | Visible use |
|---|---|---|
| Object address | 32-bit address-like value | Locates the header and record array. |
| Context address | 32-bit address-like value | Locates boundary and validation-control fields; passed to the dependency. |
| Auxiliary value A | Opaque 32-bit value | Forwarded on the initial short-header diagnostic path. |
| Auxiliary value B | Opaque 32-bit value | Forwarded twice on the initial short-header diagnostic path. |

The visible address arithmetic and unsigned comparisons operate at 32-bit width. Arithmetic that exceeds that width is interpreted modulo \(2^{32}\); it must not be silently replaced with unbounded arithmetic.

### Object layout

All offsets are byte offsets from the object address.

| Offset | Size | Interpretation |
|---|---:|---|
| 0–3 | 4 bytes | Not read by this function. |
| 4–7 | 4 bytes | Declared object length, decoded as an unsigned big-endian 32-bit value. |
| 8–11 | 4 bytes | Not read by this function. |
| 12–15 | 4 bytes | Record count, decoded as an unsigned big-endian 32-bit value. |
| 16 onward | 12 bytes per record | Consecutive records. |

Each record contains three unsigned big-endian 32-bit fields:

| Record-relative offset | Size | Contract name |
|---|---:|---|
| 0–3 | 4 bytes | Range start |
| 4–7 | 4 bytes | Range end |
| 8–11 | 4 bytes | Mapping base |

These names describe their role in the checks, not an established external format.

### Context layout

| Offset | Size | Visible meaning |
|---|---:|---|
| 4 | 32 bits | Boundary used by header and declared-extent checks. |
| 8 | 1 byte | Optional-check enable flag; any nonzero value enables the check. |
| 400 | 32 bits | Exclusive upper bound for the optional derived-value check. |

The 32-bit context fields are read as native words. Their byte order is not established independently by the supplied material. Unlike the object fields, they are not explicitly reconstructed from big-endian bytes.

The context boundary is compared against address-like sums. The source does not establish whether it is formally a one-past-end pointer, an address limit, or some other equivalent representation.

### Output and preconditions

Normal completion returns the 32-bit value zero.

There is no visible output buffer, allocation request, ownership transfer, or successful-result object. The function itself performs no explicit memory writes.

For a fault-free ordinary execution, the context fields and every object byte actually accessed must be readable. The visible checks are not a complete memory-safety guarantee: address calculations can wrap, the source provides no lower-bound check, and continued execution after a diagnostic depends on the dependency.

## Behavior

### 1. Fixed-header boundary check

Before reading either header field, the function reads the 32-bit context boundary at offset 4. It compares that boundary, unsigned, against the object address plus 16, with the addition evaluated at 32-bit width.

If the boundary is strictly below that computed address, it calls F-7b0b9523800e with these visible argument values, in order:

1. Context address.
2. The value 8.
3. Auxiliary value A.
4. Auxiliary value B.
5. Auxiliary value B again.

Equality does not trigger this call.

No explicit local return follows the call. If the dependency returns normally, the function proceeds to read the header. Consequently, the fixed-header check must not be described as an unconditional guard against subsequent invalid reads.

### 2. Header decoding

The function decodes the declared length from offsets 4–7, then the record count from offsets 12–15. Each value is assembled from four bytes in big-endian order.

These decoded values are retained locally for the subsequent checks. Later changes to the corresponding header bytes do not cause the function to decode them again.

### 3. Declared-extent and record-capacity checks

The function compares the context boundary with the 32-bit sum of the object address and declared length.

If the computed declared end is strictly greater than the boundary, it calls F-7b0b9523800e with the context address and the value 8.

Otherwise, it computes the required length as:

\[
(12 \times \text{record count} + 16) \bmod 2^{32}
\]

It calls the same dependency with the context address and the value 8 if the declared length is strictly less than that required length.

For stable context memory, the combined diagnostic condition is therefore:

- The declared end exceeds the boundary; or
- The declared end does not exceed the boundary, but the declared length is below the wrapped required length.

The two reasons share one diagnostic call site. A declared end equal to the boundary passes the extent comparison. A declared length equal to the computed required length passes the capacity comparison. Extra declared length is allowed; exact equality to the record-array size is not required.

The visible comparison logic references the context boundary separately for its ordering and equality decisions. It must not be treated as an established single immutable snapshot when concurrent mutation or other asynchronous changes are possible.

As with the first diagnostic, there is no explicit local exit after this call. If the dependency returns, record processing follows.

### 4. Record traversal

The first record begins at object offset 16. Records are visited in increasing index order, advancing the record address by 12 bytes per iteration. Exactly the decoded record count controls the loop, subject to faults or dependency control transfers.

For each record, the function reads and decodes all three fields before performing that record’s validation calls. The field groups are read in this order:

1. Range start.
2. Range end.
3. Mapping base.

The byte assembly establishes big-endian values. The supplied evidence does not independently establish a machine-level temporal ordering for every byte load within an expression.

No additional per-record comparison against the context boundary is performed.

### 5. Per-record range checks

The first record check rejects a range whose end is less than its start, using unsigned comparison. On this condition, the function calls F-7b0b9523800e with the context address as the only established argument.

For every record after the first, the function also compares its start with the preceding record’s end. If the current start is less than or equal to that preceding end, it calls F-7b0b9523800e with the context address as the only established argument.

Thus, uninterrupted diagnostic-free traversal requires:

- Every range end to be greater than or equal to its own start.
- Every later range start to be strictly greater than the immediately preceding range end.

Zero-span ranges, where start equals end, pass the first check. A later record beginning exactly at the preceding end fails the second check.

If diagnostic calls return, both range diagnostics can occur for one record. The reversed-range check occurs before the cross-record ordering check.

The preceding-end state is a local value, initially irrelevant for the first record. At the end of an iteration it becomes the end decoded for that record, including when an earlier check reported a violation but returned.

### 6. Optional derived-value check

After the range checks and any diagnostic calls they cause, the function reads the byte at context offset 8.

If that byte is zero, it skips both the offset-400 bound read and the optional comparison for that record.

If the byte is nonzero, it reads the 32-bit bound at context offset 400 and forms:

\[
\text{derived value}
=
(\text{mapping base} + \text{range end} - \text{range start})
\bmod 2^{32}
\]

If the bound is less than or equal to this derived value, it calls F-7b0b9523800e with the context address and the value 16.

Passing this check therefore requires the derived value to be strictly less than the bound. Equality fails.

The expression contains no added unit for an inclusive range length. The function checks the mapping base plus the difference between end and start, not mapping base plus a record element count.

The enable flag is reread on every record. The bound is likewise reread for every record on which the flag is enabled. A diagnostic dependency that changes these context fields can therefore influence the remaining checks, including the optional check on the current record.

### 7. Completion

After the final record, the function returns zero if execution remains local and reaches the return.

A zero record count skips all record reads, range checks, enable-flag reads, and optional-bound reads. Header-related checks still occur.

## Side effects

The function’s direct observable effects are memory reads and calls to F-7b0b9523800e. No direct store to the object or context is visible.

The following distinctions are important:

- There is no visible allocation or deallocation by this function.
- No object or context ownership transfer is established.
- The dependency may have side effects or may not return; neither behavior is specified here.
- If the dependency returns, the function generally continues rather than converting the diagnostic into a local failure return.
- Record fields already decoded remain local values even if a dependency changes their backing bytes.
- Future records are read later and are not protected by a whole-array snapshot.
- Context values used after a dependency call are read at their documented points, not assumed preserved.

A normal zero return alone is not evidence that no diagnostic occurred.

## Dependencies

### F-7b0b9523800e

This is the only visible external dependency.

| Trigger | Established visible arguments |
|---|---|
| Fixed-header boundary failure | Context, 8, auxiliary A, auxiliary B, auxiliary B |
| Declared extent or record-capacity failure | Context, 8 |
| Range end below range start | Context |
| Later start not strictly above preceding end | Context |
| Enabled derived-value bound failure | Context, 16 |

The values 8 and 16 are established argument values, not established application-level error categories.

The dependency’s formal signature, return convention, memory ownership behavior, global-state effects, and control-transfer behavior are unknown. Calls showing only the context do not establish a particular diagnostic-code argument.

No dependency-produced value contributes a defined result to a later comparison. In particular, the optional check uses the context bound read after the range diagnostics rather than a diagnostic return value.

## Acceptance examples

These examples are conditional static expectations, not runtime observations. Unless stated otherwise, addresses do not wrap, memory is readable and stable, and no dependency call occurs except those identified.

1. **Empty object.**  
   A record count of zero, declared length of 16, and boundary equal to the object address plus 16 produce no diagnostic. The function returns zero without reading the enable flag or optional bound.

2. **Unused trailing bytes.**  
   One record requires 28 bytes. A declared length of 40 can pass when the boundary covers all 40 bytes and the record checks pass. The additional 12 bytes are not inspected as records.

3. **Declared extent exactly at the boundary.**  
   A declared end equal to the context boundary passes the extent check. Other checks remain independently applicable.

4. **Declared record storage too short.**  
   With one record and declared length 27, where the declared end otherwise fits the boundary, the capacity check calls F-7b0b9523800e with context and 8. If it returns, the function still attempts to read the 12-byte record.

5. **Zero-span range.**  
   A first record with start 7 and end 7 passes the range-shape check. With the optional check enabled, mapping base 3 and bound 4 also pass because the derived value is 3.

6. **Adjacent endpoint equality fails.**  
   If one record ends at 20 and the next starts at 20, the next record triggers the ordering diagnostic. A next start of 21 passes that comparison.

7. **Optional bound is exclusive.**  
   For start 10, end 12, and mapping base 5, the derived value is 7. Bound 8 passes; bound 7 triggers the context-and-16 diagnostic.

8. **Disabled optional check.**  
   A zero enable byte skips the optional check regardless of mapping base. The offset-400 bound need not be read on that iteration.

9. **Wrapped derived arithmetic.**  
   For start 0, end 1, and mapping base \(2^{32}-1\), the derived value wraps to zero. With the check enabled and bound 1, this comparison passes; there is no separate overflow diagnostic.

10. **Wrapped record-capacity arithmetic.**  
    A record count of 357,913,940 makes \(12 \times \text{count} + 16\) wrap to zero. Consequently, this capacity comparison alone does not reject a declared length of 16. Record traversal still attempts the stated count; this is not evidence that such an input is safe.

11. **Multiple diagnostics with a returning dependency.**  
    Suppose the previous end is 10 and the next record has start 8 and end 7. If the dependency returns without changing relevant state, the function reports the reversed range first and the ordering violation second. It then evaluates the current enable flag.

## Uncertainties

The following are **documented_only**:

- The object’s external format name and the meaning of its unread header bytes.
- The semantic units of the three record fields.
- Whether the context boundary is formally an exclusive memory endpoint.
- Native byte order and alignment requirements for the context word reads.
- Whether the dependency returns normally, throws, terminates, or transfers control elsewhere.
- The meaning of diagnostic values 8 and 16.
- Any dependency effects on memory, globals, allocation, or ownership.
- Whether callers guarantee nonwrapping addresses, sufficient backing storage, valid record counts, or synchronized access.
- Whether diagnostics are intended to stop execution despite the visible local continuation.

An inferred intent is validation of ordered, disjoint ranges with an optionally bounded mapping. That interpretation does not justify strengthening the actual checks, adding overflow rejection, or assuming a nonreturning error handler.

## Evidence

The established behavior comes solely from the supplied static function representation and its dependency-ID mapping:

- Explicit byte reconstruction establishes big-endian object fields.
- Unsigned operations establish the range comparisons and 32-bit arithmetic behavior.
- Visible offsets establish the header, record, and context accesses.
- Branch placement establishes diagnostic ordering and conditional context reads.
- The absence of explicit stores establishes that this function has no direct memory-write path in the supplied representation.
- The final return establishes zero on normal completion, not a success/failure result protocol.

No runtime tests have been performed. Machine-level access ordering, dependency implementation behavior, and external format semantics remain unverified.
