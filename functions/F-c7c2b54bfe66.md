<!-- Trusted provenance: F-c7c2b54bfe66; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c7c2b54bfe66

## Purpose

F-c7c2b54bfe66 processes a caller-supplied sequence of fixed-stride records through F-c2e9c7166e37. Each record supplies one 32-bit value, paired with a destination position derived from a caller-supplied base position.

When an optional mode is enabled, the function also inspects an object obtained through F-b05552b91301 after each record is processed. A qualifying object has its leading byte changed from 8 to 19. Qualification additionally requires that the object’s associated signed position lie within the half-open interval covered by the entire requested sequence.

The function returns the sequence’s initially captured signed count, not a success count. A null sequence descriptor returns zero without invoking either dependency.

These behaviors are established by the supplied static evidence. The semantic meanings of the records, destination positions, object tags, and enclosing context are not established.

## Interface

### Inputs

The interface has four arguments:

| Argument | Representation and role |
|---|---|
| Context | A 32-bit value used as an address and passed unchanged to F-c2e9c7166e37. Optional inspection also obtains a container address from its offset 12. |
| Sequence descriptor | A nullable address of a descriptor containing a signed 32-bit count at offset 0 and a record-table address at offset 12. |
| Base position | A signed 32-bit integer used to derive per-record positions and the optional inspection interval. |
| Inspection flag | A 32-bit integer. Zero disables optional inspection; every nonzero value enables it. |

The source representation uses 32-bit words and integer-held addresses. This contract describes that represented layout; it does not assert portability to a wider-pointer execution environment.

### Descriptor and record layout

For a nonnull descriptor, the relevant fields are:

| Byte offset | Width | Meaning |
|---|---:|---|
| 0 | 4 bytes | Signed record count |
| 12 | 4 bytes | Address of the first record |

Other descriptor bytes are not directly accessed.

Records have a stride of 12 bytes. Only the first four-byte word of each record is directly read by this function. That word is supplied as the second argument to F-c2e9c7166e37. Its semantic type and signedness are not established independently of the dependency.

The remaining eight bytes of each record are skipped by this function. Their existence in the stride does not establish that they are unused elsewhere.

### Context and inspected-object layout

When inspection is enabled and at least one record is processed:

- The context has a four-byte container-address field at byte offset 12.
- The container has a signed 32-bit field at byte offset 12. That field, reduced by one, supplies the second argument to F-b05552b91301.
- A nonnull object returned by F-b05552b91301 has a one-byte tag at offset 0.
- If its tag is 8, its signed 32-bit position at offset 4 may be read.
- If all qualification conditions hold, its tag byte must be writable.

Calling the container field a “count” or describing the lookup as retrieving a “last element” would be an interpretation, not an established interface guarantee.

### Output

The return value is a signed 32-bit integer:

- Zero for a null descriptor.
- Otherwise, the descriptor count captured on entry, including a negative count.

The return value does not report whether dependency calls succeeded or how many tag bytes changed.

## Behavior

### Descriptor capture and traversal

A null descriptor ends processing immediately. The context, base position, and inspection flag do not cause any direct memory access on this path.

For a nonnull descriptor, the record-table address is captured before the count. Both are captured before processing begins. Later changes to either descriptor field do not replace those captured values.

A positive captured count causes that many record-processing iterations, provided execution continues normally. A zero or negative count causes no iterations. In those cases, the captured count is returned unchanged, although the descriptor’s record-table field has still been read.

Records are visited in increasing address order, at offsets 0, 12, 24, and so on from the captured table address. The function does not copy the table in advance. Each record’s first word is read when that record is processed.

Consequently, a dependency that changes the contents of a later record can affect a later call. A dependency that changes only the descriptor’s table-address field does not redirect traversal.

### Required processing call

For each visited record, F-c2e9c7166e37 receives:

1. The original context value.
2. The current record’s first 32-bit word.
3. The base position plus the zero-based record ordinal.

