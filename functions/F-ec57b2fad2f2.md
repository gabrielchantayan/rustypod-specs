<!-- Trusted provenance: F-ec57b2fad2f2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ec57b2fad2f2

## Purpose

F-ec57b2fad2f2 scans a sentinel-terminated sequence of fixed-stride records and returns a two-word result through caller-provided storage.

A record qualifies only when:

- An initial comparison against the caller’s comparison argument returns zero.
- A dependent lookup produces an object whose 32-bit discriminator equals the requested discriminator.
- That object’s one-byte attribute equals the caller’s flags with bit 2 cleared.
- A second comparison against the same caller argument returns zero.

The first qualifying record ends the scan. Its associated object address and record payload become the result. If no record qualifies, both result components are zero before the flag-derived adjustment described below.

Independently of whether a match is found, input flag bit 2 controls bit 0 of the first output word. This is an OR operation, not a replacement of the address’s low bit.

This description establishes search and result-construction behavior. Interpreting the records as names, the object as a resource, or the result as an ownership-bearing handle would be inferred intent, not established behavior.

## Interface

### Inputs

The function has four inputs:

1. **Output storage address.** Points to storage for two consecutive 32-bit words. The first word begins at offset 0 and the second at offset 4.
2. **Comparison argument.** An uninterpreted 32-bit value forwarded unchanged as the second argument of both comparison calls. Its actual type and validity requirements depend on the comparison dependency.
3. **Requested discriminator.** An unsigned 32-bit value compared for exact equality with an object field.
4. **Flags.** An unsigned 32-bit value used in two distinct ways:
   - Mask `0x00000004` selects the flag that contributes a Boolean value to bit 0 of the first output word.
   - Mask `0xFFFFFFFB` clears that flag before comparison with an unsigned one-byte object attribute.

There is no explicit record-table input. The initial record address is obtained from a global pointer.

### Outputs

No explicit return value is established. The observable result is written to output storage:

| Offset | Width | Meaning |
|---|---:|---|
| 0 | 32 bits | Matching object address, or zero if no match, bitwise-ORed with 1 when input flag bit 2 is set |
| 4 | 32 bits | Payload from the matching record, or zero if no match |

The first word is written before the second.

A matching object’s address is used as a 32-bit value. No alignment guarantee is established, so the first word must not be described unconditionally as an address with a safely reserved tag bit.

### Preconditions

Successful completion requires:

- Writable output storage covering both 32-bit words.
- A readable global table pointer and a readable traversal path through the records to a sentinel or a match.
- Valid memory for every field and comparison operand actually reached.
- Valid return values and applicable memory from the lookup dependency.
- Satisfied, but otherwise unknown, preconditions of the comparison and lookup dependencies.

The function contains no explicit checks for null output storage, missing termination, invalid record pointers, invalid lookup results, or insufficient memory.

The memory layouts below are the layouts used by the supplied static representation. Byte order, alignment requirements, and portability to another execution environment are not established.

## Behavior

### Record layout and traversal

Records are visited in ascending address order with a stride of 16 bytes.

Only these record locations have established direct uses:

| Record-relative offset | Width | Use |
|---|---:|---|
| 0 | 1 byte | Zero marks the end of the sequence |
| 4 | 32 bits | Address-like value used to form the first comparison operand |
| 8 | 32 bits | Payload returned for a qualifying record |

The remaining record bytes have no established direct role in this function. Dependencies may inspect additional bytes.

The global table pointer is sampled at entry. The function does not visibly reload that global while traversing. A dependency changing the global pointer therefore does not itself redirect the current scan. Changes to record contents or other referenced memory may still affect later reads.

At the start of each record examination, the candidate object value is reset to zero. The first direct record read is the byte at offset 0.

If that byte is zero:

- No comparison or lookup is performed for that record.
- The scan ends with a zero candidate object and a zero payload.
- The function performs the two output writes.

There is no explicit record count or traversal limit. Without a reachable sentinel or qualifying record, successful termination is not established.

### Initial comparison

For a non-sentinel record, the function reads the 32-bit field at offset 4 and adds 2 to obtain the first argument to the comparison dependency. The second argument is the caller’s comparison argument.

