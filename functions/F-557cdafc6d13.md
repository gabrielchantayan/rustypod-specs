<!-- Trusted provenance: F-557cdafc6d13; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-557cdafc6d13

## Purpose

The supplied static representation describes an ordered sequence of indirect calls, address calculations, byte accesses, and 32-bit memory accesses. It does not establish an application-level purpose.

The visible behavior repeatedly scales values returned by dependencies, uses the results as addresses, copies low bytes from words into byte fields, writes shifted values into word fields, and accesses low absolute addresses. No ordinary return is visible. The represented path ends at an invalid-instruction boundary.

This contract describes the visible path conditionally: each operation is reached only if preceding calls and memory accesses complete and execution continues as represented. It does not establish that the entire path is executable or that the recovered interface is correct.

## Interface

**Explicit inputs**

- Two 32-bit values are declared.
- The first has no visible use.
- The second is forwarded as the second argument of the first indirect call. No visible validation, interpretation, or dereference of that value occurs locally.

**Additional entry state**

The representation also consumes state not supplied through the declared parameters:

- An entry address used for an initial word read and byte write.
- An unsigned 32-bit entry quantity used in scaling, masking, shifts, and address calculations.
- An entry control-link value used to derive the first indirect-call target.
- Secondary call-result state later consumed as arguments or addresses without appearing as an ordinary primary return value.

Consequently, the two declared arguments alone are insufficient to specify a usable calling interface. The calling convention and required machine state are **documented_only**.

**Outputs**

No ordinary return value is declared or visibly produced. Observable outputs consist of memory writes, indirect calls, and the final invalid-instruction boundary. Dependency effects are additional outputs whose nature is unknown.

**Memory requirements**

Every reached read requires accessible memory at its computed address; every reached write requires writable memory. The visible accesses include:

| Access form | Width | Meaning established by the representation |
|---|---:|---|
| Base plus 0x74 | 32 bits | Read a word, sometimes subsequently treated as an address |
| Base plus 0x01 | 8 bits | Read or replace a byte |
| Base plus 0x0D | 8 bits | Replace a byte |
| Base plus 0x14 | 32 bits | Store a word |
| Base plus 0x24 | 32 bits | Store a word |
| Base plus 0x34 | 32 bits | Store a word |
| Zero-extended byte plus 0x11 | 8 bits | Access an absolute address from 0x11 through 0x110 |
| Absolute 0x0D and 0x11 | 8 bits | Replace bytes directly |

These offsets do not establish a structure definition, allocation boundary, or ownership relationship.

## Behavior

### Arithmetic and ordering conventions

Byte stores retain only the low eight bits of their source. Byte loads used in address calculations are unsigned and therefore range from 0 through 255.

The representation uses 32-bit quantities and 64-bit paired results. For a paired result, “low half” and “high half” refer to its low and high 32 bits, not to memory byte order.

Unsigned right shifts are zero-filling. In particular:

- Masking with 0x7FFFFFFF before shifting right removes bit 31.
- Masking with 0x3FFFFFFF retains bits 0–29.
- Masking with 0x07FFFFFF retains bits 0–26.
- Shifting a 32-bit word right by 22 produces at most 0x3FF.
- A byte-derived address plus 0x11 remains in the inclusive range 0x11–0x110.

Multiplications, additions, and left shifts below describe the visible 32-bit address/value calculations. A modulo-\(2^{32}\) machine interpretation is useful but is not established for every signed expression by this representation alone. Examples involving overflow are therefore conditional on that interpretation. No overflow checks are visible.

All listed phases preserve visible order. Intervening dependencies may change memory. A later load must not be replaced by an earlier snapshot unless equivalence is independently established.

### Entry effects and initial call sequence

The function first reads a 32-bit word at the entry address plus 0x74 and writes its low byte at the entry address plus 1.

It then invokes F-unknown-entry with three visible arguments:

1. Twice the unsigned entry quantity.
2. The second explicit input.
3. The entry quantity shifted right by two.

