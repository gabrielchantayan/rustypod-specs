<!-- Trusted provenance: F-e5a48907bbdb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e5a48907bbdb

## Purpose

F-e5a48907bbdb accepts two 32-bit inputs, performs an extensive ordered sequence of indirect dependency invocations, transforms dependency results, and then reads and writes memory using computed addresses and fixed low addresses. Its last represented action is an unresolved indirect transfer with four explicit arguments.

The application-level purpose is not established. Constants resembling characters do not establish text formatting, and recurring offsets do not establish an object type, protocol, or buffer layout.

This contract distinguishes three evidence levels:

- **Established static evidence:** interfaces, operations, widths, and ordering explicitly visible in the supplied representation.
- **Conditional consequences:** conclusions requiring a stated arithmetic or memory model.
- **documented_only:** behavior that the supplied material cannot resolve.

No runtime tests have been performed.

## Interface

### Explicit inputs

| Input | Width and interpretation | Established use |
|---|---|---|
| First input | Uninterpreted 32-bit value | Forwarded unchanged to the first indirect dependency. |
| Second input | Signed 32-bit integer | Forwarded unchanged to the first dependency and used in the opening left-shift calculations. |

Neither input is independently established as a pointer, length, object handle, or flags field. Any additional argument requirements imposed by a dependency are **documented_only**.

### Implicit inputs

Execution also depends on:

- An incoming link-related execution value used to form the first call destination.
- Subsequent indirect destinations derived from retained continuation-related values.
- Primary and secondary dependency results.
- Results represented as 64-bit quantities.
- Existing memory at computed addresses and fixed low addresses.
- A separate global word later loaded as an address base.
- The unresolved final transfer destination.

Memory and execution state are therefore part of the effective input. The function is not established as a pure transformation of its two explicit arguments.

### Outputs and completion

The represented interface has no explicit return value. Observable outputs comprise dependency invocations, memory accesses, state changes caused by dependencies, and the final indirect transfer.

A return is represented after that transfer is treated as a call. Whether the transfer returns, and whether control subsequently returns to the original caller, remain **documented_only**.

There is no visible status code, output-length result, validation result, or structured error translation.

### Execution requirements

Reaching the represented end requires executable indirect destinations, usable dependency results, and permission to perform every reached memory access at its represented width. Alignment, address mapping, and arithmetic requirements depend on the actual execution environment.

These are preconditions for successful execution, not conditions the function visibly checks. In particular, the body does not establish protection against low-address accesses, invalid addresses, expired storage, or dependency failures.

## Behavior

### Overall sequencing

The visible body contains a long indirect-call region followed by a direct-memory region and the final unresolved transfer. No explicit conditional branch, retry loop, bounds check, or early-return path appears in the supplied representation.

That observation does not establish that dependencies always return or that the represented function boundary is reliable.

Invocation order and access order matter. Calls lacking displayed arguments are still invocations. Duplicate writes remain represented writes, even when their values agree. Neither may be removed merely because a local result appears unused.

### Opening dependency interactions

The second input is initially shifted left by six positions in a signed 32-bit expression. The first dependency receives the unchanged first input, unchanged second input, and this shifted quantity, in that order.

Two more invocations follow. The third invocation supplies a primary result used by the fourth: that result is shifted left by eight, while the input-derived quantity is shifted left by another six positions.

A later opening invocation receives four explicit arguments: zero, a secondary result from the preceding invocation, the twice-shifted input-derived quantity, and zero. The secondary result is an actual dependency; it is not established as zero or as the preceding primary result.

### Result widths and transformations

Later arguments depend on primary results, secondary results, and halves of represented 64-bit results. Transformations include:

- Left shifts with counts reaching 31.
- Explicitly unsigned right shifts.
- Masks selecting low-order fields.
- Extraction of low and high 32-bit halves.
- Reuse of one transformed quantity in multiple argument positions.
- Explicit zero arguments interspersed with result-derived arguments.

The high half of a 64-bit result comprises bits 32–63. An unsigned whole-result right shift by 34, 36, or 44 consequently corresponds to shifting that high half right by 2, 4, or 12. These are not oversized shifts of a 32-bit operand.

