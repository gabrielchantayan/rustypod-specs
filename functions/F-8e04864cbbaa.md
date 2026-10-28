<!-- Trusted provenance: F-8e04864cbbaa; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8e04864cbbaa

## Purpose

F-8e04864cbbaa copies two 32-bit words from an input record into a destination object and derives one byte of destination state.

The first input word has two distinct roles:

- Its original value is copied unchanged.
- With bit 0 cleared, it supplies an address used to obtain a byte at offset `0x4C`, unless the resulting address is zero.

Bit 0 of a later read of the first input word controls whether bit 2 is set in the derived byte.

**Inferred intent:** the input resembles a two-word descriptor whose first word combines an address with a one-bit tag. The destination stores the descriptor and a cached attribute byte. This interpretation does not establish object types, ownership rules, or meanings for the tag and attribute bits.

## Interface

The function accepts two arguments and has no return value.

| Argument | Required interpretation |
|---|---|
| Destination base | Address of writable storage containing a 32-bit field at offset `0x20`, another at offset `0x24`, and a byte field at offset `0x28`. |
| Input record | Address of two readable, contiguous 32-bit words at offsets `0` and `4`. The first word is read again later. |

For this contract, a byte is 8 bits and a word is 32 bits. These widths are supported by the recovered types and the four-byte field spacing. The address-bearing word is also 32 bits; the evidence does not support treating it as a wider host pointer.

The required destination footprint extends through offset `0x28`, inclusive. The shown behavior does not require every preceding byte of that footprint to be readable or writable; it requires the three specified fields to support their respective accesses.

When the first input word, with bit 0 cleared, is nonzero, one additional byte must be readable at that resulting address plus `0x4C`.

Successful execution requires:

- Valid storage for every access actually taken.
- Any alignment required by the execution environment for the 32-bit accesses.
- A valid interpretation of the 32-bit address-bearing word.
- Address calculations that produce usable locations in that environment.

No null check protects either argument. Zero has special handling only for the address obtained by clearing bit 0 of the first input word.

The evidence does not establish a particular calling convention, byte order, memory-protection model, or concurrency contract.

## Behavior

### Input capture and destination updates

The function first reads the input word at offset `0`, then the input word at offset `4`. Both reads occur before either destination word is written.

It then writes:

1. The captured first word to destination offset `0x20`.
2. The captured second word to destination offset `0x24`.

Both copies retain all 32 bits. In particular, bit 0 of the first word remains present in the destination copy.

The metadata lookup uses the **captured first word**, with only bit 0 cleared. Bits 1 through 31 are unchanged. This is not a general alignment operation, a range check, or validation that the result identifies an object.

If the cleared value is zero, the initial derived byte is zero and no metadata-byte access occurs. Consequently, captured first-word values `0` and `1` both suppress the metadata lookup.

If the cleared value is nonzero, the initial derived byte is read from that address plus `0x4C`. This access occurs after both destination word writes.

### Flag derivation

After choosing the initial derived byte, the function reads the first input word again. It tests bit 0 of this later value.

- If bit 0 is clear, the derived byte is unchanged.
- If bit 0 is set, bit 2 of the derived byte is set.

The operation does not clear bit 2 when the tag is absent. A bit already present in the metadata byte remains present.

The final byte is written to destination offset `0x28`. This is the last explicit memory operation shown. The function then returns without a value.

### Ordering and aliasing

The static operation order is:

1. Read both input words.
2. Write both captured words to the destination.
3. Conditionally read the metadata byte.
4. Reread the first input word.
5. Write the derived byte.

This ordering matters when storage overlaps.

The metadata address always comes from the initially captured first word. The tag decision instead comes from the later read. Those values need not agree if a destination write changes the input record or another permitted mechanism changes the storage.

Likewise, metadata storage can overlap a destination word. In that case, the metadata read observes the state after the two word writes, subject to the target’s memory semantics.

The final byte write can overlap input storage or metadata storage. Such a write occurs after all explicit reads and therefore does not retroactively change the current invocation’s result.

These are recovered operation-order statements, not a guarantee of inter-thread synchronization, atomicity, compiler barriers, or device-memory access behavior.

### Arithmetic boundaries

Clearing bit 0 produces an even 32-bit value but does not establish four-byte alignment. The metadata access itself is only one byte wide.

