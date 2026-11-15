<!-- Trusted provenance: F-8869ebebda85; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8869ebebda85

## Purpose

F-8869ebebda85 interleaves indirect dependency calls with byte, halfword, and word memory accesses. Its represented effects include storing shifted dependency results, copying low bytes from loaded words, clearing computed destinations, writing a shared output word twice, and finally calling address zero with three zero arguments.

No application-level purpose is established. Repeated field-copying and clearing patterns do not establish that this is an initializer, allocator, serializer, or supervisor-call implementation.

This is a conditional behavioral contract derived from the supplied static representation. Hidden register inputs, unresolved call targets, overlapping global symbols, and low-address accesses leave the validity of the recovered routine boundary and executable interpretation unresolved. No runtime tests have been performed.

## Interface

### Explicit input and result

- One explicit input: an uninterpreted 32-bit value.
- The first dependency receives that value unchanged as its sole displayed argument.
- No return value is represented.
- Normal return is possible only if every preceding operation completes and the final call through address zero returns. There is no displayed success or failure status.

### Additional environmental inputs

The explicit argument is insufficient to determine behavior. The representation also consumes:

- An incoming link value used to calculate the first dependency target.
- An incoming base value used for a byte store at displacement 9 and a word load at displacement 0x34.
- An incoming 32-bit payload used in early word stores.
- Primary and secondary register results from dependencies.
- A final secondary result interpreted as a writable byte pointer.
- A shared source word read at two distinct points.
- A byte at address 1, read repeatedly.
- A three-byte global contribution associated with address 9.
- A shared output word written at two distinct points.

These are environmental inputs, not defaults supplied by the routine.

### Widths, arithmetic, and layouts

| Item | Represented width or layout |
|---|---|
| Explicit argument and ordinary word values | 32 bits |
| Word loads and stores | 32 bits |
| Byte loads and stores | 8 bits |
| Final clearing store | 16 bits |
| Paired dependency results | 64 bits, consumed as low and high 32-bit halves |
| Final output assembly | One low byte plus an upper 24-bit contribution |

Unsigned right shifts are logical. Byte stores retain the low eight bits of the supplied value. Loaded lookup bytes are zero-extended before adding 0x11; the destination calculation is not an eight-bit wrapping addition.

For address scaling and left shifts, this contract uses a **conditional 32-bit machine interpretation**: only the low 32 bits survive. The representation also contains signed calculations whose overflow and left-shift semantics are not independently established. These formulas are therefore not a claim of portable language-level arithmetic.

Endianness is unknown. Taking the low byte of a loaded word must not be equated with reading the byte at that word’s lowest address.

Displacements 1, 9, 0x14, 0x34, and 0x74 describe actual represented address calculations. They do not establish object types, allocation boundaries, or a conventional structure layout.

### Preconditions

Every reached dependency target must support the represented call, and every reached memory access must be valid at its width. Mapping, alignment, privilege, ownership, and lifetime requirements remain environmental.

The routine visibly validates none of these conditions.

## Behavior

### Ordering model

The represented body has no conditional branches or loops. Each later operation is reached only if earlier calls return and earlier accesses complete.

Dependencies are assigned opaque occurrence IDs **F-U001**, **F-U002**, and onward, in encounter order. These IDs distinguish call occurrences, not necessarily distinct implementations.

Calls with no displayed arguments remain observable boundaries. They may consume hidden state, modify memory, or return secondary register values. A forwarding group below means a dependency produces a word and the next dependency receives that word unchanged. Intervening calls must not be removed or treated as preserving globals.

### 1. Initial forwarding and hidden-base accesses

F-U001 receives the explicit input. F-U002 returns a word, and F-U003 receives it unchanged.

After three further calls, F-U007 returns a paired result. Before F-U008 is invoked:

1. Store the low byte of the incoming hidden base at that base plus 9.
2. Load a word from that base plus 0x34.
3. Pass F-U008 the paired result’s low half shifted left by two, its unchanged high half, and the loaded word.

After four further calls, a secondary register result selects two different addresses:

- Store the incoming hidden payload at four times the selector plus 0x14.
- Load a word at the **unscaled** selector plus 0x14.
- Pass the unscaled selector to the next dependency.

The store precedes the load. Their locations can overlap; the calculations must not be merged into one scaled field access.

### 2. Shifted word propagation and first shared sample

Following a forwarding group and four calls, a primary result supplies another selector:

- Store the hidden payload at four times this selector plus 0x14.
- Load a word at the earlier loaded word shifted right by 22, plus 0x14.
- Pass four times the new selector to the next dependency.

After four further calls, retain a new primary result. Shift the newly loaded word right by one, then:

- Store the retained primary result shifted left by two at four times that shifted selector plus 0x14.
- Load a word at the **unshifted** newly loaded word plus 0x14.
- Pass the right-shifted selector to the next dependency.

After another forwarding group and four calls, store the same retained primary result shifted left by two at four times a new dependency-selected value plus 0x14.

Then read the shared source word for the first time. This sample is retained independently of subsequent changes to the source. The next dependency receives the latest selector multiplied by four.

### 3. Extraction and zero-valued word stores

After four intervening calls:

- Shift the last word loaded in the preceding phase right by 22.
- Shift the first shared-source sample right by 23.
- Store the former value at four times a secondary-result selector plus 0x14.
- Invoke a dependency with four arguments: the secondary selector twice, the shared sample shifted right by 23, and the other shifted value.

During the following forwarding sequence, the already shifted shared sample is shifted right by 23 again. For a 32-bit unsigned source, this second shift necessarily produces zero.

Subsequent ordered effects are:

1. Store the retained 22-bit-shifted value at another dependency-selected word location, scaled by four and displaced by 0x14.
2. After additional calls, store a 32-bit zero at a new secondary-result-selected location, scaled by four and displaced by 0x14.
3. Pass that secondary selector to a dependency.
4. After a forwarding sequence, store the doubly shifted shared sample—also zero—at another selected location with the same scaling and displacement.
5. Load a word from the earlier secondary selector masked with 0x1fffffff, shifted right by 19, and then increased by 0x14.

The final load begins between addresses 0x14 and 0x413 inclusive. Its starting address need not be word-aligned.

### 4. Byte-indirect writes and early clears

After further calls, shift the last loaded word right by 22 and retain the result.

The first byte-indirect write then:

- Reads a byte at the doubly shifted shared sample plus 1.
- Adds 0x11 to the zero-extended lookup byte.
- Stores the low byte of the retained shifted word at that destination.

Because the address component is zero, the lookup occurs at address 1. The destination ranges from 0x11 through 0x110.

A dependency receives the same secondary 32-bit result in three argument positions and zero in a fourth. After one intervening call, another dependency receives zero.

Following additional calls:

- Read a byte at four times the retained shifted word plus 1.
- Clear the byte at that lookup value plus 0x11.
- Pass a newly returned primary result shifted left by two to the next dependency.

Further calls produce another primary result, which is passed shifted left by two in two positions. A later returned word is forwarded unchanged.

Next:

- Read the byte at address 1 and clear the byte at its value plus 0x11.
- Capture a secondary result for later use.
- Invoke a dependency with that captured value repeated three times.
- Obtain a new primary result and pass it with the previously retained forwarded word shifted right by 23.

Equal argument values do not establish ownership, pointer identity, or any special aliasing convention.

### 5. First shared-output write and repeated low-byte copies

After the next call group:

1. Read the byte at address 1 again.
2. Retain its zero-extended value as a base.
3. Clear the byte at that base plus 0x11.

Further calls repeat the single-position and two-position forwarding of primary results shifted left by two. A subsequent dependency returns a word.

Store that returned word into the shared output word **before** forwarding it unchanged to another dependency. Retain the returned value separately from the shared memory location; later arguments use the retained value, not a fresh output-word read.

After one more call:

- Load a word at the retained byte-derived base plus 0x74.
- Store its low byte at that base plus 1.

The base is not refreshed from address 1.

A dependency then receives the earlier captured secondary result shifted left by 11. Another produces a paired result, forwarded as its low half, high half, and the retained first output value shifted right by 23.

