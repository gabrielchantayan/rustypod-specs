<!-- Trusted provenance: F-c173faf04cca; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c173faf04cca

## Purpose

F-c173faf04cca transforms an encoded input sequence, initializes object-related storage using a nonzero context value, writes a status byte, and conditionally publishes an address through a caller-provided output slot.

The status decision is established only by the visible dependency invocation: F-fca51b715561 receives the result of F-44362605ab79, the address of a locally transformed reference, and the integer 10. A zero return causes status 4; a nonzero return causes status 0. The supplied body does not establish that F-fca51b715561 compares bytes or interprets 10 as a byte count.

The input and a fixed twelve-byte reference undergo the same four-byte permutation and XOR transformation. The reference contents, object semantics, and status meanings are unavailable.

Explicit rejection paths cover missing required arguments, an unsuitable input extent, and failure of the temporary allocation. Later pointer results are used without equivalent checks.

This contract describes static evidence. No runtime tests have been performed.

## Interface

### Arguments

| Argument | Visible representation | Contract |
|---|---|---|
| Context value | 32 bits | Must be nonzero to enter processing. Passed unchanged as the second argument to F-398e0e9a318a. Whether it represents an address, handle, or other value is unknown. |
| Encoded input address | 32-bit address representation | Must be nonzero. Passed to F-1cb585febd84 and subsequently used for byte reads over the returned extent. |
| Output-slot address | Address of a four-byte slot | Must be nonzero. Receives one 32-bit address value on the ordinary publication branch. |
| Opaque value | 32 bits | Initially copied into local storage. No subsequent dependency argument, branch, or published value explicitly uses it. |

A nonzero address is not sufficient to establish valid or suitably aligned memory.

There is no explicit length argument. F-1cb585febd84 determines the extent; its method and memory requirements are **documented_only**. The extent must not be assumed to be a string length.

### Return value

The visible result is a 32-bit integer:

- A global fallback value on explicit early rejection.
- Zero after ordinary publication, provided the remaining calls return normally.
- Negative 108 when the retained interior object address minus four is zero, again provided cleanup calls return normally.

The fallback value is unknown and may equal either zero or negative 108. Return value alone therefore does not necessarily distinguish all paths.

### Object-related addresses and layout

Three addresses must remain distinct:

1. The result of the allocation-like dependency called with 16.
2. The **interior object address**, returned by F-398e0e9a318a.
3. The **publication address**, four bytes before the interior object address.

The routine does not establish that the interior object address equals the first allocation result plus four. Consequently, it does not establish that the publication address equals that allocation result.

Direct object-related writes are:

| Base | Offset | Width | Value |
|---|---:|---:|---|
| Result of the call requesting 16 | 0 | 1 byte | Zero |
| Interior object address | 0 | 4 bytes | Global metadata value read at that stage |
| Interior object address | 8 | 1 byte | 4 or 0 according to F-fca51b715561’s return |

Other layout, initialization, alignment, and storage requirements are **documented_only**. The four-byte writes use the target’s memory representation; byte order is not established here.

## Behavior

### Argument gating and fallback selection

The routine first captures the global fallback value.

If any of the context value, encoded input address, or output-slot address is zero, it reads that global again and returns the later reading. It calls none of the listed dependencies and performs no direct output-slot write on this path. The fourth argument is not validated.

With all three required arguments nonzero, the routine copies all four arguments into a sixteen-byte local area before requesting the input extent. These copies are not subsequently used as ordinary functional inputs. Later reference construction overwrites the first thirteen bytes of that area.

### Extent determination and temporary allocation

F-1cb585febd84 receives the encoded input address. Its result is treated as an unsigned 32-bit extent.

Only extents divisible by four are accepted. A nonmultiple of four returns the fallback captured before this dependency call, not a fresh global reading. A change to the fallback global inside the dependency therefore does not change that selected result.

For an accepted extent, F-d33f6d597e5d receives the extent plus one. A zero result returns the same initially captured fallback without entering object processing.

Zero is an accepted extent. The largest accepted unsigned extent is 4,294,967,292, so adding one does not overflow a 32-bit unsigned value. These arithmetic facts do not guarantee that the requested allocation or subsequent addresses are usable.

For a positive accepted extent, the loop advances by four until its offset equals the extent. The terminal offset remains representable, including at the largest accepted extent.

### Input transformation

The temporary allocation receives one transformed byte for every input byte, followed by a zero terminator.

Each four-byte group uses this mapping:

| Destination offset within group | Source offset within group | XOR value |
|---:|---:|---:|
| 0 | 2 | 0xCA |
| 1 | 0 | 0xA6 |
| 2 | 3 | 0x01 |
| 3 | 1 | 0x4A |

