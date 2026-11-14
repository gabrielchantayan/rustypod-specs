<!-- Trusted provenance: F-ec7bf836be5a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ec7bf836be5a

## Purpose

F-ec7bf836be5a performs an ordered sequence of indirect dependency invocations, indexed word transfers, field extractions, selector-directed byte updates, and global assignments. Its terminal sequence clears a dependency-selected byte, constructs a global word from a three-byte source and a returned byte, clears a computed halfword, and invokes address zero.

No application-level purpose is established. Scaling, masks, and recurring offsets suggest encoded references or compact records; this is **inferred intent**, not an established layout.

This contract describes static evidence only. It does not establish that the apparent entry boundary is correct, that execution reaches every operation, or that the routine completes.

## Interface

### Effective inputs and outputs

The apparent interface has no explicit parameters. Effective inputs include:

- Incoming link-related state determining the first indirect target.
- An incoming unsigned 32-bit **entry reference**.
- Ordinary dependency results, paired results, and secondary register outputs.
- Two global word locations, **G-A** and **G-B**.
- Selector bytes, including repeated reads at absolute address 1.
- A three-byte source beginning at absolute address 9.
- Words and bytes at computed addresses.

No ordinary return value is established. Observable outputs consist of direct memory writes, dependency invocations and arguments, and whatever effects those dependencies produce.

Input initialization, register conventions, accessibility, ownership, and lifetime are unknown. The routine performs no visible validation.

### Widths and addressing

The representation distinguishes 8-bit bytes, 16-bit halfwords, 32-bit words, and 64-bit paired results. A paired result is described by its low and high 32-bit halves.

Here:

- **word(A)** means a 32-bit access at byte address A.
- **byte(A)** means an 8-bit access at byte address A.
- **halfword(A)** means a 16-bit access at byte address A.
- **low8(V)** means the least significant eight bits of V.

Addresses and offsets are in bytes. Multiplication by four is explicit; an expression lacking that multiplication is not implicitly word-indexed.

Unsigned right shifts are logical. A 32-bit unsigned value shifted right 22 bits lies in 0–1023; shifted right 23 bits, it lies in 0–511. Byte writes retain only eight bits.

Several left shifts, multiplications, and additions involve signed-looking 32-bit expressions. Their overflow and signed-shift semantics remain unresolved. Conclusions expressly conditioned on **32-bit wrapping arithmetic** use arithmetic modulo 2³² and do not resolve those expressions’ execution semantics.

### Preconditions and completion

Reaching any operation requires every preceding call to return and every preceding access to complete. Memory mappings, permissions, alignment requirements, and valid call targets are not established.

An explicit return follows the address-zero invocation. Ordinary completion depends on that invocation returning.

## Behavior

### Sequencing and observations

The visible path has no explicit branch or loop.

A **captured** value is retained from its stated observation point; later uses do not imply rereading its source. Explicit repeated reads are fresh observations. Intervening dependencies are not assumed to preserve memory, globals, or externally observable state.

The following stages preserve direct-access order and significant dependency relationships. Groups described as intervening calls remain part of the sequence; their effects and implicit inputs are unknown.

### 1. Initial word propagation and extraction

Three initial indirect invocations occur. The third returns **initial result**.

The routine writes initial result shifted left two bits to word(20 + 4 × (entry reference shifted right one bit)). It then captures **first source word** from word(entry reference + 20), before invoking a dependency with entry reference shifted right one bit.

These first two addresses use different scaling rules.

A later returned word is forwarded to another dependency. Additional invocations produce **initial index**. The shifted initial result is written to word(20 + 4 × initial index). G-A is then read and captured as **first global snapshot**, followed by a dependency invocation with 4 × initial index.

After further calls, the routine derives:

- **First extracted field:** first source word shifted right 22 bits.
- **Global upper field:** first global snapshot shifted right 23 bits.

It stores first extracted field at word(20 + 4 × a secondary output). The following dependency receives that secondary output twice, then global upper field and first extracted field.

