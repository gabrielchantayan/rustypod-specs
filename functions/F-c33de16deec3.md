<!-- Trusted provenance: F-c33de16deec3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c33de16deec3

## Purpose

F-c33de16deec3 performs descriptor-driven initialization of a destination object, delegating an intermediate operation to F-a1e36949e45c and then installing three 32-bit values at the address returned by that dependency.

A null descriptor selects a default-descriptor path. That path also initializes three locations in the original destination before invoking the dependency. A non-null descriptor bypasses those preliminary destination writes.

**Established behavior:** descriptor selection, preliminary writes, dependency arguments, and three post-call writes are visible in the supplied static evidence.

**Inferred intent:** the descriptor’s first word and the metadata at negative offsets from that word resemble an object-layout or dispatch-table mechanism. The evidence does not establish a particular object model, class hierarchy, or runtime convention. This contract therefore describes words and byte offsets rather than assigning those meanings.

## Interface

### Inputs

The function has three inputs:

| Input | Interpretation established by the evidence |
|---|---|
| Destination | Address of writable storage. The null-descriptor path writes at byte offsets 0, 336, and 340 before calling the dependency. It is also passed unchanged as the dependency’s first argument. |
| Descriptor | Either null or the address of a word-oriented descriptor. A null value selects a descriptor through a global pointer cell. |
| Auxiliary value | An opaque 32-bit value passed unchanged as the dependency’s third argument. This function does not otherwise inspect it. |

The visible representation uses 32-bit words and 32-bit address arithmetic. Word indices therefore correspond to four-byte increments.

The descriptor fields directly used by this function are:

| Byte offset | Width | Role |
|---|---:|---|
| 0 | 32 bits | Header word, copied to the dependency’s returned address after the call |
| 4 | Address position | Start address passed as the dependency’s second argument |
| 16 | 32 bits | Value for the first metadata-directed write |
| 20 | 32 bits | Value for the second metadata-directed write |

No complete descriptor size or layout is established. In particular, the dependency may access additional storage beginning at descriptor offset 4.

### Referenced global state

Three global cells participate in default initialization:

- **Default-descriptor pointer cell:** holds a descriptor address.
- **First default-value cell:** holds a 32-bit value written to destination offset 336.
- **Second default-value cell:** holds a 32-bit value written to destination offset 340.

These names describe their visible uses, not ownership or a wider initialization protocol.

### Output

The function has no explicit return value. Its observable results are memory writes and any effects of F-a1e36949e45c.

The dependency returns an address used by this function. That address is not returned to the caller of F-c33de16deec3.

### Preconditions for defined successful execution

The function performs no visible validation. Successful execution requires, at the times of the relevant accesses:

- Readable global cells.
- A usable descriptor selected either explicitly or through the default path.
- Readable descriptor words at offsets 0, 16, and 20 after the dependency returns.
- A destination satisfying the dependency’s requirements.
- On the default path, writable destination words at offsets 0, 336, and 340.
- A dependency result that permits the header write and both metadata-directed writes.
- Readable metadata at the header-derived addresses described below.
- Address calculations and accesses that are valid for the actual execution environment.

The supplied evidence does not establish alignment guarantees, allocation extents, or behavior for invalid addresses.

## Behavior

### 1. Capture the default descriptor pointer

The function first reads the default-descriptor pointer cell and retains that pointer value.

This read occurs even when the caller supplies a non-null descriptor. In that case, the retained value is not subsequently used by the visible body.

No descriptor contents are read at this initial step.

### 2. Select the descriptor and perform optional default initialization

If the caller’s descriptor is non-null, the function uses that supplied descriptor and skips the preliminary destination writes.

If the caller’s descriptor is null, the following ordered operations occur before the dependency call:

1. Read the default-descriptor pointer cell again, then read the 32-bit word at the address obtained by that second pointer read. Retain that word as the preliminary header.
2. Read the first default-value cell and write its value to destination byte offset 336.
3. Read the second default-value cell and retain its value.
4. Write the preliminary header to destination byte offset 0.
5. Write the retained second default value to destination byte offset 340.
6. Select the pointer captured in step 1 of the overall behavior as the descriptor for the dependency and the remaining operations.

