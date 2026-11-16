<!-- Trusted provenance: F-d7b3baad3c44; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d7b3baad3c44

## Purpose

F-d7b3baad3c44 performs an ordered sequence of unresolved indirect invocations, derives integer values from their results, mutates memory using mixed-width accesses, and finishes with a four-argument indirect invocation.

The supplied static representation does not establish an application-level purpose. Address-like payloads do not establish pointers, records, or containers. Numeric constants that also represent printable characters do not establish text output.

This contract distinguishes:

- **Established static evidence:** represented accesses, invocation operands, widths, retained values, and ordering relationships.
- **Conditional consequences:** behavior requiring successful dependency returns, accessible memory, a stated arithmetic model, and specified aliasing conditions.
- **documented_only:** unresolved behavior for which no concrete acceptance rule is established.

No runtime tests have been performed.

## Interface

### Inputs and outputs

The represented interface has no explicit parameters and no defined return value. Implicit inputs include incoming control context, dependency results, secondary result channels, fixed storage, and memory reached through computed addresses.

Observable outputs include indirect invocations, their represented arguments, intermediate memory accesses, and final memory contents. Effects performed inside dependencies are additional unknown outputs.

A return is represented after the terminal invocation. Reaching it requires every preceding operation to complete and the terminal dependency to return.

### Widths and arithmetic

| Quantity | Represented treatment |
|---|---|
| Byte access | 8 bits |
| Halfword store | 16 bits |
| Word access and ordinary integer value | 32 bits |
| Paired dependency result | 64 bits |
| Computed address | Derived from represented 32-bit values |
| Byte widened into a word payload | Zero-extension to 32 bits |
| Value narrowed into a byte payload | Low eight bits |

Conditional arithmetic examples use a **32-bit bit-vector model**: additions, subtractions, multiplications, and left shifts retain the low 32 bits. Intermediate truncation is significant. This model does not establish portable semantics for overflowing signed-looking expressions.

Endianness, alignment requirements, memory permissions, and ordinary-memory versus device-memory behavior are not established.

### Preconditions and errors

Every reached invocation target must support execution in the applicable calling context. Every reached memory location must support the represented access width. The function performs no represented validation of those conditions.

Direct low-address accesses are part of the behavior; an environment that rejects every such access cannot complete the represented path.

There is no local error result, bounds check, recovery branch, rollback, or cleanup protocol. Invalid-access and invalid-target outcomes are documented_only. Completion and termination are not guaranteed.

## Behavior

### Dependency prefix

The prefix contains a long, straight-line sequence of unresolved indirect invocations. The initial target depends on incoming control context. Later prefix targets are derived from embedded control-location values with a displacement of 4094 bytes.

Invocations expose zero through four represented arguments. Inputs include 32-bit results, portions of 64-bit results, secondary 32-bit result channels, zeros, masked values, and shifted values. An invocation with no represented arguments may still consume implicit machine state.

Transformations include fixed left shifts through count 31, logical right shifts of explicitly unsigned values, and extraction from unsigned 64-bit results using right shifts by 34, 36, and 44.

Widths and transformation order constrain the results:

- Under the conditional bit-vector model, a word shifted left 28 and subsequently left 28 becomes zero; a widened 56-bit shift is not equivalent.
- Retaining 24 low bits before shifting left six produces at most 0x3FFFFFC0.
- Retaining 22 low bits before shifting left eight produces at most 0x3FFFFF00.
- Explicitly zeroed retained values remain zero through subsequent shifts under this model.

Invocation order remains significant even when an argument is zero or a result appears unused. Unknown dependencies may modify globals or memory subsequently consumed by this function.

### Descriptive values

These labels identify value relationships, not recovered types or source identifiers.

| Label | Meaning |
|---|---|
| Shifted handle | Retained dependency-result word shifted left 25 |
| Scaled base | Another retained result multiplied by 2²⁵ |
| Small index | Bits 6–14 of a 32-bit value already shifted left eight |
| Small base | Twice the small index |
| Final pair | Last represented 64-bit result before explicit stores |
| Payload word | Retained dependency result shifted left eight |
| Byte-derived base | Zero-extended byte read from absolute address 0x05 |
| Link seed | Word captured at byte-derived base plus 0x54 |
| Original fourfold base | Four times the link seed |
| Adjusted seed | Original fourfold base minus 150 |
| Adjusted base | Four times the adjusted seed |
| Adjusted eightfold base | Eight times the adjusted seed |
| Linking base | Word captured from fixed storage before the absolute 0x60 store |
| Late seed | Word obtained through the later byte-based offset-0x74 lookup |
| Late doubled base | Twice the late seed |
| Late fourfold base | Four times the late seed |

