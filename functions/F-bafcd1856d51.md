<!-- Trusted provenance: F-bafcd1856d51; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bafcd1856d51

## Purpose

F-bafcd1856d51 conditionally transfers a 64-byte, sixteen-word region from an indexed internal object to a caller-supplied destination. It first obtains an object address through a table lookup, then rejects a null destination, then calls F-828f890d3c4d. Only a dependency result equal to 1 permits the transfer.

Following the transfer, the function examines bit 8 of a separate 32-bit word in the selected object. That bit determines whether the function returns zero or `0xFFFFFF13`. Both outcomes occur **after all sixteen destination writes**.

The apparent intent is to retrieve a fixed-size object record subject to an eligibility check, with an additional status reported after retrieval. The meanings of the record, the eligibility condition, and the status bit are not established.

This contract describes the supplied static evidence. No runtime tests have been performed.

## Interface

### Inputs

| Input | Representation | Established use |
|---|---|---|
| Selector | 32-bit value | Its low 16 bits, interpreted as a signed integer, select a table entry. The full original 32-bit value is passed to F-828f890d3c4d. |
| Destination | Pointer to writable 32-bit words, or null | Receives sixteen consecutive words when the dependency returns exactly 1. |

The destination has no accompanying length. On a transfer path, the caller must provide storage covering destination byte offsets `0x00` through `0x3F`, inclusive.

The supplied representation uses 32-bit words for the copied data and return value. It also represents the selected object address through a 32-bit integer-sized value. Exact platform pointer rules, alignment requirements, and address-overflow behavior are not independently established.

### Return value

| Return bit pattern | Signed 32-bit interpretation | Condition |
|---|---:|---|
| `0xFFFFFF0B` | −245 | The destination is null, after the initial object lookup has completed. |
| `0x00000003` | 3 | The destination is non-null and F-828f890d3c4d returns anything other than exactly 1. |
| `0x00000000` | 0 | The dependency returns 1, the transfer completes, and the subsequently read status word has bit 8 clear. |
| `0xFFFFFF13` | −237 | The dependency returns 1, the transfer completes, and the subsequently read status word has bit 8 set. |

These are established return patterns, not established error names. In particular, `0xFFFFFF13` does not mean that the transfer was skipped or rolled back.

### Preconditions for ordinary completion

The function does not establish the following conditions itself:

- The initial table access must be readable for the selector-derived index, even when the destination is null.
- If the dependency returns 1, the previously selected object must still support the subsequent source and status reads.
- If the dependency returns 1, the destination must support all sixteen 32-bit writes.
- Address calculations and accesses must satisfy the execution platform’s requirements.
- F-828f890d3c4d must return normally for execution to reach either the transfer or the return value 3.

There is no visible bounds check on the selector and no visible validity check on the selected object address.

## Behavior

### 1. Select and retain an object address

Before checking the destination, the function derives a signed index from the selector’s low 16 bits.

The index range is −32,768 through 32,767. Each table entry occupies `0x18C` bytes, or 396 bytes. The function reads an object-address-sized value at the beginning of the selected entry.

The mathematical displacement from the table base therefore ranges from −12,976,128 through 12,975,732 bytes. Those displacements fit within a signed 32-bit integer. Whether adding a displacement to the actual table base produces a valid address is a separate, unchecked condition.

Consequences include:

- Low 16-bit values `0x0000` through `0x7FFF` select nonnegative indices.
- Low 16-bit values `0x8000` through `0xFFFF` select negative indices.
- Selectors with identical low 16 bits select the same table entry, assuming the same table-base value.
- Such selectors need not produce the same overall behavior: F-828f890d3c4d receives their complete, potentially different, 32-bit values.

The selected object address is retained locally. The function does not visibly repeat the table lookup after calling the dependency.

### 2. Reject a null destination

After the object-address read, the function checks whether the destination is null.

For a null destination, it returns `0xFFFFFF0B` without calling F-828f890d3c4d and without performing the transfer or reading the object’s status word.

