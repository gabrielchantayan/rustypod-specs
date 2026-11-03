<!-- Trusted provenance: F-90790bfc6bdf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-90790bfc6bdf

## Purpose

F-90790bfc6bdf requests storage, conditionally initializes selected locations, calls a dependency with the resulting address, and returns that address. When the request returns null, it instead calls a different dependency with an associated 32-bit value and returns null.

The established behavior is sparse initialization. The function writes a one-byte marker, the associated value, and two copies of a sampled pair of 32-bit words. It does not directly initialize the entire requested region.

A construction or registration role is plausible inferred intent, not an established contract. The evidence does not establish the object type, marker meaning, ownership transfer, or continued validity of the returned address.

## Interface

### Inputs

| Input | Established representation | Direct use |
|---|---|---|
| Context reference | Address of a readable 32-bit word | Reads that word before requesting storage |
| Associated value | Opaque 32-bit value | Passes it to the null-path dependency or writes it into non-null storage |
| Pair reference | Address of two consecutive 32-bit words | Reads both words on the non-null path |

The associated value is treated as a 32-bit value without an established signedness or semantic type. This function does not dereference it or perform arithmetic on it.

The context reference must support a 32-bit read before the storage request. There is no explicit validation of that reference.

The pair reference is accessed only after the storage request returns non-null and after the marker and associated-value writes. Successful completion of that path requires both pair words to be readable at that time. The function does not itself require the pair reference to be readable on the null-result path, because it does not access it there. Dependency-specific requirements remain unknown.

No explicit non-aliasing check appears. Aliasing can change which pair values are sampled, so the ordering requirements below are part of the contract.

### Output

The output is a pointer:

- If F-f053f140662a returns null, the function returns null after F-e5380ae57ea1 returns normally.
- If F-f053f140662a returns non-null, the function returns that same address after initialization and normal return from F-895c599e0ca7.

No separate status is returned. No result from either branch-specific dependency is inspected.

Returning an address does not establish that it still designates live storage, that the caller owns it, or that its initialized contents remain intact. Those guarantees depend on the unknown behavior of F-895c599e0ca7.

### Widths and destination layout

The storage request supplies the numeric size `0x44`, equivalent to 68 bytes. The supplied body does not independently establish the dependency’s size-parameter width.

The explicit memory accesses use one-byte and 32-bit widths. The following offsets are byte offsets from the non-null address returned by F-f053f140662a:

| Offset | Write width | Written value |
|---|---:|---|
| `0x00` | 1 byte | `0x94` |
| `0x10` | 4 bytes | Associated value |
| `0x14` | 4 bytes | First sampled pair word |
| `0x18` | 4 bytes | Second sampled pair word |
| `0x1c` | 4 bytes | First sampled pair word again |
| `0x20` | 4 bytes | Second sampled pair word again |

These six writes cover 21 distinct destination bytes. The highest directly written byte is at offset `0x23`, decimal 35.

Within the requested 68-byte region, the function directly leaves offsets `0x01`–`0x0f` and `0x24`–`0x43` unwritten. This does not guarantee that their contents are unchanged across the whole call: dependencies may initialize or modify them.

The marker write affects one byte only. It does not establish a four-byte marker field or clear adjacent bytes.

## Behavior

### Storage request and branch selection

Before calling any dependency, the function reads the 32-bit word designated by the context reference. It then calls F-f053f140662a with:

- The sampled context word.
- The size value 68.

There is one explicit context read and one request. No retry is present.

The context reference is not reread after the request. Changes to that memory during the dependency call do not change the argument already supplied.

The pair has not yet been read. Changes made to the pair during F-f053f140662a can therefore affect the later copied values.

The sole explicit branch condition is whether the returned address is null. There is no additional success, capacity, alignment, or validity check.

### Null-result path

If the request returns null, the function calls F-e5380ae57ea1 with the associated 32-bit value. If that dependency returns normally, the function returns null.

On this path, the function:

- Does not read either pair word.
- Does not perform destination initialization.
- Does not call F-895c599e0ca7.
- Does not inspect a result from F-e5380ae57ea1.

