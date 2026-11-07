<!-- Trusted provenance: F-708a2ebfb28e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-708a2ebfb28e

## Purpose

F-708a2ebfb28e conditionally reads a transformed 32-bit value from a record selected by dependency F-e81ed1edd673. It validates the initial handle, rejects a zero selection result, checks two record bytes, applies a length comparison, and writes a scaled decoded value to the caller’s output location.

The established behavior is a byte-level transformation and a sequence of checks. Interpreting the selected data as a record with a 20-byte header and a length field is useful inferred intent, not an independently established format specification.

The supplied static representation establishes five return paths: four return values obtained from unresolved global words, and one literal-zero success return. The four global values are not known and are not established to be distinct or nonzero.

## Interface

The function has four inputs:

| Input | Represented form | Role |
|---|---|---|
| Selection token | 32-bit opaque value | Passed unchanged as the dependency’s second argument. |
| Output destination | Pointer to a 32-bit integer object | Receives one 32-bit result if all checks pass. |
| Buffer handle | Pointer to a 32-bit integer object | Its contents provide the represented buffer base address. The handle itself is passed to the dependency. |
| Requested offset | Signed 32-bit integer | Participates in the unsigned length check and in the eventual address calculation. No explicit nonnegative check exists. |

The return value is a 32-bit word.

For clarity, this contract assigns descriptive labels to four unresolved return-value globals:

- **Invalid-input status:** returned by the initial guard.
- **No-selection status:** returned when the dependency returns zero.
- **Marker-rejection status:** returned when either checked marker byte is nonzero.
- **Length-rejection status:** returned when the unsigned length comparison fails.

A separate unresolved global word supplies the **result multiplier**. These labels describe use, not known numeric values or guaranteed semantic meanings.

### Memory requirements

A null buffer handle and a null output destination are explicitly checked. Other memory validity is not checked.

A nonnull buffer handle must be readable even when the output destination is null: the handle’s contents are inspected before the output-null condition is reached.

When execution reaches record processing, the current handle contents must identify memory valid for the accesses described below. When execution reaches the final store, the output destination must be writable as a 32-bit integer object.

The supplied representation uses 32-bit integers to hold the buffer base and intermediate offsets. It does not independently establish the complete address-space model, alignment rules, or behavior of an invalid address. Valid-memory preconditions must therefore cover the actual target’s addressing and alignment requirements.

No input allocation size, backing-buffer size, or ownership descriptor is supplied. The function does not establish that its computed addresses lie inside a real allocation.

## Behavior

### 1. Initial validation

The initial checks are short-circuited in this order:

1. Check whether the buffer handle is null.
2. If it is nonnull, read its 32-bit contents and check whether they are zero.
3. Only if those contents are nonzero, check whether the output destination is null.

If any condition rejects the call, the function reads and returns the invalid-input status.

Consequences:

- A null buffer handle is not dereferenced.
- A nonnull handle containing zero prevents the dependency call.
- A null output destination does not protect against an invalid nonnull buffer handle: the handle is read first.
- The wrapper performs no output store on this path.

This validation does not prove that the represented base address points to readable data.

### 2. Selection dependency

After the initial checks pass, the function invokes F-e81ed1edd673 with:

1. The buffer handle.
2. The selection token.

The dependency’s return is treated as an unsigned 32-bit value, called the **selected offset** in this contract.

A return of zero causes the function to read and return the no-selection status. Every nonzero unsigned return proceeds to record processing. In particular, a return of one is accepted; there is no upper-bound rejection at this stage.

The initial buffer-base value is not retained for record processing. After a nonzero dependency result, the function reads the buffer handle again and uses that new value as the base.

Therefore:

- The dependency may change which base is used.
- A nonzero base before the call does not establish a nonzero or valid base afterward.
- There is no second null-base check.
- Nothing supplied establishes that the dependency preserves the output object, global words, buffer contents, or memory ownership.

### 3. Marker checks

Using the post-dependency base, the function first reads the byte at:

**base + selected offset + 15**

It multiplies that byte’s character interpretation by 23 and narrows the result back to a character-sized value. If the narrowed result is nonzero, the function immediately reads and returns the marker-rejection status.

Only if that first transformed byte is zero does it read the byte at:

**base + selected offset + 14**

The same transformation and zero test apply. A nonzero result returns the marker-rejection status.

Under the represented eight-bit truncation behavior, multiplication by 23 has no nonzero byte mapping to zero: 23 is relatively prime to 256. Thus the effective acceptance condition is that both marker bytes are zero. This equivalence does not depend on whether the initial character interpretation is signed, provided the narrowing retains the low eight bits.

The access order matters:

- Relative byte 15 is checked before relative byte 14.
- A rejection at byte 15 prevents the read of byte 14.
- Either marker rejection prevents all subsequent length-field and selected-value reads.

### 4. Length-field decoding and comparison

If both marker checks pass, the function reads four bytes beginning at:

**base + selected offset**

Each byte is interpreted as unsigned and transformed independently:

**transformed byte = (23 × byte) modulo 256**

The four transformed bytes form one unsigned 32-bit value in most-significant-byte-first order. Call this value the **decoded length**.

For four bytes in ascending address order, the first transformed byte supplies bits 31–24, the second bits 23–16, the third bits 15–8, and the fourth bits 7–0.

The rejection condition is:

**(decoded length − 20) modulo 2³² < (requested offset + 4) modulo 2³²**

The comparison is unsigned.

If it is true, the function reads and returns the length-rejection status. Equality passes.

Important boundaries:

- A decoded length of exactly 20 produces a left operand of zero.
- A decoded length below 20 wraps during unsigned subtraction. It is not automatically rejected.
- Negative requested offsets are not rejected as such.
- The addition of four on the right uses unsigned 32-bit arithmetic, so it can wrap.
- This comparison is not a general allocation-bounds check.

The representation does not impose a relative evaluation order among the four byte reads used to construct the decoded length. Their address significance is established; their individual read order is not.

### 5. Value decoding and output

If the length comparison passes, the function forms a value-relative offset from:

**selected offset + 20 + requested offset**

The selected offset makes the represented arithmetic unsigned and 32-bit. The resulting bit pattern is assigned to a signed 32-bit temporary before the final base-relative address calculation. On the apparent 32-bit target this corresponds to wrapped offset arithmetic. The supplied representation alone does not establish portable signed-overflow or out-of-range conversion behavior on another target.

Starting at the resulting address, four bytes are decoded using the same transformation as the length field:

- Multiply each unsigned byte by 23.
- Retain its low eight bits.
- Assemble the transformed bytes most-significant first.

The decoded 32-bit value is multiplied by the result multiplier. The low 32 bits are stored at the output destination. No overflow check or saturation is present.

The function then returns literal zero.

The four selected-value reads and the multiplier read must provide the store’s value, but their internal evaluation order is not established by the supplied representation.

## Side effects

### Direct effects

On the successful path, the function performs one explicit 32-bit store to the output destination.

On each rejection path, it performs no explicit output store. This does **not** guarantee that the output object remains unchanged on paths that called F-e81ed1edd673: the dependency’s effects are unknown.

The wrapper contains no explicit allocation, release, buffer write, or handle write.

### Aliasing

No non-aliasing requirement or check appears in the supplied representation.

The output destination may overlap the handle, record data, or another readable object, provided the accesses themselves remain valid. The successful store occurs after the record data needed for its value has been evaluated. There are no explicit later buffer reads.

Consequently, a successful output store can overwrite the handle or source bytes when the caller supplies overlapping storage. This contract does not assume such overlap is intended.

### Global state

The return-status words are read on their corresponding return paths. The result multiplier is read on the successful computation path.

Their values are not established to be immutable. In particular, no preservation guarantee is available for the dependency. A status or multiplier used after the dependency must not be assumed to equal its pre-call value.

Concurrent modification, memory-mapped I/O semantics, and synchronization requirements are not established.

### Errors outside the explicit checks

The function supplies no explicit handling for inaccessible memory, invalid computed addresses, alignment faults, or dependency failure mechanisms other than a zero return.

No rollback is present. If the dependency changes state before a later rejection, the wrapper does not undo those changes.

## Dependencies

### F-e81ed1edd673

**Established interface use**

- Called only after the initial validation passes.
- Receives the buffer handle followed by the opaque selection token.
- Returns a value interpreted as unsigned 32-bit.
- Zero causes immediate return of the no-selection status.
- Every nonzero value is used as the selected offset.