Both fields use captured words rather than fresh reads.

### 2. Repeated field stores and selector updates

A dependency-produced word is forwarded. The routine also derives **doubly extracted field** by shifting global upper field right another 23 bits. This field is necessarily zero because global upper field is at most 511.

Further calls produce an index. First extracted field is written to word(20 + 4 × that index), followed by a call with the scaled index.

Another call group supplies an unsigned secondary output, **masked selector**. The routine writes zero to word(20 + 4 × masked selector), then invokes a dependency with masked selector itself, not its scaled value.

After result forwarding and more calls, another index is obtained. Global upper field shifted right 23 bits is written to word(20 + 4 × that index), making this another zero store.

Before the next call, the routine captures a word at:

**20 + ((masked selector AND 0x1fffffff) shifted right 19 bits).**

There is no additional multiplication by four. The offset above 20 is in 0–1023 and need not be word-aligned. The following dependency receives four times the most recently obtained index.

After intervening calls, the captured word is shifted right 22 bits to obtain **selector field**, in 0–1023. The routine reads byte(doubly extracted field + 1), then writes low8(selector field) to byte(17 + that selector byte). Because doubly extracted field is zero, the selector read is at absolute address 1.

A dependency then receives one secondary word output repeated in its first three visible argument positions, followed by zero. Later invocations include one supplied with zero.

After additional calls, byte(4 × selector field + 1) is read and byte(17 + the resulting selector byte) is cleared. This selector-read address belongs to 1, 5, …, 4093.

### 3. Absolute selectors, publication, and a captured byte base

Further dependencies receive scaled results, duplicated scaled results, and forwarded results. The routine then freshly reads byte(1) and clears byte(17 + that value).

A secondary output is captured as **saved shift operand** and passed three times to a dependency. A later result is passed to another dependency together with a retained unsigned result shifted right 23 bits.

After more calls, the routine freshly reads byte(1), capturing **saved byte base**, and clears byte(saved byte base + 17).

Calls using scaled and duplicated arguments follow. A returned **published word** is written to G-B and then passed to a dependency. After another call, the routine reads word(saved byte base + 116) and writes its low byte to byte(saved byte base + 1).

Saved byte base is in 0–255. Its use here does not reread byte(1), even if dependencies changed that location.

The routine next invokes a dependency with saved shift operand shifted left 11 bits. A subsequent paired result supplies its low and high words to another dependency, whose third visible argument is published word shifted right 23 bits. This argument uses the captured published word, not a fresh read of G-B.

### 4. Repeated word-to-byte transfers and redistribution

A returned index multiplied by four becomes **copy base**. The routine reads word(copy base + 116) and writes its low byte to byte(copy base + 1).

After a call with copy base and additional invocations, another result is captured as **link index**. Shifting it left two bits forms the unsigned **link reference**.

The transfer at copy base is then repeated: word(copy base + 116) is freshly read, and its low byte is written to byte(copy base + 1). Calls separate these transfers, so their observed values can differ.

The next dependency receives:

1. Saved shift operand shifted left 12 bits.
2. Link reference.
3. Link reference again.
4. Link index shifted left 18 bits.

Subsequent result-forwarding calls produce another four-times index, **redistribution base**. The routine captures **redistributed word** from word(redistribution base + 116), then writes its low byte to byte(redistribution base + 1).

A dependency receives redistribution base. Further calls produce an index, and redistributed word is stored at word(20 + 16 × that index). This store has a sixteen-byte stride.

The following dependency receives link reference logically shifted right two bits and then shifted left 16 bits.

Under 32-bit wrapping arithmetic, this argument equals link index shifted left 16 bits. Both results depend only on the original low 16 bits. Loss of the original upper two bits when forming link reference does not make these final bit-vector results unequal. This equivalence does not establish equivalent definedness for the signed-looking expression that originally forms link reference.

After more calls, redistributed word is stored at word(20 + 4 × another returned index), followed by a dependency invocation with the scaled index.