These calls occur in record order. Their return values are ignored.

The optional inspection for a record occurs only after that record’s F-c2e9c7166e37 invocation has returned. The next record is not processed until the current record’s optional inspection is complete.

There is no explicit branch on a dependency status, no rollback, and no early termination based on a normal return value from F-c2e9c7166e37.

### Optional inspection

A zero inspection flag suppresses all direct inspection-related context accesses, all F-b05552b91301 calls, and all tag changes by this function.

For a nonzero inspection flag, every processed record is followed by one F-b05552b91301 call. The lookup arguments are obtained from context and container state at that time, after the preceding F-c2e9c7166e37 call.

The first lookup argument is the container address obtained through the context’s offset-12 field. The second is derived from the offset-12 signed field of a container obtained through that same context field, minus one.

The source contains separate evaluations of the context’s container field for these two arguments. Their evaluation order is not established. Stable memory makes them agree; the evidence does not justify a guarantee that they constitute one atomic snapshot.

Inspection uses the lookup’s returned address as follows:

- A null result causes no object access.
- A nonnull result has its leading byte examined.
- A leading byte other than 8 ends inspection of that object.
- For tag 8, the signed position at offset 4 must be at least the base position.
- Only if that lower-bound condition passes is the upper-bound condition evaluated.
- The position must also be strictly less than the base position plus the captured total count.
- If both bounds hold, the leading byte is changed to 19.

The interval is based on the whole requested sequence, not the current record’s position. An object associated with a later position in that interval can therefore qualify during an earlier iteration.

The position field appears in two separate conditional reads: one for the lower bound and, if necessary, one for the upper bound. The static evidence does not establish atomicity or a stable snapshot across those accesses.

### Integer and address boundaries

The count, base position, ordinal, inspected position, and container field used for the lookup index are represented as signed 32-bit integers.

Three arithmetic boundaries matter:

- The base position plus each processed ordinal.
- The base position plus the captured count, evaluated during qualifying inspection.
- The container’s offset-12 field minus one.

The supplied evidence does not establish overflow handling. This contract therefore does not promise wrapping, saturation, a checked error, or any particular result when evaluated signed arithmetic exceeds its representable range.

For defined in-range behavior:

- Every per-record position actually passed to F-c2e9c7166e37 must be representable.
- Every inspection upper endpoint actually evaluated must be representable.
- Every lookup index subtraction actually evaluated must be representable.

The table traversal must also support the required address advances and accesses. A positive count does not validate that the table exists or is sufficiently large.

## Side effects

### Direct effects

The only explicit data modification performed by this function is a one-byte write of value 19 to the leading byte of a qualifying lookup result.

The function does not directly modify:

- The descriptor count or table-address field.
- Record contents.
- The context’s container field.
- The container’s offset-12 field.
- The inspected object’s position field.

The same returned object may be inspected on multiple iterations. If it retains tag 19 after an earlier change, it no longer qualifies for another write because the required initial tag is 8. Dependencies may alter that state between inspections.

### Dependency effects and lifetime

Effects of F-c2e9c7166e37 and F-b05552b91301 are unknown. They must not be assumed to preserve shared state, descriptor contents, future records, context fields, object tags, or object positions.

There is no explicit allocation, deallocation, ownership transfer, retain, release, or cleanup operation in the supplied function. This does not establish that its dependencies perform none.

For normal completion, memory needed by subsequent direct accesses must remain valid despite dependency calls. In particular:

- The captured table address remains in use even if a dependency changes the descriptor.
- Each later record must remain readable when reached.
- Context and container memory used for an enabled inspection must be readable at that inspection.
- A qualifying lookup result must remain readable and writable through the tag update.

The function does not establish synchronization. Concurrent mutation, reentrant modification, and object-lifetime rules are documented_only unknowns beyond the explicit access sequence.

### Errors

There is no explicit error return distinct from the captured count and no validation of nonnull addresses.

