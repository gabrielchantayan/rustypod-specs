<!-- Trusted provenance: F-e20da8d344de; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e20da8d344de

## Purpose

F-e20da8d344de constructs a descriptor, derives two address-like values using a selected table entry, links the descriptor into a caller-provided head slot, and returns the descriptor’s second word.

There are two construction modes:

- A zero-valued mode argument obtains the primary value directly from F-638beded7699.
- Any nonzero mode argument constructs an auxiliary descriptor, obtains backing storage through F-638beded7699, and derives an aligned, potentially tagged value from that result.

The function performs no visible argument validation, allocation-failure recovery, rollback, or release. A zero result from F-638beded7699 does not prevent publication of the main descriptor.

“Descriptor,” “backing storage,” “head slot,” and “tagged” describe the apparent roles of the data. The field contents and access ordering below are established by the supplied static evidence; the broader resource model is not.

## Interface

### Inputs

The function accepts four arguments:

| Argument | Role |
|---|---|
| Head-slot address | Address of a writable word holding the current list head or equivalent link value. The old value is copied into the new descriptor before this slot is replaced. |
| Requested quantity | An integer used in arithmetic passed to F-638beded7699. In nonzero mode, it is also stored in the auxiliary descriptor and used as an unsigned loop bound. Its unit is not established. |
| Table index | Selects an entry consisting of an offset word and a mask word, with an eight-byte entry stride. No bounds check is visible. |
| Mode | A full integer whose equality to zero selects the construction path. Its low eight bits are also stored in the main descriptor. |

No supplied evidence establishes valid index bounds, a maximum requested quantity, or a supported subset of nonzero mode values.

### Width and representation

The visible layout uses four-byte words and one-byte status fields. Pointers are stored in word-sized integer fields. This supports a 32-bit address and integer interpretation of the supplied representation; portability to a different data model is not established.

Byte offsets in this document are relative to the beginning of the relevant descriptor. Endianness is not established. The one-byte stores affect only the addressed byte, not the entire containing word.

### Main descriptor

A 24-byte allocation is requested for the main descriptor.

| Byte offset | Width | Established role |
|---:|---:|---|
| 0 | 4 bytes | Primary value from the selected construction path. |
| 4 | 4 bytes | Offset-adjusted, masked value; ordinarily the returned value. |
| 8 | 4 bytes | Requested quantity plus the selected offset, computed before the mode-specific work. |
| 12 | 1 byte | Low eight bits of the mode argument. |
| 13–15 | 3 bytes | Not explicitly assigned after the external initialization call. |
| 16 | 4 bytes | Auxiliary-descriptor address in nonzero mode; otherwise not explicitly assigned after initialization. |
| 20 | 4 bytes | Previous value read from the caller’s head slot. |

An external routine receives the allocation and the length 24 before these fields are populated. Its semantics are unknown. Consequently, fields not explicitly assigned cannot be specified as zero.

### Auxiliary descriptor

Nonzero mode requests an additional 16-byte allocation.

| Byte offset | Width | Established role |
|---:|---:|---|
| 0 | 4 bytes | Raw result from F-638beded7699, initially zero. |
| 4 | 4 bytes | Aligned value, possibly modified by a high-bit tag, initially zero. |
| 8 | 4 bytes | Original requested quantity. |
| 12 | 1 byte | Status byte: initially zero; set to one after the nonzero-result path completes its loop. |
| 13–15 | 3 bytes | No visible initialization. |

### Output

Under ordinary, nonaliasing execution, the function returns the word at main-descriptor offset 4:

**(primary value + selected offset) bitwise-AND selected mask**

The final return reads that field after publishing the new head. If the caller’s head slot aliases descriptor storage, publication can change what is returned. Nonaliasing is therefore a precondition for the ordinary output interpretation.

## Behavior

### 1. Main allocation and table selection

The function first requests 24 bytes from F-8157a180d901, then passes the returned address and length 24 to an external routine whose behavior is not supplied.

There is no visible check for allocation failure before initialization or subsequent writes.

After that call, the function reads the global table-base value and retains that base locally. It then:

1. Writes the low byte of the mode argument at main-descriptor offset 12.
2. Reads the selected offset word at table base plus eight times the table index.
3. Adds that offset to the requested quantity.
4. Writes the sum at main-descriptor offset 8.

The offset is retained for later use. The mask word is not read yet.

This timing matters: the initial external call occurs before the table base is read, whereas later dependency calls occur after the base and offset have been captured.

### 2. Zero-mode construction

When the full mode argument equals zero, the function calls F-638beded7699 with:

- The already computed requested-quantity-plus-offset value.
- The constant 55.