A subsequent result shifted left two bits forms **replacement reference**. It is written to word(20 + (link reference AND 0xfffffffc)), then passed twice to a dependency.

### 5. Paired references and final selected-byte writes

After result forwarding and further calls, zero is written to word(20 + 4 × another returned index). A dependency then receives that scaled index.

Further invocations supply a paired result:

- Its low word shifted left two bits forms **paired reference**.
- Its high word supplies **paired index**.

Paired reference is written to word(20 + 4 × paired index). G-A is then freshly read as **second global snapshot**.

A dependency receives paired reference, paired index, paired reference, and paired reference. After additional calls, paired reference is stored at word(20 + 4 × another returned index).

The routine then captures **final source word** from word(second global snapshot + 20). The snapshot is used directly as a byte-address base, without multiplication by four.

After intervening calls, another paired result supplies **final low word** and **final high word**. The routine reads byte((redistributed word shifted right 23 bits) + 1), then writes low8(final source word) to byte(17 + that selector byte).

A dependency receives final high word three times and final low word shifted left two bits as its fourth visible argument.

Following more calls, including result forwarding, the routine reads:

**byte(((final low word AND 0x3fffffff) shifted right 21 bits) + 1).**

It writes low8(final source word shifted right 22 bits) to byte(17 + the resulting selector byte).

Both selector-read addresses are in 1–512. Selected destinations are in 17–272. Final source word remains captured across the intervening dependencies.

### 6. Terminal effects and global overlap

Further invocations include scaled arguments, duplicated scaled arguments, and result forwarding. The last result-producing dependency supplies **terminal byte**, an 8-bit result, and a secondary pointer output.

The terminal effects occur in this order:

1. Clear the byte addressed by the secondary pointer output.
2. Read the three-byte source beginning at absolute address 9 and assign G-B a word whose upper 24 bits are that source quantity and whose lower eight bits are terminal byte.
3. Clear halfword((final low word AND 0x3fffffff) shifted right seven bits).
4. Invoke address zero with three visible zero-valued arguments.
5. Return if that invocation returns.

The pointer-selected clear precedes the three-byte source read. If they overlap, that clear can change the source quantity subsequently used in the G-B assignment.

No explicit whole-word read of G-B is shown in constructing this assignment. Its upper 24 bits come from the three-byte source read after the pointer clear. Whether this preserves G-B’s then-current upper bytes depends on the unresolved physical overlap between G-B and that source. The assignment therefore cannot be characterized categorically as either preserving or replacing those bytes with unrelated storage.

The halfword destination is in 0–0x007fffff. Neither nonzero addressing nor alignment is ensured. If it overlaps G-B or other earlier destinations, it can modify memory after the terminal G-B assignment.

## Side effects

Direct effects include indexed word stores, word-to-byte transfers, selector-directed byte writes and clears, two reads of G-A, two assignments to G-B, and a final halfword clear.

No direct allocation, release, ownership transfer, error construction, or rollback is established. Dependency effects remain unknown.

Aliasing is not excluded among computed addresses, globals, selected bytes, and the terminal source. Later reads observe then-current memory, including prior direct writes and dependency effects. The static representation warns that larger global symbols overlap smaller symbols, but does not resolve the complete layout.

Numeric low-byte extraction does not establish memory endianness. Likewise, the numeric 24-bit terminal source does not establish the ordering of its constituent bytes. Physical transaction widths, atomicity, concurrency behavior, and tearing are unknown.

There is no visible bounds check, null check, alignment check, failure branch, or recovery path. A fault or nonreturning dependency can leave earlier effects committed while preventing later effects.

## Dependencies

Unresolved invocation occurrences are designated **F-000001**, **F-000002**, and onward in visible execution order, including the terminal invocation. These opaque IDs distinguish occurrences, not proven distinct implementations.

- **F-000001** derives its target from incoming link-related state.
- **F-000002** and **F-000003** use fixed address-derived target expressions.
- **F-000003** supplies initial result.
- Subsequent occurrences mostly use fixed address-derived expressions.
- The terminal occurrence targets address zero.