A null result from F-b05552b91301 is an ordinary no-change outcome. A null record table, invalid context, invalid container, inaccessible returned object, or arithmetic overflow is not converted into a defined error result by the visible function.

Dependency exceptions, faults, nonlocal exits, and failure conventions are unknown. The normal return contract assumes control returns from each invoked dependency.

## Dependencies

### F-c2e9c7166e37

Invoked once per record when the captured count is positive.

Its arguments are the original context, the current record’s first word, and the derived per-record position. Its return value is ignored.

Its interpretation of those arguments, side effects, ownership behavior, and failure modes are documented_only unknowns. It runs before the optional lookup for the same record, so its changes may affect that lookup.

### F-b05552b91301

Invoked once per processed record only when the inspection flag is nonzero.

It receives a context-derived container and an index formed by subtracting one from a container field. Its result is treated as a nullable address to the tag-and-position layout described above.

Whether it performs bounds checking, accepts negative indices, allocates memory, changes container state, or returns borrowed or owned storage is unknown. The caller performs no explicit release of the result.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Unless stated otherwise, all required memory is valid, arithmetic is representable, and dependencies return normally.

1. **Null descriptor.**  
   With a null descriptor, the result is zero. Neither dependency is called, regardless of the other inputs.

2. **Zero count.**  
   With a readable descriptor containing count zero, the table-address field and count are captured. No record is read, neither dependency is called, and the result is zero.

3. **Negative count.**  
   With count −3, no record processing or optional inspection occurs. The result is −3, not zero and not an error indicator.

4. **Record stride and derived positions.**  
   With count 3, base position 20, inspection disabled, and first record words A, B, and C at table offsets 0, 12, and 24, F-c2e9c7166e37 receives those words in that order with positions 20, 21, and 22. The result is 3.

5. **Inspection sees post-processing state.**  
   If F-c2e9c7166e37 changes the context’s container field, the following enabled inspection obtains its lookup arguments from the resulting context state, rather than from a container captured at function entry.

6. **Half-open range.**  
   With base position 20 and captured count 3, a stable returned object with tag 8 and position 20 or 22 is changed to tag 19. Position 19 or 23 does not qualify. The interval is [20, 23).

7. **Tag mismatch.**  
   A nonnull lookup result with tag 19 is not changed by this inspection. Its position is not needed for qualification.

8. **Null lookup result.**  
   A null result causes no object access or write. Later records are still processed normally.

9. **Whole-range qualification on the first iteration.**  
   With base position 20 and count 3, an object with tag 8 and position 22 may be changed immediately after processing the first record. Qualification is not limited to position 20.

10. **Descriptor mutation does not change the captured request.**  
    If a dependency changes the descriptor count from 3 to 1 after the first record, normal traversal still uses the captured count 3, and the normal return remains 3.

11. **Container field zero.**  
    If the lookup-index source field is zero, the supplied index is −1. The visible function does not reject it. The dependency’s response is unknown.

## Uncertainties

The following remain documented_only:

- Semantic meanings of record words, positions, tags 8 and 19, and the container field.
- Full layouts beyond the accessed offsets.
- Dependency behavior, including mutation, failure, ownership, and lifetime guarantees.
- Overflow behavior and the exact target execution model beyond the represented 32-bit layout.
- Physical memory-access counts, compiler optimizations, and synchronization semantics.
- Behavior when separate reads observe changing container or position fields.
- Whether callers impose stronger validity, range, or exclusivity requirements.

A plausible intent is to process a sequence and retag certain associated objects. That interpretation is not needed for, and does not strengthen, the established contract.

## Evidence

This contract is based solely on the supplied static function body and opaque dependency mapping.

Established evidence includes the null-descriptor branch, captured descriptor fields, 12-byte record stride, ordered processing calls, optional post-call inspection, tag and interval predicates, one-byte update, and return of the captured signed count.

No dependency implementation or runtime observation was supplied. No runtime tests have been performed. Acceptance examples describe consequences of the static behavior under their stated conditions, not verified execution results.