The call does not by itself establish cleanup, resource release, error reporting, or consumption of the associated value.

### Non-null-result path

After a non-null request result, the observable direct accesses occur in this order:

1. Write marker byte `0x94` at destination offset `0x00`.
2. Write the associated 32-bit value at destination offset `0x10`.
3. Read the first 32-bit word through the pair reference.
4. Read the next 32-bit word, four bytes after the first.
5. Write the sampled first word at destination offset `0x14`.
6. Write the sampled second word at destination offset `0x18`.
7. Write the sampled first word at destination offset `0x1c`.
8. Write the sampled second word at destination offset `0x20`.

The function then calls F-895c599e0ca7 with the destination address. If that call returns normally, the function returns the same address.

Both pair words are sampled before any of the four pair-copy writes. Each sampled value is reused for both destination copies. The function does not reread the input pair to produce the second copy and does not read back the first destination copy.

Absent interference or an access failure, immediately before the final dependency call:

- The words at offsets `0x14` and `0x1c` are equal.
- The words at offsets `0x18` and `0x20` are equal.

These are initialization-point guarantees. They are not unconditional guarantees about memory after F-895c599e0ca7 returns.

### Aliasing and sampling

Entry-time pair contents are not necessarily the values copied.

The marker and associated-value writes precede both pair reads. If either write overlaps the pair input, it can change the sampled values even when F-f053f140662a leaves the pair untouched.

For example, if the pair begins at destination offset `0x10`, its first sampled word is the associated value just written there, absent intervening modification. Its second word is sampled from destination offset `0x14` before the pair-copy writes begin.

Once both reads have completed, subsequent destination writes cannot change the values already sampled. Thus, overlap between the input pair and a pair-copy destination does not cause later copies to resample overwritten input.

The body supplies no synchronization mechanism. The pair is obtained by two separate 32-bit reads, not by an established atomic eight-byte snapshot. Concurrent interference may prevent the two words from representing one coherent external state.

### Arithmetic and memory requirements

The function uses fixed destination offsets and a fixed size request. It performs no data-dependent size calculation, loop, or arithmetic transformation of the associated value or pair words.

For successful non-null completion, destination address calculations must designate valid writable locations through offset `0x23`. The pair must support the two specified reads, and any platform alignment requirements must be satisfied. Requirements imposed by F-895c599e0ca7 may be stronger.

Pointer width, address-overflow behavior, byte order, and alignment rules are not established by the supplied evidence. The 32-bit copies describe word values, not a portable byte-serialization format.

## Side effects

### Direct memory and control effects

The function directly performs one 32-bit context read and, on the non-null path, two 32-bit pair reads and six destination writes.

It calls F-f053f140662a once. If that call returns normally, exactly one branch-specific dependency is subsequently called:

- F-e5380ae57ea1 for a null result.
- F-895c599e0ca7 for a non-null result, after all explicit destination writes.

Neither input reference is directly used as a write destination. Nevertheless, destination writes can modify memory reachable through either reference if those regions alias.

### Indirect effects and ownership

No preservation of globals, input memory, resource state, or storage lifetime may be assumed for any dependency.

There is no separate explicit storage-release operation in this function. That observation does not exclude release inside a dependency.

The evidence does not establish:

- Whether F-f053f140662a allocates fresh storage or returns existing storage.
- Whether F-e5380ae57ea1 releases or consumes anything represented by the associated value.
- Whether F-895c599e0ca7 retains, publishes, mutates, transfers, or releases the supplied storage.
- Whether the caller acquires any ownership obligation.

### Errors and abnormal completion

A null request result is the only explicitly handled failure condition.

There is no visible recovery for invalid references, inaccessible memory, insufficient storage, misalignment, or abnormal dependency completion. If an access or dependency prevents normal continuation, later operations and the ordinary return are not guaranteed to occur.

Destination writes precede F-895c599e0ca7. No rollback is shown if that dependency fails to return normally.

## Dependencies