Different target expressions do not prove different implementations. Calls shown without explicit arguments can still consume implicit state.

Established interactions include forwarded word results, split low/high paired results, secondary outputs used as indices or pointers, and repeated argument values. Visible argument lists do not establish a complete calling convention.

No dependency is assumed to preserve globals, preserve addressed memory, acquire ownership, release memory, or return successfully.

## Acceptance examples

These are conditional static examples, not executed tests. Each assumes preceding calls return, required accesses succeed, and arithmetic has the stated interpretation.

1. **Different initial address rules.** With entry reference 8 and initial result 3, the first store writes 12 at address 36. The following source read is at address 28.

2. **Ten-bit extraction.** First source word 0xffc00000 yields first extracted field 1023. Word stores retain 1023; byte truncation is a separate operation.

3. **Double extraction is zero.** Every unsigned 32-bit first global snapshot yields zero after two successive logical right shifts by 23. The associated stores contain zero, and the corresponding selector read uses address 1.

4. **Selected-byte truncation.** With selector field 1023 and selector byte 6, the selected destination is address 23 and receives 255.

5. **Fresh transfer source.** If word(copy base + 116) is first 0x11223344 and later 0xaabbccdd, the two destination-byte writes are 0x44 and 0xdd.

6. **Sixteen-byte stride.** Redistribution index 3 places the sixteen-byte-stride store at address 68, not address 32.

7. **Conditional wrapping equivalence.** Under 32-bit wrapping arithmetic, link index 0xc0001234 forms link reference 0x000048d0. Shifting that reference right two bits and then left 16 yields 0x12340000, equal to directly shifting the original index left 16 under the same model. Signed-expression definedness remains unresolved.

8. **Independent global snapshots.** If G-A changes between its reads, initial field extraction still uses the first snapshot; final-source addressing uses the second.

9. **Terminal composition.** If the three-byte source observed after the pointer clear is numerically 0x123456 and terminal byte is 0xab, the terminal assignment writes 0x123456ab to G-B.

10. **Conditional upper-byte preservation.** If the three-byte source physically corresponds to G-B’s upper bytes with the required byte ordering, terminal composition can preserve those bytes as observed after the pointer clear. If that clear changes an overlapping source byte, the assignment incorporates the changed value. Disjointness and this correspondence are not established.

11. **Low and unaligned halfword destinations.** A masked final low word below 128 produces halfword destination zero; a value of 128 produces destination 1. Neither access is guaranteed valid.

## Uncertainties

The following remain **documented_only**:

- Application purpose, record interpretation, and apparent entry-boundary correctness.
- Incoming register provenance and complete calling conventions.
- Dependency identities, implicit inputs, preservation, ownership, and return behavior.
- Memory mappings, permissions, alignment, atomicity, and concurrency rules.
- Signed arithmetic, overflow, and address-calculation semantics.
- Endianness and complete physical overlap among globals and smaller objects.
- Whether the terminal three-byte source represents G-B’s then-current upper bytes.
- Ownership and lifetime of addressed storage.
- Validity of low absolute addresses and the address-zero invocation.
- Reachability of the terminal sequence and ordinary return.

The wrapping equivalence does not resolve signed-expression semantics. The absence of an explicit whole-word G-B read does not resolve whether its upper bytes are preserved through overlapping source storage.

## Evidence

Established static evidence comprises the visible invocation sequence, access widths, constants, masks, unsigned shifts, argument expressions, captured-value uses, and direct assignments.

Range bounds, the double-extraction zero result, and the conditional wrapping equivalence follow from the stated integer widths.

The terminal source is read after the pointer-selected byte clear. The assigned word combines that 24-bit quantity with terminal byte. Static evidence does not establish disjointness between the source and G-B’s upper bytes.

Encoded-reference and compact-record interpretations are inferred intent only. Valid execution, dependency preservation, complete storage layout, and successful completion are not established. No runtime tests have been performed.