Visible masks retain fields including 24, 22, 21, 20, 19, 18, 17, 16, 15, 14, 13, 12, and 11 bits. Their placement matters: masking before a shift is not interchangeable with masking afterward.

Some quantities begin at zero and are subsequently only shifted. They remain zero under ordinary bit-vector semantics, but this does not eliminate the intervening dependency invocations.

### Arithmetic model

The representation distinguishes signed 32-bit expressions, unsigned 32-bit expressions, uninterpreted words, and 64-bit results.

The contract does not automatically widen 32-bit intermediates. Nor does it equate signed multiplication with signed left shifting outside an explicitly stated bit-vector interpretation. Signed overflow, signed left shifting, address overflow, and conversions between signed and unsigned representations need machine-specific resolution.

A left shift followed by a right shift does not recover discarded high bits.

When conditional examples below use modulo-\(2^{32}\) arithmetic, every relevant 32-bit intermediate is truncated to that width. This is an example condition, not an independently established rule for every signed expression.

### Distinct values entering the direct-memory region

The following descriptive labels distinguish values with different origins:

| Label | Established derivation |
|---|---|
| **High-shift base** | The low 32-bit portion of a retained dependency result, shifted left by 25 in a represented signed expression, then retained as an unsigned 32-bit quantity. |
| **Packed base** | A different retained signed 32-bit primary result multiplied by 0x02000000 using signed 32-bit arithmetic. |
| **Later eight-shift value** | Another retained primary result shifted left by eight. |
| **Field index** | A separate retained result shifted left by eight, masked with 0x7fff, then shifted right unsigned by six. |
| **Final pair** | The low and high 32-bit halves of the last represented 64-bit dependency result before direct memory accesses. |

The packed base is established as a multiplication result, not as a left-shift result. Multiplication by 0x02000000 and shifting left by 25 produce the same bit pattern only under a suitable explicitly assumed 32-bit bit-vector model.

The high-shift base and later eight-shift value must not be conflated.

### Opening direct-memory effects

The first word store places the **high-shift base** at the address formed from the final pair’s high half plus 0x14. The next byte store writes the low byte of that high half at the high-shift base plus 9.

A word store then places the packed base at 256 times the final pair’s low half plus 0x54. After that store, the byte at the high-shift base plus 9 is overwritten with the low byte of the high-shift base.

Thus, both opening byte stores use the high-shift base as their address base. The later eight-shift value is not their base or the value of the first word store.

The next access interval establishes an important readback invariant:

- A zero word is written at the packed base plus 0x14.
- A byte is read at the packed base plus 5.
- A zero word is written at twice the field index plus 0x14.
- The byte just read is zero-extended and written as a word at the packed base plus 0x34.
- A word is read at the packed base plus 0x14.

Another zero word is then written at twice the field index plus 0x14. The low byte of the retained readback word is written twice at twice the field index plus 0x15.

Under ordinary byte-addressed memory with 32-bit modulo addressing, no concurrent modification, and no device-induced effects, that readback word is zero. The intervening computed store writes only zero bytes, even if it overlaps the readback word. The potentially nonzero byte-derived word is written at packed base plus 0x34, 0x20 bytes away from the readback word; these four-byte regions do not overlap under this model.

Device-memory behavior remains a separate uncertainty. It is not a reason to claim that ordinary aliasing makes this readback nonzero.

### Fixed-address read and subsequent address dependencies

After the duplicated byte stores at twice the field index plus 0x15, the function reads the byte at fixed address **0x05** and zero-extends it. This read precedes the byte write at fixed address **0x01**.

The byte read at 0x05 becomes an address base. Subsequent accesses include a byte write at that base plus 0x15 and a word read and overwrite at that base plus 0x54.

Its value is a read-time value, not necessarily the byte present on entry. Earlier computed writes may alias address 0x05. A conforming interpretation must account for those preceding writes before determining this address base.

The later eight-shift value now participates in stores, including a byte store at packed base plus 0x11 and a word store through a readback-derived address plus 0x54. The byte location is subsequently overwritten with zero. These uses do not retroactively change the bases of the opening stores.

### Computed memory layout and middle-region effects

The remaining direct accesses combine retained values, newly loaded words, and zero-extended bytes as address bases. Repeated offsets are byte offsets, not proof of a shared structure type.

Visible widths include:

| Width | Representative uses |
|---|---|
| 8 bits | Offsets 1, 5, 8, 9, 0x0d, 0x10, 0x11, 0x15, and 0x19; fixed low addresses. |
| 16 bits | Destinations formed by multiplying computed values by 2 or 8. |
| 32 bits | Offsets 4, 0x10, 0x14, 0x20, 0x24, 0x34, 0x40, 0x44, 0x54, 0x64, and 0x74. |

Byte stores retain the low eight bits. Explicit unsigned byte reads are zero-extended when used as word values. Halfword stores affect 16 bits.

A word loaded from the address based on the byte at 0x05 is retained before that location is overwritten. It drives later scaled bases, self-valued stores, and accesses at offsets 0x24, 0x44, and 0x54. A separate chain reads a byte and then a word at that byte-derived base plus 0x74. The byte-derived address is not established as a conventional table index or object identifier.

Several locations receive successive distinct values, and some byte locations are written twice. Later loads must use memory after preceding writes, including any overlapping writes.

One segment writes 0x3a as a byte, as a halfword at a differently scaled destination, and as a word at a related base plus 0x20.

Another segment writes to one word destination at offset 0x20 in this order: 0x20, 0x2f, a zero-extended byte loaded from that base plus 4, and 0x20. The byte load occurs after the first two stores. Under ordinary modulo byte addressing, those particular offsets do not overlap each other; external or device effects remain unresolved.

### Global-base acquisition and repeated stores

After the four-value overwrite segment, the function reads a separate global word and retains it as an address base. **That global read occurs before the word store of 0x20 at fixed address 0x60.**

A long repeated-store region follows. Each group orders its effects as:

1. A word write at a scaled retained base plus 0x54.
2. A byte write at the retained global-derived base plus 0x19.
3. A byte write at another computed base plus 0x11.
4. A word write at the global-derived base plus 0x34.

The first group differs from the following groups in both the scaled base for its first store and the value of its last store. In descriptive terms, the first group uses eight times an earlier retained word; following groups use eight times an adjusted quantity formed by subtracting 0x96 from four times that retained word.

The global address base is retained across these groups; the representation does not show a fresh global-base load for each group. This retention does not imply that the global storage itself remains unchanged.

The repeated region is an ordered write sequence, not an established idempotent initialization. Its machine-level repetition count is **documented_only** without independent validation.

### Late reads, overlapping globals, and completion

**The word at fixed address 0xa9 is read after the repeated-store region.** It must not be treated as an entry-state value or moved before those stores when computed destinations could affect its storage.

Subsequent reads retain a word from a computed base plus 0x74, a word from another base plus 0x14, and a word from a further base plus 4. These retained values survive later overwrites of related memory.

The ending region includes:

- A fixed byte write at 0x3a.
- A byte write of 0x6c.
- Byte copies using low portions of retained values.
- A byte write at 0x67 using the low byte of the word read at 0xa9.
- A later byte read at 0x67 whose value is written to 0x21.
- Two further reads at 0x67, each supplying a computed byte store, with computed writes between the reads.

Those reads at 0x67 are distinct read events. Computed writes between them may overlap 0x67, so they must not silently become copies of one cached original byte.

A signed right shift late in the region uses a count taken from a computed address’s low eight bits. Its count is not visibly restricted to 0–31. Negative-operand behavior and out-of-range counts require machine-specific resolution.

Before the final transfer, the function stores the halfword 0x65 through a doubled loaded value, stores the word 100 at a computed destination plus 0x54, and writes another word through an address formed by doubling a subsequently loaded word.

The final transfer receives, in order:

- 0x72.
- A retained word loaded from offset 0x64 of a computed base.
- The earlier retained word loaded from offset 0x14.
- A word loaded from offset 0x74 of the same base used for the offset-0x64 load.

There is no visible rollback or final consistency check.

## Side effects

Direct effects include computed-address reads and writes, fixed-address storage changes, and the final transfer. Established fixed locations include a byte read at 0x05; byte writes at 0x01, 0x15, 0x21, 0x3a, and 0x67; word writes at 0x54 and 0x60; and a late word read at 0xa9.

Global symbols are not proven independent storage objects. Overlapping-width accesses and computed aliases must be interpreted through shared memory, with endianness and mapping supplied by the execution environment.