The next visible call produces a paired 64-bit result. A subsequent call receives its low half, high half, low half again, and the entry quantity masked with 0x7FFFFFFF and shifted right by one.

After further calls with no explicit arguments, a returned word is multiplied by four. At that scaled address, the function reads a word at offset 0x74 and writes its low byte at offset 1. The scaled address is then passed to another dependency.

After additional calls, the function reads the byte at twice the entry quantity plus 1. It uses that byte as an absolute-address component and writes zero at the byte value plus 0x11.

### Early derived-address updates

A call receives secondary result state from an earlier dependency. A following primary result is retained as a base address and passed to another call.

Later, the function rereads the byte at twice the entry quantity plus 1. At that byte-derived address plus 0x11, it writes the most significant byte of twice the entry quantity. The earlier zero write and this write need not address the same byte: intervening dependencies can change the selector byte.

Another returned value, scaled by four, is passed to a dependency. A later result is also scaled by four and retained as a working address.

The function then:

- Reads a 32-bit word at the previously retained base plus 4.
- Reads a byte at that word’s value plus 1.
- Writes zero at the byte value plus 0x11.
- Calls a dependency with three identical arguments, each equal to the working address.

After intervening calls, it shifts the retained word right by two, reads a byte at that shifted address plus 1, and writes zero at the newly read byte value plus 0x11. The byte value itself is retained as a small address base.

A later returned word, scaled by four, is passed to another dependency.

### Word extraction and first indexed word stores

The function reads a word at the working address plus 4. Separately, it reads a word at the small address base plus 0x74 and writes that word’s low byte at the small address base plus 1.

A dependency receives four visible arguments:

- The latter word.
- The working address shifted left by 19.
- The same shifted address again.
- A recent primary result shifted left by two.

The word read at working-address-plus-4 is shifted right by 22 and retained as an extracted value. A following call receives a recent result and the word read at small-base-plus-0x74 shifted right by two.

After more calls, a returned value is scaled by four. The function reads a word at this new address plus 0x74, writes its low byte at the new address plus 1, and passes the new address to a dependency.

Following further calls, that retained word is stored at an address formed from secondary result state multiplied by four, plus 0x14. The secondary result state is then passed to a dependency.

A later call receives a recent result and two identical values formed by masking the extracted value with 0x1FF and shifting left by 21.

A subsequent primary result selects another word destination: four times that result plus 0x14. The stored word is the extracted value shifted left by 23. Four times the result is then passed to a dependency.

### Masked values and repeated offset-0x34 updates

A later returned word becomes an unmasked index. Before masking it, the function stores four times that index at sixteen times the index plus 0x14.

It then masks the index with 0x3FFFFFFF, retains both the masked value and that value shifted right by two, and passes the masked value to a dependency.

Another returned value selects a destination at four times that value plus 0x14. The masked value is stored there, and four times the destination-selecting value is passed to a dependency.

After more calls:

- Eight times the destination-selecting value becomes a retained field base.
- Its low byte is stored at the earlier word-from-small-base value plus 1.
- A new dependency result shifted left by two is stored at field-base-plus-0x34.
- That shifted result is passed to a dependency.

The new result is also masked with 0x3FFFFFFF and retained as another address base.

After another paired return, the function writes the low byte of the retained field base at this masked address base plus 1. It then replaces the word at field-base-plus-0x34 with the paired return’s high half. The low half shifted left by two is passed to a dependency.

The two writes at field-base-plus-0x34 are distinct observable events. The second does not eliminate the first.

### Byte updates and absolute-address writes

Sixteen times the earlier destination-selecting value becomes another retained base.

The function writes:

- The low byte of this base at the address formed by masking the earlier new result with 0x3FFFFFFF, shifting right by 20, and adding 0x11.
- The low byte of four times the unmasked index at the retained base plus 0x0D.

A call receives two identical arguments formed by shifting a recent result left by two. A following result is passed to another call together with that result shifted right by three.

After additional calls, the function writes the low byte of the retained base directly at absolute address 0x11. It repeats the write at retained-base-plus-0x0D, then passes a recent result shifted left by two to a dependency.

