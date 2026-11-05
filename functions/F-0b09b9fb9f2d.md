<!-- Trusted provenance: F-0b09b9fb9f2d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0b09b9fb9f2d

## Purpose

F-0b09b9fb9f2d performs setup on a context-associated handle, enumerates entries through a dependency, and publishes a descriptor address and a data-address value into destinations selected by that dependency. For each entry whose **first descriptor word, as read after descriptor-address publication, is nonzero**, it invokes an external operation with that value and a freshly calculated context-derived argument.

The timing of that read is part of the contract. A descriptor containing zero when returned by its construction dependency does not necessarily cause status 25: the preceding descriptor-address publication may overwrite that word through aliasing.

The domain meaning of the context, handle, descriptor, data address, and external operation is not established. “Setup,” “enumeration,” “descriptor,” and “publication” describe observable roles, not proven application semantics. In particular, the external operation is not established to clear, initialize, retain, or release memory.

The function is not transactional. Setup effects, dependency effects, and destination writes can survive an early return. No rollback or explicit resource release is visible.

## Interface

The supplied representation exposes four inputs and a 32-bit status result.

| Input or result | Representation and observed use |
|---|---|
| Context | Address represented by a 32-bit integer. Fixed-offset fields are accessed relative to it. |
| Auxiliary argument | A 32-bit value neither read nor forwarded by the visible body. |
| Initial descriptor destination | Address of a writable 4-byte slot. Initializes a mutable descriptor-destination cursor. |
| Initial data-address destination | Address of a writable 4-byte slot. Initializes a mutable data-destination cursor. |
| Status result | Setup’s nonzero status, zero after enumeration terminates, or 25 when the descriptor word actually read after publication is zero. |

The initial destination arguments are not necessarily the addresses ultimately written. Enumeration receives the addresses of the two cursor slots and may replace their contents. Cursor values persist between enumeration calls unless changed.

### Context layout

| Byte offset | Access width | Observable role |
|---|---:|---|
| `0xEC` | 1 byte | Selects setup argument 16 when equal to 4; otherwise selects 8. |
| `0x108` | 4 bytes | Signed quantity shifted left by nine for two independently evaluated arguments per completed nonzero entry. |
| `0x174` | 4 bytes | Handle storage. Setup receives its address; each enumeration attempt receives its current contents. |

The returned descriptor is used as an address from which one 4-byte word is read. That word is subsequently treated as a data address. No other descriptor field is directly accessed.

### Preconditions and validity

Valid execution requires accessible storage for every reached context read and every reached destination write. On an enumeration result of zero, the current destination cursors must designate writable 4-byte locations.

After descriptor-address publication, the returned descriptor address must support the ensuing 4-byte read. The function does not validate that address before dereferencing it. A nonzero word passed to the external operation must meet that dependency’s requirements; those requirements are not supplied.

There are no visible checks for null context, destination validity, descriptor validity, alignment, capacity, arithmetic range, or permitted aliasing. These are execution requirements, not checked failure conditions. The function supplies no defined status specifically for an invalid pointer or memory-access failure.

## Behavior

### Setup and early return

The function first reads the selector byte at context offset `0xEC`. Exactly 4 selects the value 16; every other byte value selects 8. This selection is made once and is not recomputed during enumeration.

It initializes its two mutable destination cursors from the third and fourth inputs, then calls F-6891f8be9185 with:

1. The constant 4.
2. The selected value, 16 or 8.
3. The address of the context field at `0x174`.

If setup returns a nonzero status, the function immediately returns that status. Enumeration and per-entry dependencies are not called on this path.

Setup receives writable storage by address. Its failure does not imply that the context or other state remained unchanged, and the visible function performs no rollback.

### Enumeration and cursor persistence

Following successful setup, the enumeration index starts at zero.

Each enumeration attempt reads the current 4-byte handle at context offset `0x174` and calls F-e9a8d36566ae with:

1. That handle.
2. The current index.
3. The address of the data-destination cursor.
4. The address of the descriptor-destination cursor.

Thus, the cursor originating from the fourth input precedes the cursor originating from the third input in this dependency’s argument order.

A nonzero enumeration result terminates the function with status zero. The particular nonzero result is discarded; the visible contract does not distinguish exhaustion from an error represented by such a result. On that terminating attempt, this function does not dereference either destination cursor.

A zero enumeration result permits entry processing using the current cursor values. The function does not restore either cursor to its original input between attempts. A cursor unchanged by a later enumeration retains its preceding value.

Enumeration’s own writes or other effects remain possible even when its return terminates the function.

### Construction and ordered publication

