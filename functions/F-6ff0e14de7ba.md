<!-- Trusted provenance: F-6ff0e14de7ba; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6ff0e14de7ba

## Purpose

F-6ff0e14de7ba updates three consecutive 32-bit fields in a destination object using a source flag, a conditionally read source value, and a conditionally traversed global reference chain.

The first destination field receives either zero or the source’s 32-bit value. A zero value clears the remaining two destination fields without consulting the global reference chain. A nonzero value causes the second destination field to receive a globally obtained reference and the third to receive either zero or a 32-bit value reached through that reference.

This is an established description of the supplied static evidence. Interpreting the fields as an enabled value, context reference, and associated value is possible intent, not an established semantic contract.

## Interface

### Inputs

The function takes two address-valued arguments:

- **Destination base:** identifies storage containing writable 32-bit fields at byte offsets 8, 12, and 16.
- **Source base:** identifies storage containing a readable one-byte flag at byte offset 4 and, conditionally, a readable 32-bit value at byte offset 8.

The supplied representation uses integer-valued arguments for addresses. The observable memory operations use one-byte and four-byte widths. The target pointer representation and address-arithmetic rules are not independently established.

### Source layout

| Byte offset | Width | Meaning established by the function |
|---|---:|---|
| 4 | 1 byte | Selector: zero suppresses the source value read; any nonzero byte enables it. |
| 8 | 4 bytes | Value copied to the first destination field when the selector is nonzero. |

No other source fields are explicitly accessed.

### Destination layout

| Byte offset | Width | Result |
|---|---:|---|
| 8 | 4 bytes | Zero if the source selector is zero; otherwise the source value. |
| 12 | 4 bytes | Zero if the first result is zero; otherwise a globally obtained reference. |
| 16 | 4 bytes | Zero if the first result is zero or the nested reference is zero; otherwise a value read through the nested reference. |

The destination requires accessible storage through byte offset 19 for a normally completing invocation. The source requires access through offset 4 in all cases and through offset 11 when its selector is nonzero. These requirements describe the accessed ranges, not a complete object definition.

### Output and return

The function has no explicit return value. Its result is the three destination writes.

There is no explicit error result, validation result, or success indicator. Normal completion assumes that every memory access selected by the branch conditions is valid.

## Behavior

### Source selection and first write

The first observable memory read obtains the one-byte source selector at offset 4.

- If that byte is zero, the selected value is zero. The source field at offset 8 is not read.
- If that byte is nonzero, the function reads the four-byte source field at offset 8 and uses that value without normalization or range checking.

The selected value is then written as four bytes to destination offset 8. This write occurs before any access to the global reference chain.

The next branch depends on whether the selected 32-bit value is zero. It does not depend solely on whether the selector was nonzero. Thus, a nonzero selector paired with a zero source value takes the clearing path.

All nonzero source values select the reference-traversal path. No positivity test, mask, or boolean conversion is shown. A bit pattern interpreted as a negative signed value remains nonzero and follows that path.

### Zero-value path

When the selected value is zero:

1. The function writes four-byte zero to destination offset 12.
2. It then writes four-byte zero to destination offset 16.
3. It returns.

On this path, the function does not access the global reference chain.

The write to destination offset 8 has already occurred, so the established write order is offset 8, then offset 12, then offset 16.

### Nonzero-value path

When the selected value is nonzero, the function performs the following ordered accesses:

1. It obtains a global base reference through the global indirection shown in the supplied representation.
2. It reads a four-byte reference at byte offset 48 from that base.
3. It writes that reference to destination offset 12.
4. Using the reference just obtained, it reads a four-byte nested reference at byte offset 3936.
5. If the nested reference is zero, it selects zero for the final destination field.
6. If the nested reference is nonzero, it reads a four-byte value at byte offset 24 from the nested reference.
7. It writes the selected final value to destination offset 16 and returns.

The offset conversions are exact: hexadecimal 0x30 is 48, 0xF60 is 3936, and 0x18 is 24.

The reference written to destination offset 12 is not tested for zero before the access at offset 3936. A zero value at that stage does not select a defined clearing branch. It instead participates in the subsequent address calculation, whose validity depends on the execution environment.

Only the nested reference read at offset 3936 receives an explicit zero check before further traversal.

### Ordering, aliasing, and retained values

The function does not take a complete input snapshot before writing the destination.

The source selector and any required source value are read before the first destination write. Consequently, ordinary destination/source overlap does not cause those already-read inputs to be reread. Passing the same base for both arguments is not explicitly rejected.

Overlap with the global reference chain has more significant consequences:

- The write to destination offset 8 occurs before global-chain reads. If it overlaps memory used by those reads, the subsequent traversal observes memory after that write.
- The write to destination offset 12 occurs before the read at offset 3936 from the obtained reference. If those locations overlap, the later read can observe the newly written value.
- The obtained reference used for that offset-3936 read is retained from the earlier read; the function does not reread destination offset 12 to recover it.
- A nonzero nested reference is likewise used for the offset-24 read without a fresh lookup through earlier links.

These ordering properties matter for overlapping storage. They do not establish atomicity, synchronization, or safe concurrent access.

## Side effects

### Direct memory effects

On normal completion, the function explicitly performs exactly three four-byte destination writes, in this order:

1. Offset 8.
2. Offset 12.
3. Offset 16.

