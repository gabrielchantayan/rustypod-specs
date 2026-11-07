<!-- Trusted provenance: F-c70de8d2b40e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c70de8d2b40e

## Purpose

F-c70de8d2b40e performs a release-like operation on a nonzero input address. It first invokes a preparatory dependency, resolves an underlying address using metadata immediately before the input, and reduces a global accounting value. Except for one distinguished resolved address, it then validates a header, updates several indexed and aggregate accounting fields, and passes the header address to a final processing dependency.

The release interpretation is **inferred intent**, not an established ownership contract. The supplied static evidence establishes metadata access, accounting changes, and dependency calls. It does not establish which dependency releases storage, whether storage remains accessible afterward, or whether the input represents a uniquely owned allocation.

A zero input is an immediate no-op. An invalid header takes a nonreturning error path, but only after the preparatory call, global accounting update, and another dependency call have already occurred.

## Interface

### Input and result

The function accepts one value represented as a 32-bit integer and used as an address. It has no explicit return value.

The following names describe contract-level roles rather than implementation identifiers:

- **Input address:** the supplied 32-bit value.
- **Resolved address:** the input address minus the 32-bit displacement read four bytes before the input address.
- **Header address:** sixteen bytes before the resolved address.
- **Accounting object:** the object designated by one global pointer.
- **Management object:** the object obtained through a separate global pointer location after F-2fd8826557a4 returns.

Zero is treated specially before any dependency call or metadata access. The resolved address value `8` is a separate sentinel: it bypasses header validation and management-object updates, but not the earlier metadata reads and global accounting update.

### Relevant layouts

Offsets are in bytes. Fields need only be accessible when their corresponding path reaches them.

| Base | Offset | Width | Observed role |
|---|---:|---:|---|
| Input address | −4 | 32 bits | Displacement subtracted from the input address |
| Resolved address | −8 | 32 bits | Charge used in the accounting-object subtraction |
| Resolved address | −16 | 32 bits | Header word used for validation and masked size extraction |
| Resolved address | −12 | 16 bits, unsigned | Index for the first management accounting array |
| Resolved address | −10 | 16 bits, unsigned | Index for the second management accounting array |
| Accounting object | +0x38 | 32 bits | Low word of a two-word accounting value |
| Accounting object | +0x3c | 32 bits | High word of that value |
| Management object | +0x224 + 4 × first index | 32 bits | First indexed accounting field |
| Management object | +0x30c | 32 bits | First aggregate accounting field |
| Management object | +0xe0 + 4 × second index | 32 bits | Second indexed accounting field |
| Management object | +0x220 | 32 bits | Second aggregate accounting field |
| Management object | +0x310 + 4 × dependency result | 32 bits | Selected count field |
| Management object | +0x390 | 32 bits | Aggregate count field |
| Management object | +8 | 32 bits | Additional size-accounting field |

The two 16-bit indices are zero-extended before scaling by four. Their representable range is 0 through 65,535; the function does not visibly restrict them to the actual array capacities.

### Preconditions

For a defined successful execution, every reached metadata location, global pointer location, and accounting field must be accessible with the indicated width. Computed addresses must be valid for the target environment, and indices must select valid fields.

The function does not check these conditions. It also does not verify alignment, pointer provenance, ownership, prior release status, accounting sufficiency, or array bounds.

## Behavior

### 1. Zero-input exit and preparation

If the input address is zero, the function returns immediately. It performs no visible memory access or dependency call on this path.

Otherwise, it calls F-74b5a05d0094 with the original input address. Subsequent metadata and global-state reads occur after that call returns. Consequently, the contract does not assume that metadata or globals retain their values from function entry.

The function then captures the accounting-object pointer. A second zero-input check appears in the supplied representation, but there is no visible reassignment of the input between the two checks. Under ordinary by-value argument semantics, it adds no reachable case.

### 2. Address resolution and two-word accounting

The function reads the 32-bit displacement at input address minus four and subtracts it from the input address to obtain the resolved address.

It then reads the 32-bit charge at resolved address minus eight.

The low accounting word at offset `+0x38` is read and replaced by its previous value minus the charge, retaining the low 32 bits.

The high accounting word at offset `+0x3c` is then read and replaced using both:

- The sign of the charge interpreted as a signed 32-bit value.
- The unsigned borrow from the low-word subtraction.

More precisely, the high word is reduced by the sum of the charge’s arithmetic sign extension and the borrow indicator. The sign extension is zero when bit 31 is clear and negative one when bit 31 is set. The borrow indicator is one exactly when the old low word is unsigned-less-than the charge.

