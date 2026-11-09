<!-- Trusted provenance: F-f5548ddcf66a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f5548ddcf66a

## Purpose

F-f5548ddcf66a partially populates a caller-provided output structure using a source object, an opaque context, and dependency-produced data. It constructs a temporary record, performs an existence lookup, and requests insertion of that record when the lookup returns zero.

The established contract consists of field accesses, dependency calls, conditional writes, and bitwise updates. Domain meanings are unknown. Terms such as “identifier,” “category,” and “descriptor” describe operational roles rather than established application semantics.

This is not a complete output initializer. Some output bytes receive no direct writes. The word at output offset 24 retains its previous bits and may receive no direct access at all.

## Interface

The function accepts three inputs and has no explicit return value.

| Input | Representation | Role |
|---|---|---|
| Source | Address represented by a 32-bit value | Supplies fields through byte offset 131 |
| Output | Pointer to 32-bit words | Receives direct writes through byte offset 103 |
| Context | Opaque 32-bit value | Passed to lookup and insertion dependencies |

All offsets in this contract are byte offsets. Word accesses are 32 bits; byte accesses are 8 bits. Byte order is not established.

### Source layout directly consumed

| Offset | Width | Use |
|---:|---:|---|
| 92 | 32 bits | Flags, read at several separate stages |
| 100 | 32 bits | Argument to F-50bb1f23da9b |
| 104 | 32 bits | Identifier, independently reread for several operations |
| 120 | 32 bits | Selects a construction byte for three recognized values |
| 124 | 32 bits | Selects an output-offset-24 update |
| 128 | 32 bits | Selects a final output-offset-100 update |

An ordinary contiguous source allocation must provide at least 132 accessible bytes. This is an access bound, not an established complete object size.

### Output layout directly accessed

| Offset | Width | Behavior |
|---:|---:|---|
| 0 | 32 bits | Replaced with `0x1580` |
| 4 | 32 bits | Replaced with a selected descriptor value |
| 8 | 32 bits | Replaced with the second dependency-produced word |
| 12 | 32 bits | Replaced with the first dependency-produced word |
| 20 | 32 bits | Replaced with another descriptor value |
| 24 | 32 bits | Conditionally read, ORed, and written |
| 88 | 32 bits | Replaced with zero |
| 92 | 32 bits | Replaced with a fresh source-offset-104 read |
| 96 | 8 bits | Replaced with zero |
| 100 | 32 bits | Initialized from a global slot, then conditionally updated |

An ordinary contiguous output allocation must provide at least 104 accessible bytes. The byte write at offset 96 does not itself clear offsets 97–99.

Whenever output offset 24 is read, its existing 32-bit contents must be valid. This function does not initialize that word before reading it.

### Opaque global slots

The following labels identify global storage slots whose contents are used as 32-bit values:

- **Descriptor A** and **Descriptor B**: alternatives for output offset 4.
- **Descriptor C**: supplies output offset 20.
- **Lookup key A**: supplies the first lookup’s key.
- **Construction token**: supplies the first construction call’s second argument.
- **Lookup key B**: read separately for a saved insertion key and for the existence lookup.
- **Base flags A** and **Base flags B**: supply alternatives for output offset 100.

Their numeric values and domain meanings are unknown. Different slots need not contain different values. Calling them descriptors, keys, or base flags does not establish that their contents are immutable or preserved by dependencies.

## Behavior

The following sequence describes established direct operations and call boundaries. It does not include unknown effects inside dependencies. Where several expressions supply one call’s arguments, their relative evaluation order is not independently established unless stated below.

### 1. Initial calls and pair production

The function first calls F-304d68214672 without explicit arguments.

After that call returns, it reads source offset 100 and calls F-50bb1f23da9b with:

1. That 32-bit source value.
2. The address of temporary storage for two consecutive 32-bit words.
3. The 32-bit value `1`.

Both temporary words are consumed afterward. Ordinary defined completion therefore requires them to contain valid values when consumed; the dependency’s actual initialization contract is unavailable.

Neither call is established to preserve source storage, output storage, or global slots. Subsequent reads are not entry-time snapshots.

### 2. Initial output fields

After pair production, the function reads Descriptor A and then reads source flags at offset 92. If bit `0x40` is set, it reads Descriptor B and selects that value instead.

It then performs these writes in order:

1. Output offset 4 receives the selected descriptor.
2. Output offset 8 receives the second temporary word.
3. Output offset 12 receives the first temporary word.
4. Output offset 0 receives `0x1580`.
5. Output offset 20 receives the value read from Descriptor C.

The dependency-produced pair is thus placed in reversed word order in the output.

### 3. Accumulating flags at output offset 24

The function reads the 32-bit category at source offset 124.

| Category | Category-related action |
|---:|---|
| 3 | OR output offset 24 with `0x01000000` |
| 4 | OR output offset 24 with `0x02000000` |
| 5 | Do not access output offset 24 for the category |
| 6 | OR output offset 24 with `0x04000000` |
| Any other value | OR output offset 24 with `0x01000000` |