Subsequent calls produce a selector:

- Multiply it by four.
- Load a word at that scaled base plus 0x74.
- Store the word’s low byte at that scaled base plus 1.
- Pass the scaled base to a dependency.

Later calls produce another primary result. Retain four times that result. Before the next consequential call, repeat the word load and low-byte store at the **previous** scaled base. This is a fresh load; intervening dependencies may have changed the source.

The next dependency receives the captured secondary result shifted left by 12, the newly retained four-times result twice, and the unscaled new primary result shifted left by 18.

### 6. Full-word retention and further stores

After forwarding groups, another selector determines a base scaled by four:

- Load a word at that base plus 0x74.
- Retain the full word.
- Store its low byte at the base plus 1.
- Pass the scaled base to the next dependency.

After further calls, store the retained full word at **16 times** a new selector plus 0x14. This store has stride 16, not four.

A subsequent dependency receives the earlier retained four-times result shifted logically right by two and then left by 16. If the original scaling discarded high bits, the right shift does not recover them.

Further forwarding groups lead to:

- A store of the retained full word at another selector multiplied by four plus 0x14.
- A later primary result shifted left by two.
- A store of that shifted value at the earlier retained four-times result masked with 0xfffffffc, plus 0x14.
- A dependency call receiving the shifted value twice.

After another forwarding group, store a 32-bit zero at a new selector multiplied by four plus 0x14.

Later, obtain a paired result:

1. Shift its low half left by two.
2. Store that shifted value at four times its high half plus 0x14.
3. Read the shared source word a second time.
4. Invoke the next dependency with the shifted low half, the high half, and the shifted low half repeated in two additional positions.

The second shared-source read is fresh and cannot be replaced with the first sample.

### 7. Late byte-indirect writes

After further calls:

- Store the retained shifted low half at another selector multiplied by four plus 0x14.
- Load a word at the second shared-source sample plus 0x14.
- Pass the scaled selector to the next dependency.

A later dependency returns another paired result. Before forwarding it:

- Read a byte at the retained full word from the preceding phase shifted right by 23, plus 1.
- Store the low byte of the newly loaded word at the lookup byte’s value plus 0x11.
- Invoke a dependency with the paired high half repeated three times and the low half shifted left by two.

Following more calls:

- Mask that paired low half with 0x3fffffff, shift right by 21, and add 1.
- Read a byte at that address.
- Store the low byte of the newly loaded word shifted right by 22 at the lookup byte’s value plus 0x11.
- Pass another primary result shifted left by two to a dependency.

Both lookup addresses lie between 1 and 0x200 inclusive. The lookup bytes themselves are fresh reads.

### 8. Final writes and terminal call

More calls forward a new primary result shifted left by two in two positions. Following additional calls, a returned word is forwarded unchanged.

The final unresolved dependency produces an eight-bit primary result and a secondary writable byte pointer. Then, in order:

1. Store one zero byte through that pointer.
2. Read the separate three-byte global contribution and assemble a 32-bit value with that contribution in bits 31–8 and the returned byte in bits 7–0.
3. Store the assembled value into the shared output word.
4. Store a 16-bit zero at the retained late paired low half masked with 0x3fffffff and shifted right by seven.
5. Invoke **F-Z000**, address zero, with three 32-bit zero arguments.
6. Return only if F-Z000 returns.

The halfword destination ranges from zero through 0x7fffff and need not be even.

The three-byte contribution is consumed after the pointer store. If they overlap, the contribution reflects that preceding store. The subsequent halfword store and F-Z000 may also affect memory already written.

## Side effects

Direct effects comprise 8-, 16-, and 32-bit stores, repeated memory reads, and unresolved dependency calls. The shared output word has two direct 32-bit writes: the earlier returned word and the final assembled value.

There is no visible allocation, release, ownership transfer, synchronization, rollback, or cleanup protocol. This does not constrain what dependencies may do.

Computed destinations may overlap one another, globals, or dependency-accessed memory. A completed write is an operation in the execution history, not a guarantee that its value remains present or observable afterward. Later direct stores, including the second output-word write, may overwrite it; dependencies may modify the same memory.