**Unknown behavior**

The supplied material does not establish:

- How selection works.
- Whether the token names, indexes, or otherwise identifies a record.
- Whether the result is bounded by an allocation.
- Whether the dependency mutates or replaces the buffer.
- Whether it changes globals or the output object.
- Whether it allocates, retains, transfers, or releases memory.
- Whether it can terminate abnormally or fail to return.

Callers must not infer ownership or state-preservation guarantees from this wrapper.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Except where stated otherwise, all accessed memory is valid, the dependency returns normally, and address arithmetic does not encounter an unresolved target-specific condition.

### Initial rejection

With a null buffer handle, the function returns the invalid-input status without calling the dependency or reading through the handle.

With a readable handle containing zero, it returns the same status without calling the dependency.

With a readable handle containing a nonzero base and a null output destination, it reads the handle, then returns the same status without calling the dependency.

### Zero versus nonzero selection

If the dependency returns zero, the function returns the no-selection status without reading the post-call buffer base or record bytes.

If it returns one, selection itself does not reject the call. Record processing uses relative offset one from the subsequently reloaded base.

### Post-call base replacement

Suppose the dependency replaces the handle’s base value and returns a nonzero offset. Marker, length, and value accesses use the replacement base, not the value observed during initial validation.

If the replacement value is zero, the wrapper does not convert that condition into the invalid-input status. It proceeds toward the computed marker address.

### Marker short-circuit

If relative byte 15 is nonzero, the function returns the marker-rejection status without reading relative byte 14.

If byte 15 is zero and byte 14 is nonzero, it reads both in that order and returns the marker-rejection status.

### Exact length boundary

Suppose the transformed length is 24 and the requested offset is zero. The comparison is four less than four, which is false. The length check passes, and the selected value begins at relative offset 20.

With the same requested offset and a transformed length of 23, the comparison is three less than four. The function returns the length-rejection status without reading the selected value.

### Byte transformation and scaling

Suppose the length bytes are 0, 0, 0, and 24. Their transformed value is 40 because 24 × 23 modulo 256 is 40.

With requested offset zero and both marker bytes zero, that length passes the comparison. If the selected value bytes are 1, 2, 3, and 4, the decoded value is hexadecimal **0x172E455C**. With a result multiplier of one, that exact 32-bit pattern is stored, followed by a literal-zero return.

### Unsigned underflow

Suppose the decoded length is zero and the requested offset is zero. The comparison’s left operand becomes 4,294,967,276, so it is not less than four. The check passes despite the apparent length being smaller than the inferred header.

A valid selected-value address is still a separate precondition.

### Negative requested offset

For a requested offset of −4, the comparison’s right operand becomes zero. No unsigned left operand is less than zero, so the length comparison cannot reject this case.

On the apparent 32-bit wrapped-address model, the selected-value position is 16 bytes after the selected record start rather than 20. Successful memory access is not established merely by the comparison passing.

## Uncertainties

### Inferred intent

- The dependency likely selects a record.
- The first decoded word likely describes record length.
- The constant 20 likely represents a header size.
- The requested offset likely addresses a four-byte value following that header.
- The two checked bytes likely act as flags or markers.

These interpretations are not required to describe the established operations.

### documented_only

The following remain unresolved:

- Numeric values and distinctness of all four rejection statuses.
- Value, signed interpretation, and mutability of the result multiplier.
- Complete buffer layout and allocation bounds.
- Dependency side effects, ownership, and failure behavior.
- Full target address semantics and alignment requirements.
- Behavior of address arithmetic outside valid target ranges.
- Character-narrowing semantics beyond the represented eight-bit interpretation.
- Concurrent mutation and synchronization guarantees.

A zero return alone cannot be proven to distinguish success from every rejection because the rejection-status values are unavailable.

## Evidence

This contract is based solely on the supplied static function representation and the supplied mapping of its dependency to F-e81ed1edd673.

Established evidence includes the guard order, dependency arguments, zero-selection branch, post-call handle reload, marker short-circuit order, byte transformation, unsigned length comparison, final output store, and return paths.

Target-specific arithmetic interpretations and format intent are identified separately above. No runtime tests have been performed.
