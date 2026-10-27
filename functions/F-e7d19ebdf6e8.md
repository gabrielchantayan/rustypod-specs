<!-- Trusted provenance: F-e7d19ebdf6e8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e7d19ebdf6e8

## Purpose

F-e7d19ebdf6e8 selects one of two candidate byte addresses, uses the first byte at the selected address as an unsigned displacement, and returns the resulting address. Before returning that address, it reads the byte located there and writes its zero-extended value to a caller-provided output location.

If neither candidate is selected successfully, the function returns a null pointer and leaves the output location untouched.

The established behavior is address selection followed by a single-level byte-offset lookup. Interpreting the selected storage as a record, packet, string, or other particular data structure would be inferred intent, not an established contract.

## Interface

The function has three inputs and a byte-pointer result:

| Input or result | Behavioral role |
|---|---|
| Candidate table | Address of storage containing two consecutive pointer-valued slots. |
| Output location | Address at which an unsigned integer result is written on the successful lookup path. |
| Selection flag | Integer controlling whether the first candidate is considered. Zero skips it; every nonzero value permits it. |
| Return value | Address of the byte whose value was written, or null when no nonnull candidate is selected. |

### Widths and layout

The supplied representation describes the candidate table as consecutive four-byte elements. The first candidate occupies the element at byte offset 0; the fallback candidate occupies the element at byte offset 4. Each element is interpreted as a byte address.

This establishes a four-byte slot layout in the represented program. It does not establish a portable layout for an environment with a different pointer representation. In particular, replacing the slots with wider native pointers would change the table layout.

The selected displacement and the value fetched at the resulting address are unsigned bytes, with values from 0 through 255. The output is an unsigned integer represented as 32 bits in the supplied type model. The fetched byte is zero-extended: values from 128 through 255 remain positive values of that magnitude rather than becoming negative.

The selection flag is represented as an integer consistent with the same 32-bit model. Its magnitude and sign have no further role: only comparison with zero matters.

The source does not establish byte order, atomicity of accesses, or the full platform ABI.

### Access preconditions

Required storage depends on the path taken:

- The table slot actually consulted must be readable.
- If the first candidate is selected, the fallback slot is not accessed.
- If the selection flag is zero, the first slot is not accessed.
- A selected nonnull candidate must support a readable byte at its base address.
- The address obtained by adding the displacement must support another readable byte.
- On that lookup path, the output location must support the unsigned-integer write, including any platform alignment requirements.

A null output location is not accessed when selection produces no nonnull candidate. It is not supported on a successful lookup path.

A nonnull address alone does not establish that any of these access preconditions hold.

## Behavior

### Candidate selection

A nonzero selection flag gives the first candidate priority. The function reads that slot and uses its address if it is nonnull. In that case, it does not read the fallback slot.

If the flag is zero, the function proceeds directly to the fallback slot without reading the first slot.

If the flag is nonzero but the first slot contains null, the function reads the fallback slot. Thus, a nonzero flag enables preference for the first candidate; it does not prohibit fallback.

If the fallback slot is consulted and contains null, the function returns null. No candidate bytes are read, and no output write occurs.

These distinctions govern accesses as well as values. An inaccessible fallback slot is irrelevant when a nonnull first candidate has already been selected. An inaccessible first slot is irrelevant when the selection flag is zero.

### Lookup and result production

After selecting a nonnull candidate, the function:

1. Reads the unsigned byte at the selected base address.
2. Uses that value as a displacement measured in bytes.
3. Reads the unsigned byte at the base address plus that displacement.
4. Writes the fetched value, zero-extended to the output integer width.
5. Returns the address used for the second byte read.

The displacement is neither a signed offset nor an element count scaled by a larger type. Its range restricts the mathematical target to the selected base through 255 bytes beyond it, inclusive. The function performs no visible bounds check to establish that this region exists.

There is no extra increment. A displacement of one selects the immediately following byte; a displacement of zero selects the base itself.

For a zero displacement, the supplied behavior contains two byte reads at the same address: the displacement read and the result read. With stable ordinary memory, both yield zero. If memory can change between those reads, equality is not guaranteed by this function.

The displacement is not subsequently used as a length, and the function does not scan, iterate, or follow another pointer.

### Arithmetic and failures

No explicit overflow check or address-normalization rule accompanies the address addition. The source establishes byte displacement, but not a portable result when the computed address exceeds the applicable object or address-space limits. Such cases must not be specified as supported wraparound.