The source represents the offset-4 field as signed 32-bit data. The address calculation uses an increment of two, not an increment of two machine words. Exact overflow behavior for values near the signed limit is not established by this representation.

Only a zero comparison result permits further evaluation of the record.

A nonzero result causes traversal to continue at the next 16-byte record. No lookup, object-field read, or payload read is performed by this function on that path.

Although the dependency appears comparison-like, zero is the only established acceptance condition. Case handling, termination rules, encoding, and the meaning of nonzero return values are unknown.

### Object lookup and qualification

After a zero initial comparison result, the function calls F-4506e248f6d1 with:

- The current record address.
- A second argument of zero.

Its result is treated as a 32-bit object address. There is no explicit null check or error-result check before object memory is accessed.

The directly inspected object layout is:

| Object-relative offset | Width | Use |
|---|---:|---|
| 6 | Not established | Address passed to the second comparison |
| `0x48` | 32 bits | Discriminator |
| `0x4C` | 1 byte | Unsigned attribute |

The qualification sequence is ordered:

1. Read the object’s 32-bit discriminator at offset `0x48`.
2. Compare it with the requested discriminator.
3. Only if those values are equal, read the unsigned byte at offset `0x4C`.
4. Compare the zero-extended byte with the full 32-bit flags value after clearing bit 2.
5. Only if that comparison succeeds, invoke the comparison dependency with the object address plus 6 and the original comparison argument.
6. Accept the record only if this second comparison returns zero.

The attribute comparison is not a low-byte-only comparison. The masked flags remain 32 bits wide. Consequently, any set flag bit above bit 7 prevents this comparison from succeeding: clearing bit 2 cannot reduce such a value to the byte’s range of 0 through 255.

The byte is read as unsigned. Values from `0x80` through `0xFF` therefore compare as 128 through 255, not as negative values.

If any qualification condition fails, traversal advances by 16 bytes. A rejected lookup result does not become the eventual no-match result, because the candidate value is reset before the next record’s sentinel check.

### Successful selection and output construction

When the second comparison returns zero, the function reads the current record’s 32-bit payload at offset 8 and stops scanning.

This payload read occurs after both dependency comparisons and the lookup. If those dependencies mutate the record payload, the value read at this point is the value used. The function does not visibly revalidate the object discriminator or attribute after the second comparison.

The first output word is constructed from the selected object address, or zero on the sentinel path:

- If flag bit 2 is clear, the candidate value is written unchanged.
- If flag bit 2 is set, the candidate value is bitwise-ORed with 1.

Other flag bits do not directly contribute bits to the output word.

The second output word is then written with the selected payload, or zero on the sentinel path.

No output initialization is performed before traversal. If an earlier read or dependency call fails to complete normally, this function has not yet performed either explicit output write.

## Side effects

### Direct memory effects

The established direct memory effects are:

- Reading the initial global table pointer.
- Reading record sentinel bytes.
- Conditionally reading record address fields.
- Conditionally reading object discriminator and attribute fields.
- Reading one record payload on successful selection.
- Writing the first output word.
- Writing the second output word.

There are no explicit global writes, allocations, releases, or object-field writes in this function.

That does not establish that the overall operation is pure except for the output buffer. Dependencies may have additional effects.

### Dependency effects and sequencing

Calls execute sequentially. Their visible order for a record reaching all checks is:

1. Initial comparison.
2. F-4506e248f6d1.
3. Second comparison.

The function provides no rollback for dependency side effects when a candidate is later rejected.

No assumption is made that dependencies preserve globals, leave record or object memory unchanged, or avoid modifying output storage through other reachable references.

### Ownership and aliasing

No allocation, ownership transfer, reference-count change, or release is explicitly established. The returned object value is used as an address and then encoded into an output word. Whether the caller may retain, dereference, or release it is unknown.

Output storage is not checked for overlap with records or objects. All explicit search reads finish before the first explicit output write, but overlap may still overwrite shared data when the result is stored. Such overlap is not established as supported usage.

### Errors and partial completion

There is no explicit error-status output or exception-handling path.

Invalid memory or unmet dependency preconditions may prevent normal completion; their precise manifestation is outside the supplied evidence. If the first output write succeeds and the second cannot complete, a partially updated result is possible.