Under the conditional arithmetic model, the small index ranges from 0 to 508 in multiples of four. The small base ranges from 0 to 1016 in multiples of eight. Neither range establishes address validity.

### Initial mutations and captures

The first four explicit stores have this order: a word containing the shifted handle at the final pair’s upper word plus 0x14; a byte containing that upper word’s low byte at shifted handle plus 9; a word containing the scaled base at 256 times the final pair’s low word plus 0x54; and another byte at shifted handle plus 9, now containing the shifted handle’s low byte.

The final byte payload is zero under the conditional bit-vector model. The earlier byte store nevertheless remains part of the access trace.

The scaled-base offset-0x14 word is cleared before its offset-5 byte is captured. A clear of small base plus 0x14 intervenes before the captured byte is written, zero-extended, as a word at scaled base plus 0x34.

The function then reloads scaled base plus 0x14, clears small base plus 0x14 again, and writes the reloaded word’s low byte twice at small base plus 0x15. The reload is not universally replaceable with zero: preceding computed-address accesses may overlap the cleared word.

After the absolute 0x05 byte capture, absolute byte address 0x01 receives the small base’s low byte. Byte-derived base plus 0x15 receives the small index’s low byte. Scaled base plus 0x11 receives the payload word’s low byte and is subsequently overwritten with zero; a word store of the payload word at the reloaded word plus 0x54 lies between those stores.

The link seed is captured from byte-derived base plus 0x54 before that field receives the reloaded word. Small base plus 0x34 first receives the small index as a word.

### Linking before the constant-store region

Offset-0x74 word lookups feed stores at loaded addresses plus 0x54. One represented assignment contains two reads of small base plus 0x74; immutable storage or a single captured value is not established for those reads.

The region also contains repeated word stores of the link seed at link seed plus 0x24, repeated byte stores through an offset-0x74-derived base plus 9, and several overwrites of small base plus 0x14. Absolute word address 0x54 receives twice the link seed. Absolute byte address 0x15 receives the small base’s low byte.

Destination bases and payloads must remain distinct:

- Original fourfold base plus 0x54 first receives the payload word and later receives address-like values equal to the original fourfold base.
- **Small base plus 0x34 receives the original fourfold base as a 32-bit word payload.**
- That offset-0x34 store does **not** establish a destination at original fourfold base plus 0x34.
- Original fourfold base plus 0x14 is a separate word-store destination. Its payload changes during the region, including the original fourfold base and eight times the link seed.

Other accesses involve the original fourfold base’s offsets 4, 5, 9, 0x0D, 0x15, and 0x64, plus further loaded bases. Byte values read at offset 5 are used numerically as bases for word reads at offset 0x74.

Two offset-5 byte reads through another loaded base occur on opposite sides of intervening mutations. The first supplies a zero-extended word store through another destination; the second feeds the lookup producing the late seed. They are distinct reads and need not agree.

### Constant stores and adjustment

The late-seed-derived destinations receive numeric 0x3A at three widths: a byte at four times the late seed, a halfword at eight times the late seed, and a word at late fourfold base plus 0x20.

The original fourfold base is reduced by 150 to form the adjusted seed. At adjusted base plus 0x20, the ordered word payloads are 0x20, 0x2F, a zero-extended byte captured from adjusted base plus 4, and 0x20 again. The byte capture occurs after the 0x2F store and before its byte-derived word store.

### Fixed-address write and repeated linking

After the last adjusted-base offset-0x20 store, the function captures the linking base from fixed word storage. **The next represented memory access is a 32-bit store of 0x00000020 at absolute address 0x60.**

The first linking group then performs four ordered stores:

1. A word containing the late fourfold base at eight times the link seed plus 0x54.
2. A byte containing the adjusted base’s low byte at linking base plus 0x19.
3. A byte containing the linking base’s low byte at adjusted base plus 0x11.
4. A word containing eight times the link seed at linking base plus 0x34.

Later groups use adjusted eightfold base for the offset-0x54 destination base and the linking-base offset-0x34 word payload. Their reciprocal byte stores retain the same destinations and captured low-byte payloads.

This is a finite, explicitly represented repetition, not a result-controlled retry. Exact multiplicity is not enumerated in this contract and remains documented_only here. Repeated stores must not be collapsed when aliasing, access observation, or device semantics can distinguish them.