The initial pointer capture and the later pointer-cell read are distinct. The preliminary header is obtained through the later read, whereas the descriptor selected for subsequent use is the initially captured pointer.

There is no intervening dependency call during these preliminary operations. Nevertheless, the visible write order matters if destination storage overlaps global cells or descriptor storage. For example, the write at destination offset 336 precedes the read of the second default-value cell.

### 3. Invoke the dependency

The function calls F-a1e36949e45c with four arguments, in this positional order:

1. The original destination address.
2. The selected descriptor address advanced by four bytes.
3. The unchanged auxiliary 32-bit value.
4. The integer value 1.

The dependency’s returned address becomes the base for all subsequent writes.

The visible body does not require that this returned address equal the original destination. The preliminary writes on the null-descriptor path target the original destination; the post-call writes target the returned address and addresses derived from it.

All post-call behavior is conditional on the dependency returning control normally. No visible recovery path handles a failed, exceptional, or non-returning dependency invocation.

### 4. Install the descriptor header

After the dependency returns, the function reads the 32-bit word at selected descriptor offset 0. It retains this word and writes it to offset 0 of the returned address.

This is a post-call read. It is not necessarily equal to the preliminary header written on the default path: the dependency may have changed descriptor memory, and the two default-path header reads do not necessarily use the same pointer-cell observation.

The retained post-call header also supplies the metadata base for the next write.

### 5. Perform the first metadata-directed write

The function obtains a 32-bit offset from memory located 12 bytes before the retained post-call header value, treating that header value as an address for this lookup.

It writes the word from selected descriptor byte offset 16 to:

**returned address + first metadata offset**

The metadata offset occupies 32 bits and is represented as an integer in the supplied evidence. Negative offsets are not rejected.

The source establishes that this assignment follows the header installation. It does not establish a total order between the assignment’s source-value read and all reads needed to calculate its destination address.

The retained header used for the metadata lookup is the value read from descriptor offset 0 in the preceding phase. It is not replaced by a fresh read from the returned address for this first lookup.

### 6. Perform the second metadata-directed write

After the first metadata-directed assignment, the function reads the current 32-bit word at offset 0 of the returned address.

It uses that freshly read word as an address and obtains another 32-bit offset from memory located 16 bytes before it.

It writes the word from selected descriptor byte offset 20 to:

**returned address + second metadata offset**

As with the preceding assignment, the evidence does not fix the relative evaluation order of the source-value read and every destination-address read within this assignment.

The fresh header read is significant. If the first metadata-directed write overwrites offset 0 of the returned storage, the second metadata lookup uses the replacement header rather than the header previously installed.

### Arithmetic and access widths

The direct field accesses and metadata-directed stores are 32-bit accesses. Descriptor advancement by one word means four bytes. The destination default-field offsets are 336 and 340 bytes, not word counts.

The visible arithmetic uses a 32-bit address representation. This evidence does not justify extending the contract to arbitrary-width pointers, promising bounds checks, or defining portable overflow behavior. Cases that require overflowing address arithmetic remain **documented_only**.

## Side effects

### Direct writes

On the non-null-descriptor path, the visible body performs three post-call 32-bit writes:

1. Header to the returned address.
2. Descriptor-offset-16 value to the first metadata-directed address.
3. Descriptor-offset-20 value to the second metadata-directed address.

On the null-descriptor path, three preliminary 32-bit writes additionally occur, in order:

1. First default value to destination offset 336.
2. Preliminary header to destination offset 0.
3. Second default value to destination offset 340.

These counts describe write operations, not necessarily distinct memory locations. Aliasing may cause several writes to target the same word.

### Aliasing and mutation

The evidence establishes no non-aliasing restriction between the destination, descriptor, global cells, returned storage, or metadata.

Consequently:

- Preliminary destination writes may affect descriptor or global storage used later.
- The dependency may alter descriptor words before their post-call reads.
- Installing the header may change descriptor fields if returned storage overlaps the descriptor.
- The first metadata-directed write may affect the second lookup’s header, metadata, or source value.
- Source values for the two metadata-directed writes are not snapshotted together before either write.