## Dependencies

### F-4506e248f6d1

This supplied opaque dependency ID identifies the lookup invoked after a successful initial comparison.

Established use:

- Receives the current record address and zero.
- Returns a value used as a 32-bit object address.
- Is called at most once per visited non-sentinel record.

Unknowns include validation, failure signaling, allocation, ownership, lifetime, and side effects. The caller does not visibly validate its result before reading the discriminator field.

### F-unknown-1

This is a document-local opaque ID for the comparison dependency; no supplied opaque mapping identifies it. It does not assert a published dependency identity.

Established use:

- Receives two arguments.
- Returns an integer-like result for which zero permits progress.
- Is invoked first with the record-derived address plus 2.
- May be invoked again with the object address plus 6.
- Receives the same original comparison argument in both calls.

Its implementation, operand formats, memory effects, and error behavior are unknown.

## Acceptance examples

These are conditional consequences of the static behavior, not executed tests. Unless stated otherwise, all referenced memory is valid and dependencies return normally.

### Immediate sentinel

Given a first record whose offset-0 byte is zero:

- With flags `0x00000000`, output is `(0, 0)`.
- With flags `0x00000004`, output is `(1, 0)`.
- Neither dependency is called.

Other flag bits do not alter these sentinel results.

### Successful match with output tagging

Suppose the first qualifying record yields object address `0x00001000`, discriminator 7, attribute 1, and payload `0x12345678`. Both comparisons return zero.

For requested discriminator 7 and flags `0x00000005`:

- Clearing bit 2 leaves 1, so the attribute matches.
- Output is `(0x00001001, 0x12345678)`.

For flags `0x00000001`, the same candidate qualifies but output is `(0x00001000, 0x12345678)`.

### Attribute comparison retains high flag bits

Suppose a candidate has attribute 1 and the requested discriminator matches.

For flags `0x00000105`, clearing bit 2 leaves `0x00000101`, not 1. The candidate is rejected, and the second comparison is not invoked for it.

### Discriminator rejection short-circuits later work

If the lookup returns an object whose discriminator differs from the requested discriminator, this function does not read its attribute byte and does not invoke the second comparison. It advances to the next record.

### First qualifying record wins

If two records would independently satisfy every check, only the earlier record is selected. The later record is not examined after the first succeeds.

### Rejected lookup does not leak into no-match output

Suppose one record reaches lookup but fails its attribute check, and the next record is the sentinel.

The output is `(0, 0)` when flag bit 2 is clear, or `(1, 0)` when it is set. The rejected object address is not returned.

### Low address bit is not cleared

Suppose a qualifying object address is already odd, such as `0x00001001`.

Setting flag bit 2 leaves that value unchanged. Clearing flag bit 2 also leaves it unchanged. The function cannot establish from this value alone whether bit 0 originated in the address or the flag.

## Uncertainties

The following are documented_only:

- **Comparison semantics:** Zero is accepted; string semantics and operand validity rules are unknown.
- **Dependency identity:** F-unknown-1 is a local label, not a supplied mapping.
- **Table extent and synchronization:** No count, concurrency protocol, or mutation guarantee is provided.
- **Arithmetic boundaries:** Intended address values are apparent, but signed overflow and address wraparound behavior are not independently established.
- **Object validity and lifetime:** The lookup result is immediately dereferenced without visible validation.
- **Ownership:** No acquisition, transfer, or release contract can be established.
- **Alignment and tagging:** Bit 0 is ORed into the result, but address alignment is not proved.
- **Abnormal completion:** Faults, exceptions, and dependency-specific errors have no explicit handling in the supplied function.
- **Broader purpose:** The data may represent named objects with attributes, but that interpretation is inferred intent only.

## Evidence

Established static evidence consists of the supplied function representation and the supplied mapping for F-4506e248f6d1.

The contract follows its explicit record stride, field offsets, access widths, conditional evaluation order, dependency arguments, masks, and ordered output writes. It does not establish the implementations of dependencies or the contents of the global table.

No runtime tests have been performed. Acceptance examples are conditional static predictions, not observed executions.