Later, it writes four times the unmasked index at masked-address-base-plus-0x14, then writes the low byte of twice that masked address base at offset 0x0D of the same base.

### Scaled working record and selector clearing

A new returned value establishes:

- A working record address equal to 32 times the value.
- A word value equal to the value shifted left by seven.
- A bounded selector equal to the value masked with 0x07FFFFFF and shifted right by 17; this selector is at most 0x3FF.

After three more calls, the function stores the retained word value at working-record-plus-0x14. It writes the low byte of a recent result shifted left by two at working-record-plus-0x0D, then passes that shifted result to a dependency.

Following more calls, it writes:

1. The low byte of twice the earlier masked-index value shifted right by two at bounded-selector-plus-7.
2. The low byte of the bounded selector at an address selected by a fresh byte read from working-record-plus-1, plus 0x11.

A dependency receives three identical arguments, each formed by shifting the recent result left by 21.

After another group of calls, the function rereads the byte at working-record-plus-1, retains it, and writes zero at that byte plus 0x11.

### Low-address byte replacement and offset-0x24 stores

After further calls, the function writes the low byte of a retained dependency result shifted right by three at working-record-plus-0x0D.

It writes the retained selector byte directly at absolute address 0x0D. A dependency then receives twice that selector byte, 0x98, and 0x260.

Another call receives secondary result state shifted right by three. After more calls, a returned word shifted left by two is passed to a dependency that itself returns a word.

The function then:

- Writes the previously retained shifted-right-by-three byte at twice the selector byte plus 0x0D.
- Replaces absolute address 0x0D with 0xC0.
- Passes the latest returned word shifted left by two to a dependency.

A subsequent result, shifted left by two, is stored at twice-selector-plus-0x24. The function reads a word at twice-selector-plus-0x74 and retains it as a new address-bearing word. It also retains the unshifted recent result shifted left by five.

A dependency receives the address-bearing word, twice that word, the retained left-by-five value, and the address-bearing word masked with 0x7FFFFFFF then shifted right by two.

After intervening calls, the left-by-five value is stored at address-bearing-word-plus-0x24. A fresh word read from address-bearing-word-plus-0x74 is passed to a dependency.

### Final indexed stores and selector updates

A later paired return supplies a low half and a high half. The function:

- Stores an earlier retained dependency result shifted right by three at four times the high half plus 0x14.
- Retains the low half shifted left by two.
- Calls a dependency with that shifted low half repeated three times and the high half shifted right by three.

Another paired return is forwarded as low half, high half, low half again, and high half shifted right by three.

After further calls, the retained shifted low half is stored at four times a recent result plus 0x14. Four times that recent result is passed to a dependency.

A later returned word is shifted left by two and retained. The function reads the byte at address-bearing-word-plus-1 and writes the retained shifted value’s low byte at that byte plus 0x11.

Following three calls, it rereads the same selector location. At the newly selected address plus 0x11, it writes the low byte of the returned word masked with 0x3FFFFFFF and shifted right by 20.

After another call group, it reads a byte at 1024 times the retained shifted value plus 1 and writes zero at that byte plus 0x11. A dependency receives the retained shifted value shifted left by three twice, the retained shifted value itself, and a recent result shifted left by two.

After additional calls, the same selector location is read again and its selected byte is cleared again. Further calls follow, including one receiving a recent result shifted left by two. The visible path then terminates at an invalid-instruction boundary.

## Side effects

The function visibly performs byte and word stores, including stores to absolute addresses 0x0D and 0x11. Whether those addresses represent ordinary memory, mapped hardware, or invalid addresses is unknown.

No allocation, release, ownership transfer, reference counting, locking, or cleanup protocol is established. Passing an address-like value to a dependency does not establish whether that dependency reads, retains, modifies, or releases the referenced storage.

There are no visible null checks, bounds checks, status checks, retries, error returns, or rollback. If execution stops partway through, prior writes may remain observable.

Memory locations can alias. Even stores described using different bases may target the same address. No independence between fields or allocations may be assumed.