This is not a guarantee that a null-destination invocation safely returns. An invalid initial table access can prevent the function from reaching the null check.

### 3. Consult F-828f890d3c4d

For a non-null destination, the function calls F-828f890d3c4d once with the original selector.

Only a return value equal to 1 permits further object reads and destination writes by this function. Every other normally returned value produces return value 3.

There is no visible distinction among dependency results such as 0, 2, a negative value, or another non-1 bit pattern. None is forwarded to the caller.

If the dependency returns a non-1 value, this function performs no explicit destination writes. That does not establish that the destination or other memory remains unchanged: the dependency’s side effects are unknown.

### 4. Transfer sixteen words in ascending order

When the dependency returns 1, the function transfers sixteen 32-bit words from the retained object address to the destination.

| Property | Layout |
|---|---|
| First source word | Object offset `0x420` |
| Last source word | Object offset `0x45C` |
| Source extent | Object bytes `0x420` through `0x45F` |
| First destination word | Destination offset `0x00` |
| Last destination word | Destination offset `0x3C` |
| Destination extent | Destination bytes `0x00` through `0x3F` |
| Total transfer | 16 words, 64 bytes |

Each destination word corresponds to the source word at the same relative position within the transfer region. No field interpretation, conversion, masking, or byte-order transformation is visible.

The supplied representation specifies successive word assignments in increasing offset order: read the first source word and write the first destination word, then proceed to the next pair, through the sixteenth pair.

This ordering matters when memory overlaps:

- The function does not first collect all sixteen source values into a separate snapshot.
- An earlier destination write can change the value read by a later source read.
- The behavior is therefore not established as an overlap-safe block move.
- The transfer is not established as atomic or internally synchronized.

The table-selected address remains the source base even if F-828f890d3c4d changes the table entry or the global table-base state. Contents at that retained address are read only after the dependency returns, so changes the dependency makes to those contents can affect the result.

### 5. Read status after completing the transfer

After all sixteen destination writes, the function reads a 32-bit word at object offset `0x404`.

It tests mask `0x00000100`, corresponding to bit 8:

- If the masked bit is clear, it returns zero.
- If the masked bit is set, it returns `0xFFFFFF13`.

All other bits in that word are ignored by the visible decision.

The status word is read after the copy, not before it. If the destination overlaps that status word, the transfer can affect the value used to choose the return code. Likewise, changes made by the dependency can affect the status read.

No rollback, destination clearing, retry, or second dependency call is visible.

## Side effects

### Directly established effects

The function explicitly performs:

1. An initial table-based object-address read.
2. For a non-null destination, one call to F-828f890d3c4d.
3. If that call returns 1, sixteen source reads interleaved with sixteen destination writes.
4. After those writes, one status-word read from the retained object.

No explicit allocation, deallocation, ownership transfer, or persistent-state update is visible apart from writes through the destination pointer.

The object is not directly written through the retained object address. Nevertheless, destination aliasing may cause those writes to modify the object, a table entry, or other shared storage.

### Ownership and lifetime

The destination is caller-supplied storage. The function neither visibly allocates it nor explicitly releases it. It also does not visibly retain the destination pointer for later use.

These observations do not establish the dependency’s ownership behavior. F-828f890d3c4d may have side effects that are not described by the supplied evidence. In particular, its preservation of globals, object contents, and object lifetime cannot be assumed.

If that dependency invalidates the retained object address and then returns 1, this function still attempts the subsequent accesses through that address. No lifetime revalidation is visible.

### Failure boundaries

There is no explicit handling for inaccessible table entries, invalid object addresses, insufficient destination storage, unsuitable alignment, or failed memory accesses.

A failure during the transfer can leave a partially modified destination. The function provides no visible transaction or restoration mechanism. Exact platform behavior for such failures is unknown.

## Dependencies

### F-828f890d3c4d

**Invocation:** Once, only after a non-null destination has been established.

