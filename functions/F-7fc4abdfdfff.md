<!-- Trusted provenance: F-7fc4abdfdfff; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7fc4abdfdfff

## Purpose

F-7fc4abdfdfff performs an ordered sequence of indirect calls, captured-value transfers, address-derived memory writes, two byte lookups, a global assignment, and final clearing operations. Its last visible action before returning is an unconditional call to address zero with three zero arguments.

The supplied static fragment does not establish an application-level purpose. Allocation, initialization, serialization, dispatch, and cleanup are not established interpretations.

This contract describes explicit behavior in that fragment. Dependencies may introduce additional effects between the operations described below. Their memory ownership, global preservation, error behavior, and ability to return are unknown.

## Interface

### Explicit inputs

The represented interface accepts three 32-bit inputs.

| Input | Established use |
|---|---|
| First input | Passed unchanged to F-D001. No other explicit use is visible. |
| Second input | No explicit use is visible. |
| Third input | Treated as unsigned for masking and logical shifts; supplies a transformed call argument and a derived word-store address. |

The first two inputs have no established application-level type or signedness. Lack of explicit use does not establish that a dependency cannot observe incoming register or execution state.

### Implicit inputs

The routine also consumes:

- Incoming link-related execution state used to select F-D001’s target by adding 0xffe.
- A sampled global value assigned to a 32-bit retained quantity and subsequently used as an address base.
- Dependency results, including two quantities represented as 64 bits.
- An additional result-register value interpreted as a pointer for the final byte clear.

The sampled global’s declaration is absent. Its source width, signedness, access width, and any conversion into the retained 32-bit quantity are therefore not independently established.

A separate terminal global destination receives F-D065’s result. Neither that global’s declaration nor an independently typed result for F-D065 is supplied. The result width, destination width, assignment conversion, and physical write width remain documented_only. This assignment must not be classified unconditionally as a 32-bit write.

The relationship between these globals and other storage is unknown; disjointness is not guaranteed.

### Output and continuation

No explicit return value is represented.

The visible return is reached only if every preceding operation completes and the final address-zero call returns. No validity checks, alignment checks, null checks, error branches, or rollback are visible.

Whether a failing access traps, whether a dependency signals an error, and whether the final call returns are environmental or dependency-specific unknowns.

### Width and arithmetic model

Explicitly typed memory accesses use 8-bit bytes, 16-bit halfwords, and 32-bit words. Two dependency results are represented as 64-bit quantities and consumed through low and high 32-bit halves.

Unless qualified otherwise, derived addresses and scaled values below describe the apparent 32-bit bit-pattern operations:

- Multiplication by four or sixteen retains the low 32 bits.
- Address addition uses the resulting 32-bit address bits.
- Byte stores retain the low eight bits of their value.
- Shifts of the unsigned third input, captured unsigned words, and masked unsigned values are logical.

Some scaling expressions use signed intermediates in the supplied representation. It does not establish portable language-level behavior for signed overflow or signed left shifts. The bit-pattern model states the apparent low-level operation, not an unconditional source-language overflow guarantee.

Valid address mappings, permissions, alignment behavior, endianness, and device-memory semantics are prerequisites, not supplied guarantees.

## Behavior

All phases execute in the following order if preceding operations complete. No conditional branches or loops are visible.

F-D001 through F-D066 are opaque identifiers for call occurrences in execution order. Different identifiers do not establish different implementations or targets. “No explicit arguments” describes the represented call, not all execution state a dependency might observe.

### 1. Capture a word and publish its low byte

F-D001 receives the first input. F-D002, F-D003, F-D004, and F-D005 then execute in order without explicit arguments.

Four times F-D005’s 32-bit result supplies the **first base**. The routine:

1. Reads a 32-bit word at the first base plus 0x74, retaining the **first captured word**.
2. Writes that word’s low byte at the first base plus 1.
3. Calls F-D006 with the first base.

The word read precedes the byte write. Subsequent uses of the captured word do not reread its original address.

### 2. Publish the captured word and transform the third input