The linking base is retained from one fixed-storage capture; the groups do not represent repeated loads of that storage.

### Post-repetition ordering

The first two captures are a word from absolute address 0xA9 and a word from late fourfold base plus 0x74. Only then does absolute byte address 0x3A receive the low byte of adjusted eightfold base plus 0x30.

The words at late doubled base plus 0x14 and adjusted base plus 4 are captured afterward. Consequently, the 0x3A store may affect these later captures through overlap, but cannot retroactively affect the first two saved words.

Subsequent byte stores place the adjusted base’s low byte at late doubled base plus 0x0D, 0x6C at adjusted base plus 8, and the linking base’s low byte at adjusted base plus 0x0D and plus 9. Late doubled base plus 0x54 receives its own base value as a word.

Absolute byte address 0x67 receives the saved 0xA9 word’s low byte. A word read at late doubled base plus 0x44 intervenes before absolute byte address 0x21 receives a freshly read byte from 0x67.

The offset-0x44 capture supplies a byte at adjusted base plus 0x10 and a word at linking base plus 0x54. Its right-shifted result supplies a byte at late doubled base plus 1; the shift count is the adjusted base’s low eight bits. Signed right-shift treatment and counts above 31 remain documented_only. Five-bit count masking is not established.

### Offset-0x40 capture and later 0x67 reloads

Linking base plus 0x54 is overwritten with the adjusted base before the word at adjusted base plus 0x40 is captured. The saved adjusted-base offset-4 word is then written at linking base plus 0x74. Only afterward is the offset-0x40 capture stored as a word at adjusted base plus 0x10.

If the offset-0x40 capture location overlaps the offset-0x74 destination, the offset-0x10 payload is the earlier capture, not a post-overwrite reread.

After linking base plus 0x44 receives the late doubled base, two assignments to adjusted base plus 9 each use a fresh byte read at absolute 0x67. Between those reloads, the function writes the adjusted base’s low byte at late doubled base plus 9, writes the saved 0xA9 word at adjusted base plus 0x44, and captures adjusted base plus 0x54 as a word.

Overlapping stores may change 0x67. Neither reload is replaceable with the original saved low byte, and the reloads need not agree.

### Final mutations and invocation

The adjusted-base offset-0x54 capture becomes a further numerical base. The earlier late-fourfold offset-0x74 capture supplies low-byte payloads at further base plus 0x0D and late doubled base plus 0x11.

The saved late-doubled offset-0x14 word is restored at late doubled base plus 0x54. The word at adjusted base plus 0x14 is captured separately for subsequent low-byte use. Adjusted base plus 9 and plus 0x44 receive the further base’s low byte and full word, respectively.

A word read at late doubled base plus 4 supplies the near-completion base. A 16-bit store of 0x0065 at twice that base precedes capture of its offset-0x64 word.

Further base plus 0x11 receives the earlier adjusted-base offset-0x14 capture’s low byte, and further base plus 0x54 receives word value 100. The near-completion offset-0x74 word is captured before the late-doubled offset-0x64 word. Late doubled base plus 9 then receives the near-completion base’s low byte.

The last two explicit memory stores are both **32-bit word stores**:

- The captured near-completion offset-0x64 word is stored at the captured late-doubled offset-0x64 word plus 0x54.
- Immediately before the terminal invocation, **the captured late-doubled offset-0x64 word is stored at twice the captured near-completion offset-0x64 value**, with no added field offset.

The terminal dependency receives four represented arguments, in order: 0x72; the captured near-completion offset-0x64 word; the saved late-doubled offset-0x14 word; and the captured near-completion offset-0x74 word.

There is no represented success check afterward.

## Side effects

Intermediate accesses, repeated writes, overwritten values, and capture timing are observable parts of this contract, not merely steps toward final memory contents.

| Computed-base offset | Represented access |
|---|---|
| 0 | **8-bit byte, 16-bit halfword, and 32-bit word stores** at scaled destinations |
| 0x01 | Byte store |
| 0x04 | Byte read and word read |
| 0x05 | Byte read |
| 0x08, 0x09, 0x0D | Byte stores |
| 0x10 | Byte store and word store |
| 0x11, 0x15, 0x19 | Byte stores |
| 0x14, 0x24, 0x34 | Word accesses |
| 0x20 | Word stores |
| 0x40 | Word read |
| 0x44, 0x54, 0x64, 0x74 | Word accesses |