These are consequences of the visible access sequence, not guarantees that every overlapping arrangement is valid.

### Ownership and errors

The visible body contains no explicit allocation, deallocation, ownership transfer, or resource release. The dependency’s ownership behavior is unknown.

There is no explicit status result, null-result check, bounds check, or error branch. Invalid pointers, inaccessible metadata, unsuitable offsets, and dependency failures have no defined handling in the supplied evidence.

The function provides no visible rollback. If execution fails after an earlier write, the evidence does not establish restoration of prior state.

## Dependencies

### F-a1e36949e45c

Called once on every path that reaches the call.

**Visible contract at this call site:**

- Receives the original destination.
- Receives selected descriptor plus four bytes.
- Receives the auxiliary value unchanged.
- Receives integer 1 as its fourth argument.
- Supplies an address that is subsequently written through.

**Unknowns — documented_only:**

- Full parameter semantics.
- Required storage sizes and alignment.
- Whether the result aliases the destination or descriptor.
- Whether the dependency allocates, releases, retains, or transfers memory.
- Which globals or descriptor fields it may modify.
- Failure and exception behavior.
- Whether it always returns.

No preservation of global state or descriptor memory is assumed.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Unless stated otherwise, all referenced storage is valid, arithmetic is representable, and the dependency returns normally.

### Explicit descriptor with distinct target fields

Suppose the descriptor’s post-call header points to readable metadata containing offsets 24 and 40 at header-relative positions −12 and −16. Descriptor offsets 16 and 20 contain 17 and 29. The returned storage does not overlap the descriptor or metadata.

Expected effects:

- No preliminary writes are made to destination offsets 0, 336, or 340.
- The dependency receives descriptor plus four bytes.
- The returned storage receives the header at offset 0, 17 at offset 24, and 29 at offset 40.

### Default descriptor with stable globals

Suppose the default-descriptor pointer remains unchanged during default initialization, and all participating storage is disjoint.

Expected effects before the dependency call:

- Destination offset 336 receives the first default value.
- Destination offset 0 receives the default descriptor’s initial header.
- Destination offset 340 receives the second default value.
- The dependency receives the initially captured default descriptor plus four bytes.

Post-call initialization still reads descriptor fields again; it is not defined by a pre-call snapshot.

### Dependency changes a descriptor field

Suppose the dependency changes descriptor offset 16 from 17 to 93 and returns normally, with no other relevant overlap or mutation.

The first metadata-directed write uses 93. Using the pre-call value 17 would contradict the visible post-call read.

### First directed write replaces the header

Suppose the first metadata offset is zero, so the first directed write overwrites the returned storage’s header with another valid metadata-base value.

The second metadata offset is read from 16 bytes before that replacement value. Reusing the previously retained header for the second lookup would not match the visible behavior.

### Dependency returns different storage

Suppose the dependency returns a valid address distinct from the original destination.

The three post-call writes target that returned storage and its metadata-directed locations. On the default path, the three preliminary writes remain writes to the original destination.

### Invalid dependency result

If the dependency returns null or otherwise unwritable storage, the function has no visible guard before the header write. A specific fault, exception, or recovery outcome is **documented_only**, not an acceptance guarantee.

## Uncertainties

The following remain **documented_only**:

- Semantic meanings of the header, metadata offsets, and default values.
- Complete destination and descriptor layouts.
- Dependency side effects and ownership rules.
- Permitted aliasing arrangements beyond the access consequences described above.
- Alignment requirements, byte order, and memory-protection behavior.
- Concurrent mutation, atomicity, and synchronization guarantees.
- Exact overflow semantics for invalid or non-representable address calculations.
- Operand evaluation order within the metadata-directed assignments.
- Behavior after invalid memory access or dependency failure.

The 32-bit layout is established by the supplied representation; a portable architecture-independent interface is not.

## Evidence

This contract is based solely on the supplied static function body and its opaque dependency mapping.

The static evidence establishes the branch condition, global reads, direct word accesses, statement-level write ordering, dependency arguments, use of the dependency result, and the fresh header read before the second metadata lookup.

Object-model interpretations are inferred intent only. Callee internals and runtime failure behavior are not supplied.

No runtime tests have been performed.