F-D007, F-D008, F-D009, and F-D010 execute in order without explicit arguments.

The routine writes the first captured word as a 32-bit word at sixteen times F-D010’s 32-bit result, plus 0x14.

F-D011 then receives the third input logically shifted right by two and then left by sixteen, retaining a 32-bit result. Original input bits 2 through 17 become argument bits 16 through 31; all other input bits are discarded.

F-D012 executes without explicit arguments, and its 32-bit result is passed unchanged to F-D013. F-D014 then executes without explicit arguments, and its 32-bit result is passed unchanged to F-D015.

### 3. Publish the first captured word again

F-D016, F-D017, F-D018, and F-D019 execute in order without explicit arguments.

Four times F-D019’s 32-bit result supplies the **second base**. The routine writes the first captured word at the second base plus 0x14, using a 32-bit store, then calls F-D020 with the second base.

Intervening changes to the original capture location do not change the retained value used here.

### 4. Write through an address derived from the third input

F-D021, F-D022, F-D023, and F-D024 execute in order without explicit arguments.

Four times F-D024’s 32-bit result becomes the **third scaled value**. The destination address is formed by clearing the third input’s low two bits and then adding 0x14. A 32-bit store writes the third scaled value there.

F-D025 receives the third scaled value twice.

F-D026 executes without explicit arguments, and its 32-bit result is passed unchanged to F-D027. F-D028 then executes without explicit arguments, and its 32-bit result is passed unchanged to F-D029.

Low-bit clearing precedes the offset addition. It does not establish that the address is mapped or writable.

### 5. Clear a dependency-selected word

F-D030, F-D031, F-D032, and F-D033 execute in order without explicit arguments.

Four times F-D033’s 32-bit result supplies another base. The routine writes a 32-bit zero at that base plus 0x14, then calls F-D034 with the base.

This word clear is distinct from the final byte and halfword clears.

### 6. Split a result and sample a global

F-D035, F-D036, F-D037, and F-D038 execute in order without explicit arguments.

F-D038’s represented 64-bit result is divided into low and high 32-bit halves. Four times the low half becomes the **retained scaled value**. The routine writes that value as a 32-bit word at four times the high half, plus 0x14.

After that store, the sampled global is evaluated and assigned to a 32-bit retained quantity, the **global-base snapshot**. This establishes the retained quantity’s width, not the global read’s width. The missing global declaration prevents specifying the source access width or conversion independently.

F-D039 then receives, in order:

1. The retained scaled value.
2. F-D038’s unscaled high half.
3. The retained scaled value.
4. The retained scaled value.

Later dependency changes to the sampled global do not replace the retained snapshot in the represented data flow.

### 7. Republish the scaled value and capture another word

F-D040, F-D041, F-D042, and F-D043 execute in order without explicit arguments.

Four times F-D043’s 32-bit result supplies the next base. The routine:

1. Writes the retained scaled value as a 32-bit word at that base plus 0x14.
2. Reads a 32-bit word at the global-base snapshot plus 0x14, retaining the **second captured word**.
3. Calls F-D044 with the newly selected base.

There is no dependency call between this store and read. If their addresses coincide, ordinary readable and writable memory yields the newly stored value, absent asynchronous modification or exceptional memory semantics.

### 8. Perform the first lookup-directed byte update

F-D045, F-D046, F-D047, and F-D048 execute in order without explicit arguments. Both 32-bit halves of F-D048’s represented 64-bit result are retained.

The first captured word is logically shifted right by 23 and increased by one to produce a lookup address. The routine reads an unsigned byte there, adds 0x11 to its numeric value, and uses that sum as the destination address for a byte store.

The stored value is the second captured word’s low byte.

The lookup address ranges from 1 through 512 inclusive. The destination ranges from 0x11 through 0x110 inclusive. The destination addition is not truncated to eight bits; no additional base or pointer-sized table entry is visible.

F-D049 then receives F-D048’s high half three times, followed by four times its low half.

F-D050 executes without explicit arguments, and its 32-bit result is passed unchanged to F-D051.

### 9. Perform the second lookup-directed byte update