The offset-0 word store is the final explicit memory mutation described above. The inventory is not a structure declaration: bases may coincide, fields may overlap, and address-like values may be payloads rather than destination bases.

Fixed-address operations include byte accesses at 0x01, 0x05, 0x15, 0x21, 0x3A, and 0x67, and word accesses at 0x54, 0x60, and 0xA9.

No local allocation, release, synchronization, transaction, ownership transfer, or reentrancy guarantee is established.

## Dependencies

Only F-d7b3baad3c44 is resolved by the supplied identity map.

| Opaque documentary ID | Role |
|---|---|
| F-unknown-01 | Initial incoming-context-derived invocation |
| F-unknown-02 | Later control-location-derived invocation family |
| F-unknown-03 | Terminal four-argument indirect invocation |

These labels do not establish three distinct implementations or one shared implementation for the invocation family.

For every dependency, termination, error signaling, global preservation, memory preservation, allocation, retention, and release are documented_only. Passing an address-like integer does not establish ownership transfer or authorize any assumption about later memory validity.

## Acceptance examples

These are conditional static examples, not executed tests.

### Width and overwrite

With accessible memory and 32-bit bit-vector arithmetic, shifted handle plus 9 receives the upper-result byte and later zero. A single final-zero update does not reproduce the two-store trace.

A captured byte 0xE7 widened into a word payload becomes 0x000000E7, not 0xFFFFFFE7.

### Destination versus payload

At the identified offset-0x34 mutation, suppose the small base is 0x100 and the link seed is 0x80. The original fourfold base is 0x200. The required store is a 32-bit payload of 0x00000200 at address 0x134, not a store at 0x234.

This example concerns that access, not a guarantee that later writes or the terminal dependency preserve its contents.

### Final offset-0 word store

Suppose the captured near-completion offset-0x64 word is 0x120 and the captured late-doubled offset-0x64 word is 0x89ABCDEF. If execution reaches the final store and address 0x240 supports the access, the function writes the full 32-bit value 0x89ABCDEF there before invoking F-unknown-03.

A byte or halfword write is not equivalent. Byte ordering within the word remains environment-dependent.

### Absolute 0x60 ordering

The fixed-storage linking-base capture precedes the word store of 0x00000020 at absolute 0x60, which precedes the first eightfold-derived offset-0x54 store.

### Captures across overlapping writes

If the 0x3A byte store overlaps a later adjusted-base offset-4 capture, that capture observes post-store memory. The earlier 0xA9 and late-fourfold offset-0x74 captures remain saved values.

If adjusted base plus 0x40 coincides with linking base plus 0x74, adjusted base plus 0x10 receives the value captured before the offset-0x74 overwrite.

### Distinct 0x67 reloads and saved argument

Suppose the first late reload sees 0xE7, late doubled base plus 9 equals absolute 0x67, and the adjusted base’s low byte is 0x08. With no other intervening mutation of 0x67, the first adjusted-base offset-9 store receives 0xE7 and the second receives 0x08.

If later writes alter the original late-doubled offset-0x14 capture location, the third terminal argument still uses the saved word rather than a reread.

## Uncertainties

The following remain documented_only:

- Application intent and object layouts.
- Actual dependency identities, contracts, and implicit inputs.
- Calling-convention and function-boundary fidelity.
- Overflow semantics outside the conditional arithmetic model.
- Endianness, alignment, permissions, and device-memory behavior.
- Signed right-shift behavior and excessive shift counts.
- Object extents, aliasing restrictions, and memory ownership.
- Exact repetition multiplicity not enumerated here.
- Concurrency, termination, terminal-target reachability, and return behavior.

Represented access order does not establish hardware bus ordering or synchronization guarantees. Descriptive labels such as “linking” express value relationships, not established application intent.

## Evidence

Claims are grounded in the supplied static representation’s explicit access widths, arithmetic operands, invocation operands, retained captures, and ordered mutations.

The offset-0 inventory includes a word store because the final explicit mutation uses a 32-bit payload at twice a captured value. That payload is the captured late-doubled offset-0x64 word.

The offset-0x34 distinction follows from the destination being small base plus 0x34 while its payload is four times the link seed. The original fourfold base separately supplies offset-0x14 and offset-0x54 destinations.

The absolute 0x60 write, post-repetition capture boundary, adjusted-base offset-0x40 capture, and separate late 0x67 reloads are supported by their represented accesses.

Embedded comments, earlier prose, and reviewer feedback are not independent behavioral evidence. No implementation lookup, execution, runtime trace, or test has been performed.