Under an ordinary stable, nonoverlapping layout, this is equivalent to subtracting the sign-extended 32-bit charge from a 64-bit value whose low and high words are at `+0x38` and `+0x3c`, with the result retained modulo 2^64. It is **not** universally equivalent to subtracting an unsigned 32-bit charge: charges with bit 31 set produce different high-word behavior.

The low-word store precedes the high-word store. No underflow or overflow rejection is visible.

The supplied representation uses both a captured accounting-object pointer and further reads of its global pointer location during this stage. No dependency call intervenes. A stable global pointer is therefore a prerequisite for treating these accesses as one coherent two-word object; atomicity under concurrent mutation is not established.

### 3. Intermediate dependency and management-object selection

After both accounting stores, the function calls F-2fd8826557a4 with no visible arguments.

Only after that call returns does it obtain the management-object pointer from its global pointer location. Changes made by this dependency can therefore affect which management object is selected.

If the resolved address is `8`, the function now returns normally. This sentinel does not bypass the preceding charge read or accounting updates. In particular, the charge-read address for this sentinel is numerically zero; whether that address is usable depends on the execution environment.

### 4. Header validation

For every resolved address other than `8`, the function examines the 32-bit word at the header address.

It takes the nonreturning error path through F-a817d8d58c10 when:

- The header word is zero; or
- The header word has bit 0 set.

The visible condition performs a zero test first. If that read is nonzero, it reads the header again to test bit 0. Thus, these are not necessarily tests of one captured value if another actor can modify the header.

No rollback precedes the error call. The earlier accounting-object changes remain observable unless a dependency independently reverses them.

The supplied evidence marks F-a817d8d58c10 as nonreturning. The mechanism—termination, exception-like transfer, trap, or another outcome—is not established.

### 5. Management size accounting

After successful validation, the function calls F-c0fc6d033caa with the selected management-object address.

It then:

1. Reads the unsigned 16-bit first index at resolved address minus twelve.
2. Reads the header word again and extracts a size using mask `0x3ffffffc`.
3. Subtracts that size from the first indexed accounting field.
4. Subtracts the same captured size from the first aggregate field at `+0x30c`.
5. Reads the unsigned 16-bit second index at resolved address minus ten.
6. Subtracts the same captured size from the second indexed accounting field.
7. Subtracts the same captured size from the second aggregate field at `+0x220`.

Each accounting update reads the current 32-bit field and stores a 32-bit subtraction result. No sufficiency check, saturation, or explicit error branch is visible.

The mask clears bits 0, 1, 30, and 31. The extracted size is therefore a multiple of four from zero through `0x3ffffffc`. Passing validation does not imply a nonzero extracted size: a nonzero header with bit 0 clear can have all its set bits outside the mask.

The header used for size extraction is read after F-c0fc6d033caa returns. It is not necessarily the header value that passed validation. No second validation occurs after that dependency.

### 6. Count accounting and final size accounting

The function next calls F-09dcb55e8e0a with no visible arguments. Its result selects a 32-bit count field at management-object offset `0x310 + 4 × result`.

The function then performs these updates in order:

1. Decrements the selected count field by one.
2. Decrements the aggregate count at `+0x390` by one.
3. Reads the header word again, masks it with `0x3ffffffc`, and subtracts that newly obtained size from the field at `+8`.

The last subtraction does not necessarily use the size captured for the earlier four size-accounting updates. In particular, F-09dcb55e8e0a occurs between the two size reads, and its effects are unknown.

There is no visible check that the selector result is nonnegative, within bounds, or safe to scale into an address.

### 7. Final dependency calls

The function calls F-cc6c9b4a0f68 with:

- The selected management-object address.
- The header address.

After that call returns, it calls F-e586bff64fb7 with the same management-object address, then returns normally if that dependency returns.

The management-object address is retained across these calls rather than reloaded from the global pointer location. The function does not itself visibly write the header word.

## Side effects

Established direct writes are limited to the accounting fields described above. Additional effects may occur inside dependencies and are not bounded by the supplied evidence.

Observable ordering includes:

- F-74b5a05d0094 precedes address-resolution metadata reads.
- Accounting-object updates precede F-2fd8826557a4.
- F-2fd8826557a4 precedes management-object selection and sentinel handling.
- Header validation precedes F-c0fc6d033caa.
- Four management size-accounting writes precede F-09dcb55e8e0a.
- Count updates and the final size update precede F-cc6c9b4a0f68.
- F-cc6c9b4a0f68 precedes F-e586bff64fb7.

These operations are not visibly transactional. A dependency that fails to return, an invalid memory access, or an arithmetic-address failure can leave earlier effects in place.