No recognizable allocation or release is established in the direct body. Dependencies may nevertheless allocate, release, retain, or invalidate memory. No ownership transfer, borrowing guarantee, or lifetime guarantee is established for either input or any result.

If a dependency does not return or an access faults, earlier effects are not visibly undone. Fault classification, exception propagation, synchronization, thread safety, reentrancy, and atomicity remain **documented_only**.

## Dependencies

The supplied mapping provides only the opaque identity **F-e5a48907bbdb**. No opaque identities are available for the indirect callees; assigning them would invent provenance.

Unresolved dependencies include the entry-derived target, the continuation-derived invocation sequence, and the final transfer target.

Their concrete identities, complete argument conventions, secondary-result conventions, global effects, ownership behavior, failure behavior, and return behavior are **documented_only**. An invocation without displayed arguments is not proof of a zero-argument interface.

No unknown dependency is assumed to preserve globals or preserve the lifetime of memory used later.

## Acceptance examples

These are conditional static obligations, not executed tests.

### Input forwarding and width

For a second input of 1, the initial representable six-position shift produces 64. The first dependency receives the original two inputs followed by 64.

Under 32-bit modulo arithmetic, a second-input bit pattern of 0x04000000 instead produces zero for that third argument. An untruncated 64-bit value of 0x100000000 would not satisfy that conditional model.

### Distinct direct-memory values

Assume 32-bit bit-vector arithmetic and suitable valid memory. If the result supplying the high-shift base has low half 1, that base is 0x02000000. If the independent result supplying the later eight-shift value is 2, that later value is 0x200.

For a final-pair high half of 0x1234, the first direct word store writes 0x02000000 at 0x1248. The first byte store at 0x02000009 writes 0x34; the later overwrite there writes zero. Substituting 0x200 as the base or first word value is incorrect.

### Packed-base arithmetic

A retained signed result of 3 produces packed base 0x06000000 through multiplication by 0x02000000; this product is representable as signed 32-bit.

For overflowing products, equivalence to a 25-position left shift requires the stated bit-vector assumption. Portable signed overflow behavior is not established.

### Field-index boundary

Under 32-bit bit-vector semantics, the field index ranges from 0 to 508 in multiples of four. A retained result of 1 produces 4; 127 produces 508.

### Zero readback despite overlap

Assume ordinary byte-addressed modulo memory, valid accesses, and no concurrent or device-induced changes. The first word read at packed base plus 0x14 returns zero even when the intervening computed zero store overlaps it. The byte-derived store at packed base plus 0x34 cannot invalidate this conclusion.

### Read-time low-address state

If an earlier computed write changes address 0x05, the later fixed byte read uses that changed byte as its address base. Using the entry byte would violate ordering.

Likewise, if a computed write between two reads at 0x67 changes that byte, the later read must reflect the changed memory under ordinary memory semantics.

### Partial execution

If an opening dependency never returns, later direct stores need not occur. Earlier dependency effects have no established rollback guarantee.

## Uncertainties

The following remain **documented_only**:

- Application-level purpose and valid input domain.
- Concrete dependency identities and complete interfaces.
- Reliability of the represented boundary and straight-line control flow.
- Signed overflow, signed shifts, address wraparound, and variable-shift behavior.
- Endianness, alignment, mappings, buffer extents, and object layouts.
- Device-memory effects and concurrent modification.
- General alias relationships among computed addresses and globals.
- Ownership, allocation, release, retention, and memory lifetime.
- Machine-level repetition multiplicity.
- Final-transfer return behavior and externally reported errors.

General alias uncertainty does not negate the conditional zero-readback proof above. Conversely, that local proof does not justify simplifying later reads or assuming global preservation.

## Evidence

This contract is based exclusively on static inspection of the supplied representation, with prior prose and feedback treated as claims to check rather than authoritative behavior.

The visible evidence distinguishes the high-shift base, signed multiplication-derived packed base, later eight-shift value, field index, and final pair. It also establishes the fixed read at 0x05, the global-base read before the store at 0x60, the read at 0xa9 after the repeated region, and separate later reads at 0x67 amid computed writes.

No dependency implementation inspection, independent machine-level validation, runtime execution, or testing was performed. The acceptance examples express conditional consequences of the static evidence, not observed execution results.