No error checking, bounds checking, null checking, retry, or alternate failure path is represented. A fault or nonreturning call stops progress along this path. No mechanism is shown that undoes earlier operations, but their surviving memory effects depend on intervening stores and dependency behavior.

## Dependencies

- **F-U001:** target calculated from the incoming link value plus 0xff2; receives the explicit argument.
- **F-U002:** returns a word.
- **F-U003:** receives that word unchanged.
- **F-U004–F-U006:** three intervening calls without displayed arguments.
- **F-U007:** returns the first paired result.
- **F-U008:** receives its shifted low half, unchanged high half, and the hidden-base word load.
- **F-U009 onward:** subsequent unresolved call occurrences, whose consequential arguments and results are described in order above.
- **F-Z000:** terminal target at address zero; receives three zero arguments.

Most later targets are calculated from embedded anchors and fixed displacements. These patterns do not establish function identity, a common calling convention, or semantic roles.

No dependency has an established global-preservation, ownership, lifetime, failure, or return guarantee. Whether F-Z000 is executable code, a trap, an invalid transfer, or an inaccurate recovered operation is unresolved.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Low-byte copy:** A field load returning 0x123456ab causes its associated byte-copy operation to write 0xab, not four bytes.

2. **Double shift:** Every unsigned 32-bit sample shifted right by 23 and then right by 23 yields zero. The corresponding word store writes zero, and the associated byte lookup uses address 1.

3. **Scaled versus unscaled:** An early secondary selector of 3 produces a payload store at 0x20 followed by a word load at 0x17. Validity and alignment remain environmental.

4. **Byte-derived destination:** A lookup byte of 0xfe selects destination 0x10f. The addition does not wrap to eight bits.

5. **Fresh repeated read:** If an intervening dependency changes a source word before its repeated low-byte copy, the second copy uses the newly loaded value.

6. **Stride distinction:** A selector of 3 selects 0x44 for the stride-16 store, rather than 0x20 for a stride-four store.

7. **Terminal nonreturn:** If F-Z000 does not return, the routine does not return normally. Its nonreturn does not itself undo earlier operations. It also does not guarantee that any earlier written value survives: later stores may have overwritten it, computed destinations may alias, and intervening dependencies or F-Z000 may modify the same memory.

8. **Two output writes:** If execution reaches both shared-output stores, both occur in order. The earlier returned-word value is not the routine’s last direct value written there. Even the second value need not survive subsequent overlapping writes or dependency behavior.

9. **Final overlap:** If the final writable pointer overlaps the three-byte contribution, final assembly uses that contribution after the zero-byte store. Any later overlapping halfword store remains a separate effect.

## Uncertainties

The following remain **documented_only**:

- Validity of the represented routine boundary.
- Actual dependency identities, calling conventions, and hidden inputs.
- Definitions and validity of incoming register state.
- Target endianness, alignment rules, signed overflow behavior, and fault semantics.
- Address-space mappings, including low addresses and address zero.
- Actual overlap among globals, computed accesses, and dependency-accessed regions.
- Ownership and lifetime of every accessed region.
- Whether low-address operations are genuine behavior or recovery artifacts.
- Application purpose and externally meaningful object layouts.
- Final memory contents after arbitrary unresolved dependency behavior.

No unresolved callee is assumed to preserve globals, retain or release memory, or return successfully.

## Evidence

**Established static evidence:** one explicit 32-bit input; the represented dependency sequence; byte, halfword, and word access widths; shifts, masks, scaling, and displacements; two shared-source reads; two shared-output writes; and the terminal zero-address call.

**Derived consequences:** the double right shift yields zero; the documented masked-address ranges follow from unsigned widths; byte stores truncate; and equal retained values do not imply fresh memory reads.

**Inferred intent:** repeated forwarding, extraction, copying, and clearing patterns. These observations do not establish domain semantics.

No runtime execution, tests, independent instruction validation, memory-map validation, or dependency inspection has been performed. This contract describes conditional behavior supported by the supplied static representation, not demonstrated executable behavior.