For overlapping fields, each update operates on the value present when its read occurs. The accounting examples below assume nonoverlapping storage; the implementation does not visibly enforce that layout.

Ownership transfer and memory lifetime remain unknown. In particular, passing the header address to F-cc6c9b4a0f68 does not by itself prove that this dependency releases, retains, or takes ownership of the storage.

## Dependencies

| Opaque function ID | Visible interface at this call site | Established position |
|---|---|---|
| F-74b5a05d0094 | Original input address; no result used | First action for nonzero input |
| F-2fd8826557a4 | No visible arguments; no result used | After accounting-object updates |
| F-a817d8d58c10 | No visible arguments; nonreturning | Zero-header or odd-header error path |
| F-c0fc6d033caa | Management-object address; no result used | After header validation, before management writes |
| F-09dcb55e8e0a | No visible arguments; integer result used as an index | Between size-accounting and count-accounting updates |
| F-cc6c9b4a0f68 | Management-object address and header address; no result used | After all direct management writes |
| F-e586bff64fb7 | Management-object address; no result used | Final visible call on the ordinary path |

A synchronization role for the calls surrounding management updates is plausible but unproven. No dependency may be assumed to preserve globals, preserve header contents, acquire or release a lock, or manage storage ownership without additional evidence.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Unless stated otherwise, dependencies return normally, preserve the relevant fields, and all addresses and indices are valid and nonoverlapping.

### Zero input

Given an input of zero, the function returns without dependency calls, metadata reads, or accounting writes.

### Ordinary nonzero release-like path

Suppose address resolution produces a value other than `8`, the charge is 12, and the accounting words initially contain low 100 and high 7.

The accounting words become low 88 and high 7.

Suppose the validated header is `0x22`, both index fields select valid entries, and the selector dependency returns a valid count index. The masked size is 32, not 34. The four earlier management size fields and the final field at `+8` each decrease by 32; the selected and aggregate count fields each decrease by one.

The dependency sequence is F-74b5a05d0094, F-2fd8826557a4, F-c0fc6d033caa, F-09dcb55e8e0a, F-cc6c9b4a0f68, then F-e586bff64fb7.

### Borrow across the accounting words

With low word 5, high word 3, and charge 9, the stored low word is `0xfffffffc` and the high word is 2.

### Charge with its sign bit set

With low word 0, high word 10, and charge `0xffffffff`, the stored low word is 1 and the high word remains 10.

This matches subtraction of signed negative one from the combined value, rather than subtraction of unsigned 4,294,967,295.

### Sentinel resolved address

If resolution produces `8` and all preceding accesses are valid, the function still updates the accounting object and calls F-2fd8826557a4. It obtains the management-object pointer but performs no header validation or management-object updates and calls none of the later dependencies.

### Invalid header

With a stable header of zero or `0x21`, the function reaches F-a817d8d58c10 after the earlier accounting changes. It does not reach F-c0fc6d033caa or the ordinary management writes.

### Valid header with zero masked size

A stable header of `0x40000000` is nonzero and has bit 0 clear, so it passes validation. Its masked size is zero. The size subtractions leave their fields unchanged, while both count fields still decrement.

### Header change during a dependency

If the header supplies masked size 32 for the earlier size updates, but F-09dcb55e8e0a changes it to a value with masked size 64, the first four size fields decrease by 32 and the final field at `+8` decreases by 64. The function does not reconcile those values.

## Uncertainties

The following remain **documented_only**:

- The public calling convention, pointer provenance rules, and authoritative object types.
- The meaning and permitted range of the displacement.
- The identities, initialization requirements, and synchronization rules of the global objects.
- Actual capacities of the indexed accounting regions.
- The valid range and semantic meaning of the selector dependency’s result.
- Dependency behavior, including global mutation, failure modes, synchronization, and memory ownership.
- Whether repeated calls for the same input are permitted.
- Whether accounting wraparound is intentional.
- Target-specific consequences of overflowing address calculations, invalid addresses, misalignment, or concurrent access.

The supplied operations establish field widths and arithmetic structure, but not a portable language-level guarantee for every out-of-range address calculation. No byte-order requirement beyond the target’s native interpretation of these fields is established.

## Evidence

The contract derives from the supplied static function representation and the supplied opaque dependency mapping. Established behavior includes the visible branches, access widths, offsets, masks, arithmetic, call arguments, and ordering.

Allocation-release terminology, combined-counter interpretation, and possible synchronization roles are identified as inference where applicable. Dependency internals and runtime validity are not established.

No runtime tests have been performed.