It does not explicitly write the source, the global slot, or objects reached through the reference chain. Such storage can nevertheless be modified when it aliases a destination field.

No other destination fields are explicitly initialized or cleared. In particular, the function does not clear an entire object or copy an entire source structure.

### Ownership and lifetime

No allocation, deallocation, retain operation, release operation, or other function call appears in the supplied body.

Copying a reference into destination offset 12 therefore does not establish ownership transfer or lifetime extension. The caller must not infer that the referenced object remains valid after the invocation merely because its reference was stored.

The final field is copied as four bytes. Whether that field represents a scalar, handle, address, or another encoded value is unknown.

### Failure and partial updates

The function contains no explicit recovery or rollback.

If execution cannot complete a selected memory access, earlier writes may already have occurred. For example:

- A failure while obtaining the global reference can occur after destination offset 8 has been updated.
- A failure while reading the nested reference can occur after destination offsets 8 and 12 have been updated.
- A failure during the final referenced-value read can leave destination offset 16 unchanged from its previous contents.

The supplied evidence does not establish the runtime manifestation of an invalid access. A trap, exception, process termination, or other platform-specific behavior must not be presented as an explicit error API of this function.

## Dependencies

### Function dependencies

No calls to other functions are present. There are therefore no callee contracts or additional opaque function IDs to specify.

### Data dependencies

The nonzero-value path depends on:

- A readable global indirection yielding a base reference.
- A readable four-byte field at offset 48 from that base.
- A readable four-byte field at offset 3936 from the resulting reference.
- When that nested field is nonzero, a readable four-byte field at offset 24 from the nested reference.

The zero-value path does not depend on any of these global-chain locations.

The global object layout, initialization mechanism, lifetime, and synchronization policy are not supplied. They remain external prerequisites rather than behavior implemented by this function.

## Acceptance examples

These are conditional acceptance cases derived from static evidence. They are not reports of executed tests. Unless otherwise stated, assume valid, suitably aligned memory, stable references during the invocation, and no overlap between the destination and the referenced storage.

### Zero selector

Given a zero source selector, the completed destination fields at offsets 8, 12, and 16 must all contain four-byte zero.

The source field at offset 8 must not be read. The global chain must not be accessed. Its validity is therefore irrelevant to this particular execution path.

### Nonzero selector with zero value

Given a selector byte of 1 and a zero source value, the source value must be read. All three destination fields must then become zero.

A nonzero selector alone must not trigger global traversal.

### Arbitrary nonzero selector

Given a selector byte of 0x80 and a source value of 7, the selector must be treated as nonzero. Destination offset 8 must receive 7, followed by the nonzero-value traversal.

There is no requirement that the selector equal 1.

### Nonzero value with absent nested reference

Given a nonzero selected value, a valid globally obtained reference, and a zero nested reference at offset 3936:

- Destination offset 8 receives the selected value.
- Destination offset 12 receives the globally obtained reference.
- Destination offset 16 receives zero.
- No read through the zero nested reference at offset 24 occurs.

### Nonzero value with present nested reference

Given a selected value of 7, a valid globally obtained reference, and a nonzero nested reference whose offset-24 field contains the four-byte pattern 0xA5A5A5A5, the final destination fields must contain, respectively, 7, the obtained reference, and 0xA5A5A5A5.

The final pattern is copied without an additional nonzero test or transformation.

### Signed-negative source value

Given a nonzero selector and a source value represented as signed −1 in the supplied 32-bit model, destination offset 8 must receive that value and the global traversal must occur.

A rejection of negative values would add behavior not present in the evidence.

### Identical source and destination bases

Given the same base for both arguments and a nonzero selector, the source value at offset 8 is read before that location is written. The branch decision uses that selected value.

The function does not independently reject this overlap. Validity of later global-chain accesses remains a separate precondition.

## Uncertainties

The following are **documented_only**:

- **Application meaning:** the semantic names and purposes of the fields and referenced objects are unknown.
- **ABI details:** native pointer width, byte order, alignment requirements, and calling convention are not independently established. The supplied representation establishes one-byte and four-byte accesses.
- **Address arithmetic:** overflow, wrapping, and pointer-conversion behavior are not specified. Valid-use examples assume the required address calculations designate accessible storage.
- **Global linkage:** the global indirection is visible, but its declaration, initialization, and external mutation rules are not supplied.
- **Concurrency:** no locking, barriers, atomic operations, or consistency guarantees are shown.
- **Ownership:** stored reference values do not establish ownership or permission to retain or release an object.
- **Fault handling:** invalid-access behavior and interception by the surrounding runtime are unknown.
- **Broader object invariants:** the evidence does not establish whether other fields must be initialized or whether callers impose additional alias restrictions.

Inferred intent is limited to viewing the operation as propagation of a selected value together with related references. That interpretation is not necessary to satisfy the observable contract.

## Evidence

The established behavior comes solely from the supplied static function representation: its conditional one-byte source read, conditional four-byte source read, ordered destination writes, conditional global-reference traversal, nested zero check, and absence of function calls or explicit error handling.

The representation supports the access widths, offsets, branch conditions, and ordering described above. It does not establish business semantics, ownership rules, ABI details beyond the represented accesses, or runtime fault outcomes.

No runtime tests have been performed. The acceptance examples are static deductions, conditional on their stated memory and execution preconditions.