Null candidate selection is the only explicit unsuccessful outcome. It is reported by a null return without an output update. The output is not cleared, initialized, or assigned a separate error code.

Invalid memory accesses have no explicit recovery path. The available evidence does not establish whether a particular invalid access traps, is intercepted elsewhere, or has another platform-dependent consequence.

## Side effects

The sole explicit memory modification is the unsigned-integer write through the output location on the successful lookup path. The function does not explicitly modify either candidate slot, the displacement byte, or the fetched byte independently of that write.

Aliasing can nevertheless make the output write alter any of those locations. There is no non-aliasing check or protective copy of the surrounding storage.

For example, if the output location overlaps the selected bytes, the result byte is fetched before the write changes them. If the output overlaps a candidate slot, that slot can be overwritten after selection has completed. Subject to valid accesses, the returned address remains the address already computed; the function does not reread the table after writing the output.

The write spans the output integer width, not merely one byte. Its byte-level effects depend on platform byte order.

The function makes no explicit allocation, deallocation, ownership transfer, or lifetime extension. Its returned pointer is derived from caller-supplied storage, not newly created storage. The caller must not infer permission to release that address or assume that the underlying storage remains live.

No synchronization operation is visible. Concurrent modification, memory-mapped I/O, and unusual memory-access semantics require additional environmental specifications.

## Dependencies

No other function is explicitly called. There are consequently no additional opaque function dependencies to enumerate.

Execution depends on the environment’s pointer representation, byte-addressing rules, memory accessibility, alignment requirements, integer storage representation, and handling of invalid accesses. No unobserved callee behavior is needed to explain the supplied operation, and no preservation or ownership guarantees are inferred from unknown callees.

## Acceptance examples

These examples are conditional static expectations, not results of executed tests. Unless stated otherwise, all referenced storage is valid, sufficiently sized, suitably aligned, and stable throughout the call.

### First candidate takes priority

Given a nonzero selection flag, a nonnull first candidate, a displacement byte of 3 at its base, and a byte value of 200 at base plus 3:

- The first candidate is selected.
- The fallback slot is not read.
- The output receives the 32-bit unsigned value 200.
- The returned address is the first candidate’s base plus 3.

### Zero flag skips the first candidate

Given a zero flag and a nonnull fallback candidate whose displacement byte is 2:

- The first slot is not read.
- The lookup uses the fallback base plus 2.
- The output receives the byte at that address, zero-extended.
- That address is returned.

The first slot’s stored value does not affect this case.

### Null first candidate permits fallback

Given any nonzero flag, a null first candidate, and a nonnull fallback candidate, the function performs the lookup using the fallback. A negative nonzero flag has the same selection effect as a positive nonzero flag.

### No candidate leaves the output unchanged

Given a zero flag and a null fallback candidate, the function returns null without writing the output. The same result occurs with a nonzero flag when both candidates are null.

An existing output value therefore remains unchanged; it must not be interpreted as a fresh result from this call.

### Zero and maximum displacements

With a stable zero byte at the selected base, the function returns that base and writes zero.

With a displacement byte of 255, it reads the byte exactly 255 bytes beyond the selected base. There is no signed conversion to a negative displacement. This example is accepted only if that target address is valid.

### Output aliases selected storage

If a valid output location overlaps the target byte, the byte’s pre-write value is fetched and used to form the integer result. The later integer write may change the target and adjacent bytes. The returned address still identifies the original target location.

## Uncertainties

The following are **documented_only**:

- The semantic meaning and allocation sizes of the candidate regions.
- Whether the byte displacement denotes an offset field in a larger format.
- The full platform ABI, byte order, and alignment rules.
- Pointer-addition behavior outside valid object or address-space bounds.
- Invalid-access handling and any externally installed recovery mechanism.
- Concurrency, atomicity, and behavior for nonordinary memory.
- Caller ownership, storage lifetime, and intended output-aliasing restrictions.

None of these unknowns supplies an implicit validation, fallback, ownership, or recovery guarantee.

## Evidence

Established static evidence consists of the supplied function body and its represented types: conditional candidate-slot reads, null-based fallback, an unsigned-byte displacement read, a second unsigned-byte read, an unsigned-integer output write, and return of the second read’s address. No explicit calls, allocation operations, loops, bounds checks, or error handlers appear.

“Preferred candidate” and “fallback candidate” describe the observed selection behavior. Any stronger interpretation of the stored data remains inferred intent.

No runtime tests have been performed. The acceptance examples are conditional consequences of the static behavior, not observations from execution.