**Argument:** The complete original 32-bit selector, not the signed 16-bit table index.

**Result use:** Exact equality with 1 controls whether the transfer occurs. Every other normal result maps to return value 3.

**Unknowns:** Its validation rules, interpretation of the selector, state changes, synchronization, memory ownership, and object-lifetime effects are not supplied.

### Data dependencies

The function also relies on:

- Global state providing the base used for the table lookup.
- Entries spaced 396 bytes apart, each with an object address at its beginning.
- An object readable at offsets `0x404` and `0x420` through `0x45F` on the transfer path.
- Caller-provided writable destination storage on that path.

No broader table schema or object schema is established.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed test results.

### Null destination with a valid initial lookup

Given a selector whose initial table access is valid and a null destination:

- The object-address lookup occurs.
- F-828f890d3c4d is not called.
- No record transfer or object-status read occurs.
- The result is `0xFFFFFF0B`.

The same guaranteed return cannot be asserted if the initial lookup is invalid.

### Dependency declines the transfer

Given a non-null destination and a normally returning dependency result of 0:

- The initial object address has already been selected.
- The dependency receives the original selector.
- This function performs no record transfer and no object-status read.
- The result is 3.

The same branch applies to any dependency result other than exactly 1.

### Transfer with status clear

Given valid memory, no relevant overlap or interference, a dependency result of 1, and a status word whose bit 8 is clear when read:

- Destination word 0 receives the word at object offset `0x420`.
- Destination word 15 receives the word at object offset `0x45C`.
- All intervening words follow the same four-byte spacing.
- The result is zero.

### Transfer with status set

Under the same conditions, except that status bit 8 is set when read:

- All sixteen words are still transferred.
- The result is `0xFFFFFF13`.

A caller must not interpret this return as proof that the destination was untouched.

### Signed indexing and full-width dependency input

For selector `0x0000FFFF`, the table index is −1, so the entry begins 396 bytes before the table base.

Selector `0x1234FFFF` selects the same entry under unchanged table state. The dependency nevertheless receives a different full-width argument, so identical dependency behavior is not guaranteed.

### Dependency changes the table

Suppose the initial lookup selects object A, then the dependency changes the selected table entry to object B and returns 1.

Provided A remains valid, the transfer and status read still use A. The function does not reselect B.

### Forward-overlapping transfer

Suppose the destination begins four bytes after the first source word, the source initially contains distinct words, and no other interference occurs.

The first write replaces the second source word. The next source read therefore observes that replacement. This propagation continues through the forward transfer; the result is not a snapshot of the original sixteen source words.

## Uncertainties

### Inferred intent

The dependency result appears to act as an eligibility gate. The copied region appears to be a fixed-size record, and bit 8 appears to report an additional condition after retrieval. These are interpretations, not established domain meanings.

### Documented-only unknowns

The following remain documented_only:

- The domain meaning and valid range of selectors.
- Whether negative indices intentionally address valid table entries.
- Table allocation size and complete entry layout.
- Record field meanings and byte order.
- Domain names for the returned status patterns.
- Exact pointer, alignment, and address-overflow rules.
- Dependency side effects and ownership behavior.
- Concurrency guarantees and externally required locking.
- Platform consequences of invalid memory accesses.
- Whether overlapping source and destination storage is an intended supported use.

These unknowns do not change the visible branch conditions, offsets, transfer order, or return patterns.

## Evidence

The supplied static representation establishes the initial lookup before null checking, signed 16-bit index derivation, 396-byte entry spacing, full-width dependency argument, exact dependency-result comparison, sixteen ordered 32-bit assignments, final status-word read, and four return patterns.

The same representation supports the absence of visible local bounds checks, allocation, deallocation, revalidation, retry, and rollback. It does not establish the absence of such behavior inside F-828f890d3c4d or elsewhere in the surrounding system.

No runtime execution, memory inspection, dependency implementation review, or platform validation has been performed.