Each OR action reads the current output word and writes the resulting 32-bit value. It does not clear preexisting bits or normalize mutually exclusive category bits.

The function then rereads source flags at offset 92. If bit `0x20` is set, it performs a separate read-modify-write at output offset 24, ORing with `0x08000000`.

Category 5 combined with a clear `0x20` flag causes no direct read or write at output offset 24. For other categories, the category write occurs even if its mask adds no new bits.

The function next writes a 32-bit zero to output offset 88.

### 4. Selecting the construction byte

A 32-bit read at source offset 120 selects an 8-bit value:

| Mode | Selected byte |
|---:|---:|
| 1 | 0 |
| 2 | 2 |
| 3 | 1 |

For every other mode, the supplied representation establishes no assignment defining that byte on the path. Its value is **documented_only**. No default value, rejection behavior, clamping, or arithmetic mapping is established.

### 5. First lookup and record construction

The function calls F-5710a8628cba with:

1. Context.
2. The value read from Lookup key A.
3. A fresh 32-bit read of source offset 104.

From the returned address, it reads a 32-bit word at offset 4 and then an 8-bit byte at offset 8. It performs no intervening null check. Ordinary completion requires those reads to be valid.

It next captures another 32-bit source-offset-104 value and calls F-0c619ddcbcea with:

1. The address of a four-byte temporary scratch region.
2. The value read from Construction token.

The 32-bit return value is retained for the second construction call.

After the first construction call returns, the function writes the beginning of a separate 16-byte temporary record:

| Record offsets | Contents directly written |
|---|---|
| 0–3 | Identifier captured before the first construction call |
| 4–7 | Word previously read from the first lookup result |

It then calls F-0c619ddcbcea again with:

1. The address of record offsets 8–15, an eight-byte scratch region.
2. The first construction call’s return value.

The second return value is treated as an address. The function writes:

1. The previously captured lookup-result byte to returned-address offset 4.
2. The selected construction byte to returned-address offset 5.

The returned address is not established to equal the eight-byte scratch address. Consequently, these writes cannot unconditionally be described as writes to record offsets 12 and 13.

The visible operations also do not independently establish complete initialization of all 16 record bytes.

### 6. Separate key reads, existence lookup, and insertion

After both returned-address byte writes, the function reads Lookup key B and saves that 32-bit value for possible insertion.

It then performs a **separate read of the same global slot** to supply the second argument of an existence lookup. F-5710a8628cba receives:

1. Context.
2. The value from this later, independent Lookup key B read.
3. Another fresh 32-bit read of source offset 104.

The saved insertion-key read precedes the call-argument read. There is no explicit dependency call between those two reads, but no established global-stability guarantee permits collapsing them into one read. Their values must not be assumed equal. The relative evaluation order of the lookup’s key argument and identifier argument is not independently established.

If the existence lookup returns zero, the function calls F-ca0f66276073 with:

1. Context.
2. The **saved value from the earlier Lookup key B read**.
3. Another fresh 32-bit read of source offset 104.
4. The address of the 16-byte temporary record.
5. The 32-bit size value `0x10`.

Insertion does not reread Lookup key B. A mutation of that slot during the existence lookup therefore does not replace the already saved insertion argument.

If the existence lookup returns nonzero, insertion is skipped. Both construction calls and both returned-address byte writes have already occurred in either case.

The insertion result is not inspected. No retry or verification lookup follows it.

Identifiers used for the first lookup, record construction, existence lookup, insertion, and final output come from separate reads. Their equality is not guaranteed if dependencies, aliases, or other permitted mutations change the source.

### 7. Final output fields

After insertion returns, or immediately after a nonzero existence result, the function reads source offset 104 again and writes that value to output offset 92.

It then:

1. Reads Base flags A.
2. Writes one zero byte at output offset 96.
3. Writes the captured Base flags A value to output offset 100.

Next, it separately reads source flags at offset 92 for each of two conditions:

- If bit `0x02` is set on the first read, output offset 100 is replaced with the value read from Base flags B.
- If bit `0x10` is set on the second read, output offset 100 is read, ORed with `0x1000`, and written back.

Finally, a 32-bit read at source offset 128 selects:

| Final mode | Additional action at output offset 100 |
|---:|---|
| 7 | None |
| 8 | OR with `0x01` |
| 9 | OR with `0x02` |
| 10 | OR with `0x04` |
| Any other value | Return without another update |

Offset 100 is initialized before these OR operations. All masks and word operations are 32-bit. No mode-to-mask arithmetic, narrowing conversion, or range validation is shown.

## Side effects

Direct externally relevant side effects include output writes, writes through the second construction result, and dependency calls. Temporary storage is also written and passed to dependencies.

The function does not directly write the source. This does not establish source immutability: dependencies and aliasing may change later source reads.

Source/output overlap is not checked. With overlapping storage, earlier output writes can affect later source reads. For example, equal source and output bases cause the output-offset-92 write to affect the source-flags reads that follow it. Independent-field examples therefore require non-overlapping storage.