F-D052, F-D053, F-D054, and F-D055 execute in order without explicit arguments. F-D055’s 32-bit result is retained for the following call.

F-D048’s retained low half is masked with 0x3fffffff, logically shifted right by 21, and increased by one. The routine reads an unsigned byte at that address. Adding 0x11 to that byte supplies the destination for another byte store.

The stored byte consists of original bits 22 through 29 of the second captured word. It is not the next adjacent byte after the first byte publication.

F-D056 receives four times F-D055’s result.

This lookup again addresses bytes 1 through 512 and selects destinations 0x11 through 0x110. It is a fresh read after intervening calls and the first lookup-directed write. Even equal lookup addresses do not justify reusing the earlier byte value.

### 10. Assign a global and perform final clears

F-D057, F-D058, F-D059, and F-D060 execute in order without explicit arguments. F-D061 receives four times F-D060’s 32-bit result twice.

F-D062 executes without explicit arguments. F-D063 then executes without explicit arguments, and its 32-bit result is passed unchanged to F-D064.

F-D065 executes without explicit arguments. Its result is assigned to the terminal global destination. The assignment’s result width, destination width, and conversion are unresolved because the relevant global declaration and independent result type are absent.

Next, the routine writes a zero byte through the pointer represented by the additional result-register value. Its provenance and validity are not independently established.

The routine then masks F-D048’s retained low half with 0x3fffffff, logically shifts it right by seven, and writes a 16-bit zero at the resulting address. This address ranges from zero through 0x7fffff inclusive and need not be even.

Finally, F-D066 calls address zero with three zero arguments. The visible return follows only if that call returns.

The terminal global assignment precedes the pointer-directed byte clear; that clear precedes the halfword clear; both precede F-D066.

## Side effects

The following counts describe explicit accesses if execution reaches every phase. Dependency internals are excluded.

| Access category | Count | Qualification |
|---|---:|---|
| Explicitly typed 32-bit reads | 2 | First and second captured words |
| Global sampling | 1 | Source access width unknown; retained quantity is 32 bits |
| Unsigned byte reads | 2 | Lookup reads |
| Explicitly typed pre-terminal 32-bit stores | 6 | One in each of phases 2 through 7 |
| Terminal global assignment | 1 | Destination and write width unknown |
| Byte stores | 4 | Initial publication, two lookup-directed updates, pointer-directed clear |
| 16-bit stores | 1 | Final halfword clear |

The initial publication is a byte store, not another word store. Three 32-bit reads and seven 32-bit writes are not unconditional totals. Those totals would require additional declarations establishing 32-bit global accesses and the relevant assignment behavior.

These are access counts, not distinct-location counts. Explicit addresses and global storage may overlap. A write may alter a later lookup byte, and final clears may affect previously accessed storage.

Every explicit store or assignment is unconditional once execution reaches it. No rollback is visible. Earlier completed effects are not explicitly undone if a later access or call fails.

Dependencies may perform additional reads, writes, allocation, release, synchronization, or control transfers. None is assumed to preserve globals or memory.

No ownership transfer, allocation contract, or release obligation is established for inputs, dependency results, globals, or derived addresses.

## Dependencies

| Opaque IDs | Established interface at these occurrences |
|---|---|
| F-D001 | Receives the first input; target depends on incoming link-related state. |
| F-D005, F-D010, F-D019, F-D024, F-D033, F-D043, F-D055, F-D060 | Supply 32-bit results used in scaling or address formation. |
| F-D038, F-D048 | Supply represented 64-bit results consumed as two 32-bit halves. |
| F-D012, F-D014, F-D026, F-D028, F-D050, F-D063 | Supply 32-bit results passed unchanged to their immediately following consumers. |
| F-D006, F-D011, F-D013, F-D015, F-D020, F-D025, F-D027, F-D029, F-D034, F-D039, F-D044, F-D049, F-D051, F-D056, F-D061, F-D064 | Receive the arguments specified above. |
| F-D065 | Supplies the value assigned to the terminal global; result width and assignment conversion are unresolved. An additional result-register pointer is consumed afterward, with provenance unresolved. |
| F-D066 | Calls address zero with three zero arguments. |
| Remaining occurrences | Have no explicit arguments; their results are not explicitly consumed. |