For each zero enumeration result, the following observable sequence applies:

1. F-8157a180d901 is called with the constant 8.
2. The 4-byte quantity at context offset `0x108` is read and shifted left by nine.
3. F-d9e0ae06e54d is called with the preceding dependency’s result and that shifted quantity.
4. The returned descriptor address is written to the current descriptor destination.
5. The first 4-byte word at the returned descriptor address is read.
6. The word just read is written to the current data-address destination.
7. The saved word is tested against zero.

The publication in step 4 precedes the descriptor read in step 5. Consequently, the value consumed is not necessarily the descriptor’s first word as it existed when F-d9e0ae06e54d returned. Overlapping destination storage can change it before the read.

The data-address publication precedes the zero decision. That decision uses the saved result of the descriptor read, not a second read from either the descriptor or the destination.

If the saved word is zero, the function returns decimal 25, hexadecimal `0x19`. It makes no external-operation call for that entry and performs no later enumeration attempt. Both publication writes have already occurred and are not undone.

The result from F-8157a180d901 is forwarded without a visible validity check. The descriptor address returned by F-d9e0ae06e54d is likewise not checked before publication and dereference. An invalid descriptor address is therefore distinct from a readable descriptor whose post-publication first word is zero.

### External operation and continuation

If the saved descriptor word is nonzero, the function reads the quantity at context offset `0x108` again and independently shifts it left by nine.

It invokes D-01 with:

1. The saved nonzero word interpreted as a data address.
2. The newly calculated shifted quantity.

No return value from D-01 is used. If the call returns normally, the index is incremented by one and enumeration is attempted again.

The visible body contains no retry, rollback, per-entry release, or status check after D-01.

### Widths and arithmetic

The supplied representation uses 4-byte quantities for status, handles, published addresses, the descriptor’s first word, and the size source. The enumeration index and size source are represented as signed 32-bit integers.

The shift count is exactly nine. For nonnegative inputs whose shifted result is representable as signed 32-bit, the calculation equals multiplication by 512. The greatest such input is 4,194,303, yielding 2,147,483,136.

A portable result for negative signed operands or an out-of-range signed left shift is not established by the supplied representation. Checked multiplication, saturation, rejection, and guaranteed unsigned wrapping must not be substituted as established behavior.

The index has no explicit bound check. After processing index 2,147,483,647, the next increment exceeds the signed 32-bit range. Its execution outcome is unresolved here.

The two size-source reads are distinct:

- The first follows F-8157a180d901 and supplies F-d9e0ae06e54d.
- The second follows descriptor construction, both publication writes, and the nonzero decision, and supplies D-01.

Their values need not match. An intervening dependency or an overlapping destination write may change the field. Similarly, the handle is reread for every enumeration attempt rather than cached across the loop.

## Side effects

### Direct memory effects

For every entry accepted by enumeration, the function directly performs:

1. A 4-byte descriptor-address write through the current descriptor destination.
2. A 4-byte saved-word write through the current data-address destination.

The second write occurs only if execution reaches it after the descriptor read. The visible body does not directly initialize other descriptor fields or access the memory identified by the saved data address.

Setup may change the context handle field through its supplied address. Such changes are dependency effects rather than direct field assignments by this function.

### Aliasing and observable consequences

No non-aliasing check is visible.

If both destinations identify the same 4-byte slot, the later data-address publication overwrites the earlier descriptor-address publication.

If the descriptor destination equals the returned descriptor address, publication writes that address into the descriptor’s first word before it is read. For a valid nonzero descriptor address, that read then obtains the descriptor address itself, regardless of an earlier zero word. Subject to subsequent execution remaining valid, the nonzero branch invokes D-01 with that saved address.

If the data-address destination overlaps the descriptor’s first word, the zero decision still uses the saved read value. A later change to the stored descriptor word does not trigger a reread.

If a destination overlaps the context size field, publication can change the argument subsequently calculated for D-01. If it overlaps the context handle field, publication can change the handle supplied to a later enumeration attempt.

These consequences follow from the access order. They do not establish that callers are permitted to arrange such overlap, nor do they define every consequence of partial overlap, invalid alignment, or inaccessible storage.

### Ownership, lifetime, and partial completion

No ownership transfer is established for either construction dependency’s result, the returned descriptor, or its first-word address.

The constant 8 passed to F-8157a180d901 may suggest a storage request, but allocation is inferred intent, not established behavior. The visible function does not explicitly release that result, the descriptor, or the published address, including on the status-25 path.

D-01 may modify, retain, or release its argument according to an unavailable contract. Published addresses are not thereby guaranteed to remain valid after that call or after this function returns.