Ordinary completion requires valid addressed ranges and suitable access alignment. Address calculations must reach the described storage without invalid address wraparound. Invalid-pointer, misalignment, and address-overflow behavior are not established.

No explicit allocation, release, or ownership transfer is visible. The function does not explicitly release lookup or construction results, but their ownership remains unknown.

Insertion receives an address into temporary storage. That storage must not be assumed valid after the function returns. Whether F-ca0f66276073 copies synchronously, borrows only during the call, or retains the address is **documented_only**.

There is no explicit error result, diagnostic, rollback, or cleanup path. Earlier output writes precede potentially failing dependency operations and are not visibly undone if execution fails to complete.

## Dependencies

| Opaque ID | Observed use | Requirement for ordinary completion |
|---|---|---|
| F-304d68214672 | Initial call without explicit arguments | Returns normally |
| F-50bb1f23da9b | Receives a source word, pair-storage address, and `1` | Consumed pair words are valid afterward |
| F-5710a8628cba | First lookup and existence lookup | First result supports the specified reads; second result is tested against zero |
| F-0c619ddcbcea | Two chained calls using temporary scratch addresses | Second result supports byte writes at offsets 4 and 5 |
| F-ca0f66276073 | Conditional size-16 insertion request | Accepts the supplied record address and actual contents |

These are requirements imposed by the caller’s operations, not proven dependency guarantees. No dependency is assumed to preserve globals, preserve pointed-to data, initialize all scratch bytes, allocate storage, release storage, or transfer ownership.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Unless overridden, assume valid storage, non-overlapping source and output, normally returning dependencies, defined consumed temporary values, and no mutation of the particular values being illustrated.

1. **Pair placement and descriptor selection.**  
   If F-50bb1f23da9b supplies first word `0x11223344` and second word `0x55667788`, output offsets 12 and 8 receive those values respectively. The descriptor selection uses source flag `0x40` as observed after that call.

2. **Preserving existing category bits.**  
   With output offset 24 initially `0x02000005`, category 3, and the subsequent `0x20` flag test true, its resulting value is `0x0B000005`.

3. **Category 5 bypass.**  
   Category 5 and a clear `0x20` flag cause no direct access to output offset 24. A set flag instead causes one OR update with `0x08000000`.

4. **Fallback category.**  
   A category outside 3–6 follows the category-3 mask rule. It does not cause rejection or early return.

5. **Construction despite an existing record.**  
   A nonzero existence result skips F-ca0f66276073 but does not skip either construction call or either returned-address byte write.

6. **Distinct key observations.**  
   If the saved Lookup key B read yields `0x11111111` and the later argument read yields `0x22222222`, the existence lookup receives `0x22222222`. If it returns zero, insertion receives `0x11111111`. This example is conditional on such differing observations being possible; it does not claim an established mutation mechanism.

7. **Mutation during existence lookup.**  
   If both initial key reads yield the same value and the existence lookup changes the global slot before returning zero, insertion still receives the earlier saved value, not a new read of the changed slot.

8. **Absent-record insertion.**  
   A zero existence result causes one insertion call with size 16 and the saved insertion key. No insertion result is checked, and no retry follows.

9. **Final flag composition.**  
   If the respective final flag reads observe `0x02` and `0x10` set, and final mode is 9, output offset 100 becomes Base flags B OR `0x1000` OR `0x02`.

10. **Byte-width clearing.**  
    The direct zeroing at offset 96 affects only that byte. Preservation of offsets 97–99 additionally requires that dependencies and aliases do not modify them.

11. **Unsupported construction mode.**  
    For a source-offset-120 value outside 1–3, no particular selected byte is an established acceptance requirement.

## Uncertainties

The following remain **documented_only**:

- Numeric values, domain meanings, and stability guarantees of opaque global slots.
- Whether the two Lookup key B reads always observe equal values in valid executions.
- Dependency mutation, failure, initialization, ownership, and lifetime contracts.
- Whether the first lookup can return zero during valid usage.
- Whether the second construction result points into its supplied scratch region.
- Complete initialization of the insertion record, especially its final eight bytes.
- The construction byte for unsupported modes.
- Byte order, alignment requirements, and invalid-address behavior.
- Supported aliasing and concurrent-mutation models.
- Relative evaluation order of separate expressions within a call’s argument list.

An inferred intent is to produce descriptor-like output while ensuring an associated record exists. That interpretation does not establish successful insertion, deduplication, stable lookup namespaces, or ownership rules.

## Evidence

The supplied static representation establishes the explicit field widths and offsets, masks, branches, direct access sequence, dependency arguments, and lack of checked insertion results.

In particular, it distinguishes an earlier saved Lookup key B read from a later independent read used as the existence lookup argument. Only conditional insertion uses the saved value. Equality of the lookup and insertion key arguments requires an additional stability assumption.

Dependency implementations, numeric global-slot contents, complete object definitions, and runtime observations were not supplied. No runtime tests have been performed. The acceptance examples state conditional static consequences, not observed executions.