The representation contains no explicit local branch or loop on the visible path. Dependencies may nevertheless branch, fail, trap, or not return. Concurrency, atomicity, alignment requirements, and device-memory ordering are **documented_only**.

## Dependencies

Only F-557cdafc6d13 has a supplied resolved opaque identity.

For this contract:

- **F-unknown-entry** denotes the first unresolved indirect-call occurrence, whose target depends on entry control-link state.
- **F-unknown-indirect** denotes the remaining unresolved indirect-call occurrences collectively. It is not an assertion that they share one target or implementation.

These labels are documentary function IDs, not recovered identities.

The remaining calls derive targets from embedded address-like anchors and offsets. The representation shows visible argument expressions and selected primary, paired, or secondary results, but does not establish complete signatures or calling conventions.

Calls displayed without explicit arguments are still observable dependency invocations. They must not be treated as pure operations, harmless padding, or calls with no machine-state inputs. No dependency is assumed to preserve globals, preserve memory, or own or release storage.

## Acceptance examples

These examples are conditional specification checks, not executed tests.

1. **Initial low-byte copy.** If the initial word at entry-address-plus-0x74 is 0x123456AB and both accesses succeed, the local byte write at entry-address-plus-1 is 0xAB and precedes F-unknown-entry.

2. **First argument transformation.** With entry quantity 12, F-unknown-entry receives 24, the second explicit input unchanged, and 3 as its three visible arguments.

3. **Unsigned selector addressing.** If a selector byte is 0xFF, its byte-derived destination is absolute address 0x110. Interpreting the selector as signed and targeting 0x10 would not match the represented access.

4. **Reload across dependencies.** If the byte at twice-entry-quantity-plus-1 is 3 before the first clear and becomes 7 during intervening calls, the clear targets 0x14 and the later most-significant-byte store targets 0x18.

5. **Distinct field writes.** If the first offset-0x34 value is 0x40 and the later paired high half is 0xABCDEF01, both stores occur in that order. Keeping only the latter store does not preserve observable behavior.

6. **Mask boundary.** An extracted value of 0x3FF produces 0x3FE00000 after masking with 0x1FF and shifting left by 21. Under 32-bit truncating shifts, shifting the unmasked extracted value left by 23 produces 0xFF800000.

7. **Repeated selector clear.** If a dependency changes the final selector byte between its two reads, the two zero stores can target different addresses. Caching the selector across those calls changes behavior.

8. **Absolute replacement.** If execution reaches the low-address replacement phase, absolute address 0x0D first receives the retained selector byte and later receives 0xC0, with dependency calls between them.

9. **Fault before completion.** If an early address is inaccessible, this contract does not promise execution of subsequent calls or stores, restoration of earlier memory, or an ordinary error result.

## Uncertainties

**Established static evidence**

- The listed widths, masks, shifts, explicit arguments, memory accesses, and their represented order.
- Consumption of additional entry state beyond the declared inputs.
- Repeated selector reloads across calls.
- Direct low-address writes.
- Absence of a visible ordinary return before the invalid-instruction boundary.

**Inferred intent**

Repeated offsets and scaling may indicate record manipulation or encoded addresses. That interpretation is not required by this contract. No application-level role is established.

**Documented_only unknowns**

- Whether the visible instruction stream is valid and reachable.
- The complete ABI and provenance of implicit entry and secondary result state.
- Dependency identities, full signatures, side effects, and failure behavior.
- Signed-overflow and address-wrap semantics where recovery is ambiguous.
- Endianness, alignment rules, memory mappings, and valid address ranges.
- Ownership and lifetime of any address-bearing value.
- Whether execution has a valid continuation outside the supplied representation.
- Runtime termination, faults, and externally visible results.

## Evidence

This contract is based solely on the supplied static representation. Its initial control-flow warning and terminal invalid-instruction boundary materially limit confidence in executable behavior and interface recovery.

No implementation access, dependency inspection, runtime execution, or tests were performed. The acceptance examples are conditional consequences of the represented operations, not observed outcomes.