No dependency body or canonical identity is supplied. A represented argument list does not establish a complete calling convention, preservation guarantee, memory contract, or error protocol.

## Acceptance examples

These are conditional static examples, not executed tests. Calls are assumed to return and accesses to use ordinary memory semantics unless otherwise stated.

### Capture and truncation

If the first captured word is 0x89abcdef, the initial byte publication writes 0xef. The phase-2 and phase-3 word publications retain 0x89abcdef even if an intervening dependency changes the original capture location.

### Third-input transformations

For a third input of 0x00000007, F-D011 receives 0x00010000, and the phase-4 store address is 0x18. Inputs 0x00000004 through 0x00000007 produce those same derived values.

Inputs 0x00000004 and 0x00040004 produce the same F-D011 argument but different phase-4 addresses: 0x18 and 0x40018.

Under the stated 32-bit address-bit model, input 0xffffffff produces argument 0xffff0000 and phase-4 address 0x10. Accessibility of that wrapped address remains environmental.

### Snapshot and read-after-write aliasing

Suppose global sampling produces retained address bits 0x1000. A later dependency changing the sampled global does not redirect phase 7’s explicit read from 0x1014.

This example specifies the retained value, not the sampled global’s storage width or conversion.

If phase 7’s selected base also equals 0x1000, its store and following read address the same word. With ordinary memory and no asynchronous modification, the second captured word equals the retained scaled value.

### Lookup values and destinations

If the second captured word is 0xa5c3d2e1, the first lookup-directed write stores 0xe1 and the second stores 0x97.

A lookup byte of 0x20 selects destination 0x31. A lookup byte of 0xff selects 0x110, not 0x10.

If the first write changes the byte later read by the second lookup, the second lookup consumes the later contents, including intervening dependency effects.

### Final clears and completion

If F-D048’s retained low half is 0x00000180, the final halfword-clear address is 3. The routine does not align or reject it; success of the unaligned store is documented_only.

No exact terminal-global byte layout can be required without its missing type information.

If F-D066 does not return, the routine does not reach its visible return. Earlier completed writes are not explicitly undone.

## Uncertainties

### Established static evidence

The fragment establishes call order, represented arguments, explicit access widths, masks, shifts, captured-value reuse, six pre-terminal word stores, global sampling into a 32-bit retained quantity, terminal global assignment, and ordered final clears.

It does not establish the widths of either global access.

### Inferred intent

Repeated scaling by four may indicate encoded references or indexing. Repeated offset-0x14 stores may indicate a common field layout. Neither interpretation adds an acceptance requirement.

### Documented-only unknowns

The following remain documented_only:

- Global declarations, access widths, signedness, and assignment conversions.
- F-D065’s independently established result width.
- Dependency identities, effects, failure behavior, and calling conventions.
- Accuracy of represented multi-register results.
- Origin, lifetime, and validity of the final pointer.
- Global overlap, concurrency, and asynchronous modification rules.
- Address mappings, permissions, endianness, alignment, and device-memory behavior.
- Exact semantics of signed overflow and signed left shifts.
- Executability of address zero and whether F-D066 returns.
- Hidden observation of the second input or other execution state.
- Ownership, allocation, release, and error signaling within dependencies.

No runtime tests have been performed.

## Evidence

This contract uses only the supplied static fragment. No dependency bodies, global declarations, execution traces, platform specification, or runtime results were available.

Two captured-word reads have explicit 32-bit access types. Global sampling establishes a 32-bit retained quantity but does not independently establish the source access width.

Six pre-terminal stores have explicit 32-bit access types. The terminal global assignment lacks the declarations needed to classify its result or write width. It is therefore counted separately rather than added unconditionally to the 32-bit-store total.

Opaque dependency identifiers describe call occurrences without importing semantic identities. Static observations, inferred intent, and unresolved behavior remain separated.