Groups are processed in increasing input-offset order. Within each group, each source read is followed by its corresponding destination write before the next source read. Source-read order is therefore 2, 0, 3, 1; destination-write order is 0, 1, 2, 3.

That interleaving matters if source and destination storage overlap. The routine does not stage an entire group before writing it and performs no overlap check.

After all groups, it writes a zero byte at temporary-buffer offset equal to the extent. With extent zero, this terminator is the only direct write made by this transformation stage, and the transformation performs no input-byte reads. F-1cb585febd84 may already have read the input independently.

Transformed bytes may contain zero before the appended terminator. There is no visible validation or filtering of transformed values, and no established rule for how later dependencies interpret embedded zeros.

### Object-related initialization

F-8157a180d901 receives 16. Its result is immediately used for a one-byte zero write, without a null check.

F-398e0e9a318a then receives:

- The preceding result plus four bytes.
- The original context value.

Its result is retained as the interior object address. It is not checked before subsequent writes or dependency calls.

After that call, the routine captures a global reference-data address. It then reads a global four-byte metadata value and writes it at interior-object offset zero.

These globals are read at this stage, after the preceding dependencies. Their values must not be assumed to equal values that existed at function entry.

### Local reference construction

The routine transforms twelve bytes beginning at the captured reference-data address into the local area using the same mapping as the input transformation.

Exactly three groups are processed, with the same interleaved read/write ordering. A zero byte is then written at local offset twelve.

The resulting local representation is twelve transformed bytes followed by a terminator. The referenced bytes were not supplied, so the transformed contents are unknown.

The reference-data address is captured before the metadata write, but reference-byte reads occur afterward. If the metadata destination aliases the reference data, that write can affect the reference bytes subsequently read. Other aliases may also affect the transformation; disjointness is not checked.

### Dependency-driven status selection

The following calls occur in order:

1. F-756609d5a5c5 receives the interior object address and temporary-buffer address. Its return value is ignored.
2. F-44362605ab79 receives the retained interior object address.
3. F-fca51b715561 receives the preceding result, the local reference address, and the integer 10.

The routine tests only whether the final return is zero:

- Zero causes a one-byte write of 4 at interior-object offset eight.
- Any nonzero value causes a one-byte write of 0 at that offset.

No byte-comparison semantics, string semantics, ordering relation, or interpretation of the integer 10 is established. In particular, transforming twelve reference bytes and passing 10 does not establish that ten reference bytes are examined.

The interior object address is retained across these calls rather than obtained again afterward. Dependencies may change the pointed-to contents; the final status write still targets the retained address plus eight.

### Local clearing, publication, and final calls

After the status write, the routine clears local offsets zero through eleven using three four-byte zero writes in increasing offset order.

It does not perform a full sixteen-byte clear. Offset twelve was previously assigned the reference terminator, but a dependency received the local reference address before clearing; preservation of that terminator or other local bytes is not established.

The routine next forms the publication address by subtracting four bytes from the retained interior object address:

- If the result is zero, it selects negative 108 and does not directly write the output slot.
- Otherwise, it writes the publication address into the four-byte output slot and selects zero.

This is a late address test, not an allocation-failure check. Object-related writes and dependency calls have already occurred.

After either branch, a two-argument external dependency receives the temporary-buffer address and original extent. F-5e3309c9d083 then receives the temporary-buffer address.

The explicit length passed to the first final dependency excludes the appended terminator. Its actual memory effects are unknown. Both final calls occur after publication or selection of the negative-108 branch. The selected result is returned if they return normally.

## Side effects

### Direct accesses and ordering

On a fully traversed processing path, direct accesses include:

- Reading the fallback global at entry.
- Writing argument copies into local storage.
- Reading the input extent’s bytes during transformation.
- Writing the transformed temporary data and appended terminator.
- Writing one zero byte through the result of the call requesting 16.
- Reading the reference-address global and metadata global.
- Writing four metadata bytes at the interior object address.
- Reading twelve reference bytes and writing their transformed local representation.
- Writing one status byte at interior-object offset eight.
- Clearing twelve local bytes.
- Writing a four-byte publication address on the nonzero-address branch.

The routine does not directly write through the encoded input address. This is not a guarantee that input storage remains unchanged: aliases or dependency behavior may modify it.

Likewise, “no direct output-slot write” does not mean dependencies cannot modify that storage. The slot’s address is not passed explicitly to them, but it may be reachable through aliases or other state.

Publication precedes both final temporary-buffer calls. If the output slot aliases temporary storage, those calls may affect the published slot after its direct write. No final output verification occurs.

### Ownership and lifetime

The temporary address is passed to F-5e3309c9d083 at the end of the processing path. A release role is plausible but not established.

No visible direct operation releases the allocation requested with 16 or the published object. Ownership transfer, destruction rules, and the relationship between those storage regions are **documented_only**.