## Dependencies

Opaque identifiers describe call boundaries only; no dependency is assumed to preserve context, global state, or memory ownership.

| Identifier | Established interaction |
|---|---|
| F-6891f8be9185 | Receives 4, the selected 8-or-16 value, and the context handle-field address. Its nonzero status is propagated. |
| F-e9a8d36566ae | Receives the current handle, index, and addresses of the data and descriptor cursors, in that order. Zero permits entry processing; nonzero terminates with overall status zero. |
| F-8157a180d901 | Receives 8. Its result is forwarded without a visible check. |
| F-d9e0ae06e54d | Receives that result and the first shifted context quantity. Its returned address is published, then dereferenced. |
| D-01 | Document-local opaque identifier for the external operation without a supplied canonical ID. Receives the saved nonzero descriptor word and the second shifted quantity. Its result is unused. |

The described continuation assumes dependencies return normally. Their exception, fault, nonreturn, synchronization, and internal error behavior are not supplied.

## Acceptance examples

These are conditional static acceptance cases, not executed tests.

### Setup selection and failure

With selector byte 4, F-6891f8be9185 receives 16 as its second argument. With any other byte value, it receives 8.

If setup returns 7, the function returns 7 without enumeration. Setup effects already performed are not rolled back.

### Immediate enumeration termination

If setup succeeds and the first enumeration returns any nonzero value:

- Its index argument is zero.
- Neither construction dependency is called.
- This function performs no destination publication.
- The overall result is zero, irrespective of the particular nonzero enumeration result.

### One nonzero entry

Assume both size-source reads observe 2, enumeration supplies valid destinations, and the first descriptor word **read after descriptor-address publication** is nonzero.

F-d9e0ae06e54d receives 1,024 as its second argument. The descriptor address is published before its first word is read; that saved word is then published to the data-address destination. D-01 receives the saved word and 1,024.

If the next enumeration returns nonzero, its index is 1 and the function returns zero.

### Zero observed after publication

Assume the descriptor-address write completes, the subsequent descriptor read obtains zero, and the data-address write completes.

The descriptor address has been published, zero has been published to the data-address destination, and the function returns 25. D-01 and later enumeration are not called. Earlier effects remain.

A descriptor that merely contains zero before publication is insufficient to establish this outcome.

### Initial zero overwritten by overlapping publication

Assume a readable and writable descriptor has a nonzero address and initially contains zero in its first word. Enumeration selects that same address as the descriptor destination. Assume the data-address destination is separate valid storage and later required operations are valid.

Descriptor-address publication replaces the initial zero with the descriptor’s own address. The following read obtains that nonzero address, which is published to the data-address destination and passed to D-01. Status 25 is not selected for this entry.

### Different size observations

If the first size read observes 2 and an intervening effect changes the field to 3 before the second read, F-d9e0ae06e54d receives 1,024 and D-01 receives 1,536, provided the saved descriptor word is nonzero. Reusing the first calculated argument would change the visible behavior.

### Redirected destinations and invalid descriptors

If enumeration replaces either cursor, publication uses its replacement. A later unchanged cursor retains that replacement.

If construction returns an unreadable descriptor address, the function first attempts descriptor-address publication and then attempts the descriptor read. No status-25 fallback for the unreadable address is established; the outcome depends on the execution environment.

## Uncertainties

The following remain **documented_only**:

- Domain meanings, size units, and the significance of setup constants.
- Whether either construction dependency allocates storage or can fail through particular return values.
- Descriptor and data-address ownership, lifetime, and release obligations.
- D-01’s operation, return type, and retention or release behavior.
- Whether nonzero enumeration results distinguish exhaustion from errors.
- Dependency mutations of context, shared state, cursor storage, or published memory.
- Caller-permitted aliasing, alignment requirements, and partial-overlap behavior.
- Exact execution behavior for signed arithmetic outside the representable range.
- Fault handling, exceptions, and concurrency guarantees.

No synchronization is visible in this function. That observation neither establishes concurrency safety nor proves that dependencies lack synchronization.

## Evidence

The supplied static representation establishes the field offsets, accessed widths, branch conditions, constants, dependency argument order, cursor persistence, publication-before-read ordering, separate size reads, index progression, and explicit return paths described above.

In particular, descriptor-address publication precedes the descriptor-word read, and the zero decision follows data-address publication while using the saved read value. The zero-return condition therefore concerns the word actually observed after the first publication, not the descriptor’s contents before publication.

Resource construction and collection initialization are inferred intent only. Dependency bodies, caller contracts, ownership rules, and runtime behavior were not available. No runtime tests have been performed.