The result becomes the primary value.

No explicit auxiliary allocation occurs on this path. The main descriptor’s auxiliary-pointer field receives no explicit store after the external initialization call.

A zero result is not treated as an error. Construction continues.

### 3. Nonzero-mode construction

When the full mode argument is nonzero, the function requests 16 bytes from F-8157a180d901.

It explicitly initializes:

- The raw-result word to zero.
- The aligned-value word to zero.
- The quantity word to the original requested quantity.
- The status byte to zero.

It then calls F-638beded7699 with the requested quantity plus 64 as its first argument and 55 as its second argument. This request does not use the selected table offset.

The returned value is written into the auxiliary descriptor’s raw-result word.

#### Zero raw result

If the result equals zero:

- No alignment calculation is performed.
- F-20810a394e6a is not called.
- The loop is skipped.
- The status byte remains zero, absent interference outside the visible function.
- The initially zero aligned-value field supplies the primary value.

The auxiliary descriptor is still attached to the main descriptor, and the main descriptor is still published.

#### Nonzero raw result

For a nonzero result, the function forms a 32-bit unsigned intermediate by adding 31, then clears its low five bits. The resulting value is stored at auxiliary-descriptor offset 4.

This is a 32-byte alignment operation within 32-bit modular arithmetic. It is not an unbounded mathematical ceiling operation: addition can wrap, and a nonzero raw result can produce an aligned value of zero.

The function next tests bit `0x08000000` of the unsigned intermediate before low-bit clearing. This is bit 27, not the top bit. Clearing the low five bits does not change the tested bit.

If that bit is set, the function calls F-20810a394e6a with:

- The aligned value.
- The quantity currently read from auxiliary-descriptor offset 8.

After that call returns, the function rereads the aligned-value field and sets bit `0x80000000` in that field. It does not simply store the original aligned value with that bit added. Any change made to this field by the dependency can therefore affect the stored result.

Whether or not the conditional dependency call occurs, the function then executes an empty-body loop. Its counter begins at zero and is compared as an unsigned 32-bit value against the quantity field interpreted as unsigned.

There is no visible payload copy, initialization, or other body action in this loop. With a stable bound of zero, it performs no iterations. With a stable negative signed quantity, the unsigned interpretation can require billions of iterations. No time bound is established.

After the loop finishes, the function writes one to the auxiliary status byte.

### 4. Main-field completion and publication

In nonzero mode, after the auxiliary work, the function:

1. Stores the auxiliary-descriptor address at main-descriptor offset 16.
2. Reads the auxiliary aligned-value field to obtain the primary value.

Both modes then perform these operations in order:

1. Store the primary value at main-descriptor offset 0.
2. Add the retained table offset to that value.
3. Read the selected mask word using the previously captured table base and index.
4. Store the sum bitwise-AND the mask at main-descriptor offset 4.
5. Read the caller’s current head-slot word.
6. Store that old head value at main-descriptor offset 20.
7. Replace the caller’s head-slot word with the main-descriptor address.
8. Read main-descriptor offset 4 and return it.

The table offset and mask are not an atomic snapshot. The offset is read before mode-specific dependency calls; the mask is read afterward. A dependency that changes the table contents can therefore influence the mask used for completion.

A dependency that changes the global table-base variable does not, by itself, redirect the later mask read: the function uses its previously captured base. Releasing or invalidating that captured table storage would still make the later access unsafe.

Similarly, the old head is read late. If a dependency changes the caller’s head slot before publication, the value linked into the new descriptor is the value present at that late read, not necessarily the value present on entry.

## Side effects

### Memory and state

Visible effects include:

- One 24-byte allocation request on every path.
- One additional 16-byte allocation request in nonzero mode.
- An external initialization call on the main allocation.
- Explicit descriptor stores described above.
- One F-638beded7699 call.
- A conditional F-20810a394e6a call.
- Replacement of the caller’s head-slot value.
- Execution of the unsigned-bound empty loop on the nonzero-result auxiliary path.

The main descriptor becomes reachable through the caller’s head slot under normal nonaliasing conditions. In nonzero mode, the auxiliary descriptor becomes reachable through the main descriptor.

### Ownership

No visible release occurs for either descriptor or for the value returned by F-638beded7699.

The source does not establish:

- Whether either allocation transfers ownership to the caller.
- Whether F-638beded7699 returns owned storage, a borrowed address, or another encoded value.
- Whether F-20810a394e6a takes ownership, releases anything, or changes unrelated state.
- Which later operation should reclaim these objects.

Reachability is established; ownership transfer is not.

### Errors and preconditions

There is no explicit error return or validation path.

Successful ordinary execution requires, at minimum:

- Valid writable descriptor storage from the allocation dependency.
- A valid writable head slot.
- Accessible selected table words.
- Arithmetic and address calculations valid for the actual execution environment.
- Dependency behavior that leaves storage needed by later accesses valid.

A zero primary value is not an error sentinel for this function. The masked return may be nonzero even when the dependency result is zero.

No rollback is visible if a dependency fails to return normally or a memory access fails.

## Dependencies

| Opaque ID | Established use | Unknown behavior |
|---|---|---|
| F-8157a180d901 | Called with 24, and additionally with 16 in nonzero mode; results are used as descriptor addresses. | Allocation guarantees, alignment, initialization, failure convention, ownership, and global side effects. |
| F-638beded7699 | Called once with a mode-dependent first argument and constant 55; its result drives primary-value construction. | Meaning of arguments, allocation behavior, validity of returned values, ownership, and side effects. |
| F-20810a394e6a | Conditionally receives an aligned value and the auxiliary quantity when bit 27 of the alignment intermediate is set. Its return value is unused. | Transformation, ownership effects, memory writes, and global changes. |

One additional external routine receives the main-descriptor address and length 24 immediately after allocation. No opaque function ID or implementation is supplied for it. Its behavior is **documented_only**; zero-filling must not be assumed.

No unknown dependency is assumed to preserve globals or to own or release memory.

## Acceptance examples

These are conditional static examples, not executed tests. They assume valid disjoint storage, stable relevant fields except for specified dependency results, and ordinary 32-bit execution.

### Zero mode with a nonzero result

Given:

- Requested quantity 32.
- Selected offset 16.
- Selected mask `0xfffffff0`.
- Mode zero.
- F-638beded7699 returns `0x1003`.

Expected:

- Its first argument is 48; its second is 55.
- Main offset 8 contains 48.
- Main offset 0 contains `0x1003`.
- Main offset 4 contains `0x1010`.
- The previous head is linked at offset 20.
- The head slot receives the main-descriptor address.
- The return is `0x1010`.

### Zero mode with a zero result

With the same quantity, offset, and mask, but a zero dependency result:

- Publication still occurs.
- Main offset 0 contains zero.
- Main offset 4 and the ordinary return are 16.

Thus zero from F-638beded7699 does not imply zero from this function.

### Nonzero mode without tagging

Given quantity 3, offset 16, mask `0xfffffff0`, and raw result `0x1001`:

- F-638beded7699 receives 67 and 55.
- The aligned value is `0x1020`.
- F-20810a394e6a is not called.
- With a stable quantity field, the loop performs three iterations.
- Auxiliary status becomes one.
- The main primary value is `0x1020`.
- The ordinary return is `0x1030`.

### Nonzero mode with tagging

Given quantity zero and raw result `0x08000001`:

- The aligned value is `0x08000020`.
- F-20810a394e6a is called with that value and zero.
- If it leaves the aligned-value field unchanged, the subsequent tag store produces `0x88000020`.
- The loop performs no iterations.
- Auxiliary status becomes one.

### Full mode versus stored byte

A mode value of 256 selects the nonzero path but stores zero in the main descriptor’s mode byte. The byte does not independently identify which path was taken.

### Alignment wraparound

A raw result of `0xffffffff` produces an unsigned intermediate of 30 and an aligned value of zero. It still follows the nonzero-result path, including the loop and final status write.

## Uncertainties

- **documented_only — Arithmetic overflow:** The alignment addition and loop comparison explicitly expose unsigned behavior. Other additions and index scaling appear in signed integer form. Their overflow semantics are not established by this source representation; examples avoid those cases.
- **documented_only — Initializer semantics:** The external initialization call cannot be specified as clearing the descriptor.
- **documented_only — Resource meaning:** Units, allocation domain, masks, tag meaning, and lifecycle remain unknown.
- **documented_only — Concurrency:** No synchronization is visible. This contract does not promise atomic publication, race safety, or a consistent table snapshot.
- **documented_only — Aliasing:** No restrictions are checked. Aliasing the head slot with descriptor fields or other accessed storage can alter later reads and the return.
- **Inferred intent:** The layout resembles linked resource metadata with an optional auxiliary allocation record. That interpretation is not required to explain the established behavior.

## Evidence

This contract is based solely on the supplied static function representation and dependency-ID mapping.

Established evidence includes explicit call arguments, allocation request sizes, field widths and offsets, branch conditions, bit operations, read/write order, publication order, and the final field read.

Dependency implementations, the initialization routine’s semantics, runtime memory mappings, ownership conventions, and a complete target arithmetic specification were not supplied. No runtime tests have been performed.