F-756609d5a5c5 receives temporary storage before the final calls. Whether it copies, borrows, retains, consumes, or releases that storage is unknown. The routine does not prove that later object use is independent of temporary-buffer lifetime.

### Failure boundaries

There is no visible recovery for inaccessible memory, invalid pointer results, unchecked allocation failure, unsuitable alignment, dependency exceptions, or dependencies that do not return normally.

The fallback and negative-108 results are not exhaustive failure reporting. A zero result from the allocation-like call requesting 16 would encounter a write through that result before the late publication test.

## Dependencies

| Opaque ID | Established use |
|---|---|
| F-1cb585febd84 | Receives the encoded input address. Returns the value treated as the unsigned 32-bit extent. |
| F-d33f6d597e5d | Receives extent plus one. Zero is explicitly rejected; a nonzero result becomes temporary writable storage. |
| F-8157a180d901 | Receives 16. Its result is immediately written through without checking. |
| F-398e0e9a318a | Receives the preceding result plus four and the context value. Returns the retained interior object address. |
| F-756609d5a5c5 | Receives the interior object address and temporary-buffer address. Return ignored. |
| F-44362605ab79 | Receives the interior object address. Its result becomes the first argument to F-fca51b715561. |
| F-fca51b715561 | Receives the preceding result, local reference address, and integer 10. Its zero/nonzero return determines the status byte. |
| F-5e3309c9d083 | Receives the temporary-buffer address after the two-argument external dependency. |

One external dependency has no opaque ID in the supplied mapping. Its established invocation is the temporary-buffer address followed by the original extent. Its semantics remain **documented_only**; neither erasure nor release is assumed.

No dependency is assumed to preserve globals, preserve pointed-to storage, or follow an ownership convention.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Missing required argument.**  
   If any required argument is zero, no listed dependency is called and no direct output-slot write occurs. The returned fallback is the second global reading, not necessarily the entry reading.

2. **Unsuitable extent and fallback mutation.**  
   If F-1cb585febd84 returns 6, no temporary allocation is requested. The returned fallback is the entry capture, even if that dependency changed the global.

3. **Checked allocation failure.**  
   For extent 8, F-d33f6d597e5d receives 9. If it returns zero, the routine returns the entry fallback without object processing or a direct publication write.

4. **Four zero decoded bytes.**  
   For extent 4 and source bytes A6, 4A, CA, 01 in increasing address order, the transformation writes four zero bytes and then its appended zero, provided earlier writes do not alter later source reads through aliasing. Interpretation by F-756609d5a5c5 remains unknown.

5. **Zero extent.**  
   A successful temporary allocation is requested with 1. The transformation reads no input bytes, writes one zero terminator, and continues through object processing. The final two-argument dependency receives extent zero.

6. **Zero classification result.**  
   If F-44362605ab79 returns an arbitrary value, that value is passed unchanged as the first argument to F-fca51b715561, alongside the local reference address and 10. If the latter returns zero, the direct status write is 4. This conclusion does not require or establish byte equality.

7. **Nonzero classification result.**  
   If F-fca51b715561 returns negative 1, the direct status write is 0. Other nonzero results select the same status. This classification does not itself change the ordinary zero return.

8. **Ordinary publication.**  
   If the retained interior object address is 0x1004, the publication write stores 0x1000. Assuming all accesses and calls complete normally, the final result is zero and both final temporary-buffer calls occur after that write.

9. **Late zero-address branch.**  
   If the retained interior object address is numerically 4, publication subtraction produces zero. The routine selects negative 108, omits the direct output-slot write, and still makes both final temporary-buffer calls. Earlier accesses at and relative to address 4 must already have succeeded.

## Uncertainties

The following remain **documented_only**:

- Global fallback value, metadata contents, and reference contents.
- Extent-determination semantics and its memory-access requirements.
- Complete object layout, alignment, allocation guarantees, and address relationships.
- F-fca51b715561’s operation, interpretation of 10, and meaning of zero.
- Semantic meanings of status 4 and status 0.
- Dependency mutation of globals or referenced storage.
- Temporary-buffer retention and all ownership or destruction rules.
- Whether either final dependency erases or releases memory.
- Target-specific address validity and consequences of extreme address arithmetic.
- Exceptional or non-returning dependency behavior.

Decoding data, constructing an object, and using a reference to classify it are inferred intent. A bytewise comparison or reference match is not established behavior.

## Evidence

The supplied static body establishes argument gating, fallback-read timing, unsigned extent handling, allocation requests, byte transformation, direct access widths and offsets, dependency invocation order, status selection, and publication before the final temporary-buffer calls.

The supplied dependency mapping establishes the opaque IDs used here. It does not supply dependency implementations or contracts.

No global data contents, runtime traces, or executed tests were supplied. All acceptance examples are conditional consequences of the visible operations, not runtime observations.