| Opaque function ID | Arguments supplied here | Established role | Unresolved contract |
|---|---|---|---|
| F-f053f140662a | Sampled 32-bit context word; numeric size 68 | Returns the address whose nullness selects the path | Storage mechanism, capacity, initialization, alignment, ownership, and other side effects |
| F-e5380ae57ea1 | Associated 32-bit value | Called following a null request result | Resource handling, reporting, global effects, and normal-return guarantees |
| F-895c599e0ca7 | Non-null destination address | Called after the explicit initialization | Mutation, publication, ownership, storage lifetime, and other side effects |

The function does not inspect a status from either branch-specific dependency. A dependency’s internal success or failure classification does not select another visible path here. Abnormal completion can still prevent the ordinary return.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests. Each assumes valid required accesses, normal dependency completion where stated, and no interference except the effects explicitly described.

### Ordinary non-null result without early-write overlap

Suppose the associated value is `0x12345678`, and the pair contains `0x11223344` followed by `0xaabbccdd`. F-f053f140662a returns suitable non-null storage and does not alter the pair.

Additionally, neither the marker destination byte nor the four-byte associated-value destination overlaps any byte of the pair input, and the pair remains unchanged until both reads complete.

Immediately before F-895c599e0ca7:

- Offset `0x00` contains byte `0x94`.
- Offset `0x10` contains word `0x12345678`.
- Offsets `0x14` and `0x1c` contain word `0x11223344`.
- Offsets `0x18` and `0x20` contain word `0xaabbccdd`.

Normal return from F-895c599e0ca7 leads to return of the supplied storage address. Retention of those field values afterward requires an additional preservation guarantee.

### Pair overlaps the associated-value destination

Suppose the pair begins at destination offset `0x10`. Before initialization, its words are `0x11223344` and `0xaabbccdd`; the associated value is `0x12345678`. The request dependency does not change those words.

The associated-value write nevertheless replaces the first pair word before it is read. The sampled words are therefore `0x12345678` and `0xaabbccdd`, absent interference.

Before the final dependency call, offsets `0x14` and `0x1c` contain `0x12345678`, while offsets `0x18` and `0x20` contain `0xaabbccdd`. Requiring the original first pair word in this case would contradict the access order.

### Null result with an unreadable pair reference

Given a readable context word, suppose the request returns null. The function makes no direct access through the pair reference, even if that reference is null or unreadable.

It calls F-e5380ae57ea1 with the associated value and returns null if that dependency returns normally. This example makes no claim about memory accessed independently by the dependency.

### Pair changed during the request

Suppose F-f053f140662a changes the pair and returns suitable storage. The copied values come from the subsequent two reads, after the marker and associated-value writes. Entry-time values are not promised.

### Final dependency changes initialized storage

Suppose F-895c599e0ca7 changes the marker and returns normally. F-90790bfc6bdf still returns the originally supplied address; it does not restore or recheck the marker.

If that dependency instead ends the storage lifetime, normal return still does not establish that the returned address is safe to dereference.

## Uncertainties

The following remain **documented_only**:

- Object type and meaning of marker `0x94`.
- Semantic roles and signedness of the opaque 32-bit values.
- Pointer width, byte order, alignment requirements, and address-overflow rules.
- Dependency parameter widths not established by explicit memory accesses.
- Whether the requested 68 bytes are fully provisioned or initialized.
- Intended contents and use of directly unwritten regions.
- Dependency effects on globals, memory, ownership, and resource lifetime.
- Whether callers prohibit aliasing or concurrent access.
- Whether the two copied pairs represent initial/current, saved/current, or another relationship.
- Whether the final dependency preserves any initialized field.

None of these unknowns is resolved by the apparent construction pattern.

## Evidence

The supplied static body establishes the request value, null-result branch, explicit access widths and offsets, read/write ordering, dependency arguments, and returned address.

The aliasing criteria follow from that ordering: the marker and associated-value writes occur before pair sampling, while all pair-copy writes occur after both reads. Accordingly, an unchanged pair during the request alone is insufficient to guarantee preservation of its entry-time values.

Construction, registration, cleanup, and saved-state interpretations remain inferred intent only. Dependency implementations and caller contracts were not supplied.

No runtime tests have been performed. All acceptance examples are conditional specifications supported by static evidence, not claims of observed execution.