The evidence contains address additions for destination offsets `0x20`, `0x24`, and `0x28`, and for metadata offset `0x4C`. It provides no explicit overflow detection. Behavior where an address addition exceeds the usable address range is **documented_only**; no portable wraparound or successful access is promised.

## Side effects

On a valid execution, the explicit writes are exactly:

| Destination offset | Width | Value |
|---|---:|---|
| `0x20` | 32 bits | Initially captured first input word |
| `0x24` | 32 bits | Initially captured second input word |
| `0x28` | 8 bits | Derived metadata/tag byte |

The function does not explicitly initialize, clear, or copy any other destination fields.

The input record is not written through its input argument. It can nevertheless be modified through destination writes when the regions overlap.

No allocation, release, reference-count adjustment, callback, or ownership transfer is shown. Copying the two words does not establish that the destination acquires ownership of any referenced storage.

No explicit global access appears. This does not prove that the addressed storage is unrelated to global state.

There is no error return, status field, recovery branch, or rollback mechanism. If a later access fails under the target environment, the earlier destination writes have already been reached in the recovered sequence. Transactional behavior is not established.

## Dependencies

No external function dependency is present. The function makes no calls, so there are no dependency IDs to enumerate.

Its behavior depends on the execution environment for:

- The representation and usability of 32-bit addresses.
- Byte order and word-access alignment requirements.
- The effects of overlapping memory accesses.
- Invalid-access handling and address-overflow behavior.
- Any concurrent or device-driven changes to addressed storage.

No assumption about another function preserving globals or managing memory is needed or justified.

## Acceptance examples

Unless stated otherwise, these examples assume valid, nonoverlapping storage, stable input, and valid address arithmetic. They are conditional acceptance criteria, not observed test results.

### Zero first word

Given first word `0` and second word `0x12345678`:

- Destination `0x20` receives `0`.
- Destination `0x24` receives `0x12345678`.
- No metadata byte is read.
- Destination `0x28` receives `0`.

### Tag-only first word

Given first word `1` and any second word:

- Destination `0x20` receives `1`, not `0`.
- The second word is copied unchanged.
- No metadata byte is read because clearing bit 0 yields zero.
- Destination `0x28` receives `0x04`.

### Untagged address

Given a nonzero even address in the first word, with byte `0xA1` at that address plus `0x4C`:

- The original address word and second word are copied.
- The derived byte is `0xA1`.

The absent tag does not otherwise normalize the metadata byte.

### Tagged address

Given a first word whose low bit is set and whose remaining bits identify a nonzero address, with metadata byte `0xA1`:

- The destination retains the tagged first word.
- The metadata lookup uses the address with bit 0 cleared.
- The derived byte is `0xA5`.

If the metadata byte is already `0xA5`, the result remains `0xA5`.

### Existing bit 2 without a tag

Given an untagged nonzero address and metadata byte `0x04`, the derived byte remains `0x04`. An implementation that clears this bit when the tag is absent would not match the evidence.

### Input overlapping destination

Let the input record begin at destination offset `0x24`. Initially, its first word is `1` and its second word is `0`.

The function captures those two values, writes `1` to destination `0x20`, then writes `0` to destination `0x24`. That second write changes the first input word before its reread.

The metadata lookup is suppressed because the captured first word was `1`. The later tag test sees `0`, so the final byte at destination `0x28` is `0`, not `0x04`.

## Uncertainties

The following remain **documented_only**:

- Semantic types and meanings of the two copied words and derived bits.
- Whether overlapping arguments are intended, despite their statically derivable effects.
- Target byte order, alignment rules, and exact pointer representation.
- Fault behavior for invalid, unmapped, or unwritable storage.
- Address-overflow semantics.
- Atomicity, synchronization, and behavior under concurrent modification.
- Any surrounding ownership or lifecycle protocol.

The descriptor-and-tag interpretation is inferred intent, not an additional behavioral requirement.

## Evidence

Established static evidence consists of two initial 32-bit input reads, two ordered 32-bit destination writes, a conditional byte read through a bit-cleared address, a later reread of the first input word, and one final byte write. No calls or explicit error-handling paths are shown.

The contract distinguishes the initially captured address-bearing value from the later tag-bearing value because the recovered behavior reads them at different times.

No runtime tests have been performed. All acceptance examples are deductions from the supplied static evidence, subject to their stated preconditions.
