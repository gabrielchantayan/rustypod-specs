<!-- Trusted provenance: F-12decedaab4b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-12decedaab4b

## Purpose

F-12decedaab4b repeatedly invokes indirect dependencies and performs ordered memory updates using fixed locations, ambient state, dependency results, and narrow values subsequently used as addresses. Each iteration includes software breakpoint `0x92`, a decrease of frame-like state, and a final assignment to link-like state. There is no represented normal return or termination condition.

The established behavior is low-level state mutation. Initialization, message handling, object management, and resource management are not established purposes. Terms such as “owner address” describe access relationships only; they do not establish object types, ownership, allocation, or lifetime.

This contract distinguishes statically established operations from conditional arithmetic deductions and unresolved execution behavior. It does not establish that an iteration can complete in an actual environment. No runtime tests have been performed.

## Interface

### Invocation and completion

There are no explicit entry parameters or ordinary return value. Inputs are supplied through ambient state, memory, and dependency outputs:

- A link-like value used to derive the first indirect target.
- A frame-like base used for **four distinct word-access locations**: stores at offsets `0x26C` and `0x258`, and reads at offsets `0x320` and `0x280`. That base is subsequently decreased.
- An incoming 32-bit register value contributing to a byte-store destination.
- Two globally stored address-like words.
- Memory reached through fixed addresses, captured values, and dependency outputs.
- Ordinary and secondary results of indirect dependencies.

Normal continuation is unconditional repetition. A nonreturning dependency, unsuccessful memory access, arithmetic operation without an established portable result, or breakpoint that does not resume may prevent later operations. There is no explicit local error result, validation branch, recovery branch, or rollback.

### Widths and layouts

| Quantity | Representation |
|---|---|
| Byte | 8 bits |
| Halfword | 16 bits |
| Word | 32 bits |
| Paired result | 64 bits, split into low and high 32-bit words |
| Unsigned halfword read | Zero-extended to 0–65,535 |
| Address-like state | Derived from 32-bit quantities; address validity is not established |

The following descriptive labels identify captured values, not implementation identifiers:

| Label | Origin |
|---|---|
| Primary address | Word read from `0x087120B4` |
| Table base | Word read from `0x08712120` |
| Entry address | Unsigned halfword read from `0x21C48994` |
| Owner address | Word read at entry address + `0x44` |
| Selected address | Word read from `0x2218` |
| Payload | Low word returned by F-000000000028 |
| Companion | High word returned by F-000000000028 |
| Selector | Unsigned halfword read at selected address + `0x34` |
| Attribute | Unsigned halfword read at companion + `0x14` |

These labels do not imply separate allocations, nonoverlapping storage, or valid pointers.

Byte and halfword stores write the corresponding low-width bits. Unsigned 32-bit arithmetic is modulo \(2^{32}\). Signed arithmetic is not automatically wrapping: signed overflow and invalid signed left shifts have no portable result assigned here. Endianness, alignment requirements, permissions, atomicity, and memory type are unresolved.

## Behavior

### Ordering and captured state

F-000000000001 through F-000000000037 identify indirect-call occurrences in order within one iteration. They do not establish 37 distinct implementations.

Every operation is conditional on preceding operations completing. Access boundaries matter: replacing a captured value with a fresh read, moving an access across a dependency, or merging separated writes may change behavior. Unknown dependencies are not assumed to preserve globals, registers, memory, or prior writes.

### Initial calls and result publication

Each iteration begins by resetting a carried byte to zero. F-000000000001 is then invoked with two zero arguments. Its target is the current link-like value plus `0xFFE`.

A separate unsigned arithmetic byte is reset to zero only after F-000000000001 returns, before F-000000000002. Its value before that reset is not established, and no explicit use of that prior value is shown.

The primary address is captured after F-000000000003 and before F-000000000004. Both subsequent primary-relative stores use this captured value.

F-000000000005 supplies a paired result. F-000000000006 receives:

- The low 32-bit word.
- A value formed by first adding the signed 32-bit interpretations of the high and low words, then adding the unsigned arithmetic byte.

The first addition is signed and must be representable for a portable result. The subsequent addition involving the unsigned byte is unsigned. The byte is zero at this point; that does not cure overflow in the preceding signed addition.

After F-000000000006, a secondary byte associated with that call is stored at primary address + `0x11`. After F-000000000007, the zero carried byte is stored at primary address + `0x15`.

F-000000000008 supplies a halfword stored at a secondary-output address + `0x0C`. The secondary-address convention is unresolved.

F-000000000009 supplies another paired result:

- Its high word supplies the address base for a zero-halfword store at offset `0x2C`.
- Its low word, logically shifted right by 26, is stored as a word at frame + `0x26C`.
- That same retained low word, shifted left by two using unsigned 32-bit arithmetic, is stored later through a separately captured destination.

### Frame access and breakpoint boundary

After F-000000000010 and before F-000000000011, a word is read from frame + `0x320`. That captured word supplies the destination base for the retained left-shifted result. The store at destination + `0x54` occurs after F-000000000012 and before F-000000000013.

After F-000000000017 and before F-000000000018, the entry address is read as an unsigned halfword from `0x21C48994`. Its numerical range is therefore 0–65,535. This is not a full-width pointer read from that location.

After F-000000000018, a word is read from frame + `0x280` and passed to F-000000000019. Its returned 32-bit value is retained across software breakpoint `0x92`.

No represented memory access or dependency call intervenes between F-000000000019 and the breakpoint. The subsequent signed shift belongs to post-breakpoint behavior. Reaching it requires the environment to resume the represented sequence.

### Indexed and selected-location updates

After breakpoint continuation, the retained result of F-000000000019 is shifted left by two with signed 32-bit semantics. Portable evaluation requires an operand from zero through `0x1FFFFFFF`.

A fixed derived value is `0x21C4`. The owner address is captured from entry address + `0x44` before F-000000000020.

F-000000000021 supplies an ordinary word result. After that call, a separate word is read from entry address + `0x04`. F-000000000022 receives the ordinary result first and the entry-relative word second.

The table base is captured after F-000000000022 and before F-000000000023. After F-000000000023, a halfword is stored:

- Destination: captured table base plus the signed shifted result.
- Value: low 16 bits of the captured table base itself.

This does not copy table contents. The destination addition uses signed operands, so overflowing addition has no portable modular interpretation under this contract.

After F-000000000024, the selected address is read from `0x2218`, obtained from the fixed derived value `0x21C4` plus `0x54`.

F-000000000025 supplies a destination address. The halfword stored at its offset `0x04` is `0x21C4`, not the selected address.

The selector is captured after F-000000000027 and before F-000000000028. It remains unsigned in its subsequent bounded calculations.

### Payload propagation and frame movement

F-000000000028 returns the payload and companion. The payload is stored as a word at frame + `0x258` before the attribute is read from companion + `0x14`.

Before F-000000000029:

- A payload-derived offset is formed by signed left shift of the payload by six.
- A companion-derived address is formed by signed multiplication of the companion by four.
- The frame-like base is decreased by `0xF4`.
- The payload is stored at `4 × selector + 0x74`.

The frame-relative payload store uses the base before its decrease. The selector-relative store follows the decrease but does not use the frame base.

Portable payload shifting requires a signed payload from zero through `0x01FFFFFF`. Signed companion multiplication is representable for companion values from −536,870,912 through 536,870,911. These arithmetic conditions do not establish address validity.

After F-000000000029, the zero-extended attribute is stored as a word at owner + `0x54`. F-000000000030 occurs before the same attribute is stored at owner + `0x64`. These are separate updates, not a combined or atomic publication.

After F-000000000031, a halfword containing the low 16 bits of the owner address is written to the sum of the payload-derived offset and a secondary address contribution. The convention supplying that contribution is unresolved. The signed addition must also be representable for a portable result.

After F-000000000033, one byte is captured from address `5 × selector`. F-000000000034 separates this read from the subsequent destination write. The destination is attribute + incoming register value + `0x19`, evaluated in the unsigned 32-bit domain.

After F-000000000035, the companion-derived address is stored as a word at owner + `0x44`.

After F-000000000036, link-like state is assigned `0x08712031`. A word is then read at companion-derived address + `0x44` and passed to F-000000000037.

### Repetition

Return from F-000000000037 leads unconditionally to the next iteration.

The carried byte is reset before the next first call. Consequently, the byte captured near the end of one iteration is not carried forward as the next primary-relative byte value.

No explicit restoration of the frame decrease appears. If dependencies and external mechanisms do not otherwise change that state, each reached decrease contributes another `0xF4`.

If the final link-like assignment remains effective through F-000000000037, the next first target is `0x0871302F`. Preservation and executable interpretation of that target remain unresolved.

## Side effects

### Direct memory writes

The following writes are ordered within an iteration:

| Destination | Width | Value | Boundary |
|---|---:|---|---|
| Primary + `0x11` | 8 bits | Secondary byte | After F-000000000006, before F-000000000007 |
| Primary + `0x15` | 8 bits | Zero | After F-000000000007, before F-000000000008 |
| Secondary destination + `0x0C` | 16 bits | Halfword result | After F-000000000008, before F-000000000009 |
| F-000000000009 high word + `0x2C` | 16 bits | Zero | Before F-000000000010 |
| Frame + `0x26C` | 32 bits | Retained low word logically shifted right by 26 | After the preceding halfword store |
| Captured destination + `0x54` | 32 bits | Retained low word shifted left by two, unsigned | After F-000000000012, before F-000000000013 |
| Table base + signed shifted result | 16 bits | Low 16 bits of table base | After F-000000000023, before F-000000000024 |
| F-000000000025 result + `0x04` | 16 bits | `0x21C4` | Before F-000000000026 |
| Frame + `0x258` | 32 bits | Payload | Before attribute read and frame decrease |
| `4 × selector + 0x74` | 32 bits | Payload | After frame decrease, before F-000000000029 |
| Owner + `0x54` | 32 bits | Zero-extended attribute | After F-000000000029, before F-000000000030 |
| Owner + `0x64` | 32 bits | Zero-extended attribute | After F-000000000030, before F-000000000031 |
| Payload-derived offset + secondary contribution | 16 bits | Low 16 bits of owner address | After F-000000000031, before F-000000000032 |
| Attribute + incoming register value + `0x19` | 8 bits | Captured byte from `5 × selector` | After F-000000000034, before F-000000000035 |
| Owner + `0x44` | 32 bits | Companion multiplied by four | After F-000000000035, before F-000000000036 |

Dependency effects, breakpoint handling, frame movement, and the link-like assignment are additional effects.

### Aliasing, ownership, and failures

Aliasing is unrestricted. A destination may overlap a later source, global storage, frame storage, or another destination. Later reads use the memory available at their represented read boundaries. Captured values are not automatically refreshed.

No allocation, release, reference-count operation, ownership transfer, or synchronization operation is directly established. Dependencies may perform such operations, but neither ownership nor memory preservation may be assumed. Obtaining an address establishes no lifetime or access right.

Zero addresses and zero values receive no special handling. Unmapped memory, misalignment, arithmetic failures, and dependency errors have no explicit local recovery. Previously reached effects are not locally rolled back; dependencies or the execution environment may nevertheless alter them.

## Dependencies

Identifiers are opaque call-occurrence labels. Calls occur in increasing identifier order. “No explicit arguments” does not exclude ambient inputs or effects.

| Occurrence | Explicit interface or consumed output |
|---|---|
| F-000000000001 | Receives two zeros; target derived from link-like state |
| F-000000000005 | Supplies paired result |
| F-000000000006 | Receives low word and computed sum; secondary byte consumed |
| F-000000000008 | Supplies halfword; secondary destination consumed |
| F-000000000009 | Supplies paired result retained for multiple stores |
| F-000000000019 | Receives frame-loaded word; supplies signed-shift operand |
| F-000000000021 | Supplies first argument for F-000000000022 |
| F-000000000022 | Receives preceding result and entry-relative word |
| F-000000000025 | Supplies destination address |
| F-000000000028 | Supplies payload and companion |
| F-000000000031 | Secondary address contribution consumed |
| F-000000000037 | Receives word read through scaled companion |

All remaining occurrences have no explicit arguments and no ordinary result consumed in the representation.

These occurrence pairs have equal represented targets:

- F-000000000005 and F-000000000006.
- F-000000000009 and F-000000000013.
- F-000000000017 and F-000000000024.
- F-000000000019 and F-000000000031.
- F-000000000025 and F-000000000032.

Different result uses do not establish different implementations. Complete signatures, secondary-output conventions, preservation rules, error conventions, and ownership behavior remain unknown. No dependency implementation has been inspected.

## Acceptance examples

These are conditional static deductions, not observed executions. Each assumes preceding operations complete and relevant arithmetic and memory accesses are supported.

1. **Four frame locations.** An iteration that reaches all represented frame accesses writes words at offsets `0x26C` and `0x258` and reads words at offsets `0x320` and `0x280`. The `0x258` store precedes that iteration’s explicit frame decrease.

2. **Reset ordering.** The carried byte is zero before F-000000000001. The arithmetic-byte reset follows that call. Moving both resets before the first call changes the represented ordering.

3. **Retained paired result.** If F-000000000009 returns low word `0xFC000003` and high word H, the derived stores include zero at H + `0x2C`, word 63 at frame + `0x26C`, and word `0xF000000C` at the captured destination + `0x54`.

4. **Table address versus contents.** Table base `0x12345678` and signed shifted offset 12 produce destination `0x12345684` and halfword value `0x5678`. Changing only the global source during F-000000000023 does not replace the captured base.

5. **Selector boundary.** Selector `0xFFFF` produces payload-store address `0x40070` and byte-read address `0x4FFFB`. It is not negative one. Across all selector values, those addresses range from 116 through 262,256 and from zero through 327,675, respectively.

6. **Separate attribute writes.** Attribute `0x8001` produces word `0x00008001` at owner + `0x54`, then F-000000000030, then the same word at owner + `0x64`. Simultaneous visibility and atomicity do not follow.

7. **Captured byte.** If the byte captured before F-000000000034 is `0xA7`, changing only its source memory during that call does not substitute a fresh byte for the subsequent store. This is a represented dataflow relationship, not a general dependency-preservation guarantee.

8. **Unsigned wrapping.** Attribute zero and incoming register bit pattern `0xFFFFFFF0` yield byte-store destination `0x00000009`. Writability remains unknown.

9. **Signed-shift boundary.** An F-000000000019 result of `0x20000000` has no assigned portable signed-left-shift result. Unsigned wrapping must not be silently substituted.

10. **Breakpoint stop.** If breakpoint `0x92` does not resume the sequence, post-breakpoint operations are not reached in that iteration. Treatment of earlier effects remains unresolved.

11. **Accumulated frame movement.** If two iterations reach the decrease with no other changes or restoration, the cumulative decrease is `0x1E8`. Each iteration’s frame-relative payload store precedes its own decrease.

## Uncertainties

The following remain `documented_only`:

- Dependency implementations, complete interfaces, secondary outputs, preservation, ownership, and possible state restoration.
- Correspondence between frame-like, link-like, and incoming-register state and an executable calling convention.
- The arithmetic byte’s value before its first reset.
- Memory mapping, permissions, alignment, endianness, volatility, atomicity, and device-memory behavior.
- Results of signed overflow, invalid signed shifts, and target-dependent integer-to-address interpretation. Signed address additions, frame adjustment, and the initial target calculation are not automatically modular.
- Breakpoint handling, resumption, and environmental changes.
- Whether accumulating frame movement reflects executable behavior or a limitation of the supplied representation.
- Concurrency, interrupts, external observers, and feasibility of completing an iteration.

Any higher-level initialization or state-transfer purpose is inferred intent only, not an acceptance requirement.

## Evidence

Established evidence is static inspection of the supplied function representation: 37 ordered indirect-call occurrences; explicit argument and result relationships; memory-access widths and ordering; four distinct frame-relative word locations; two reset boundaries; paired-result splitting; breakpoint placement; frame decrease; link-like assignment; and unconditional repetition.

The fixed selected-source address, selector bounds, numerical examples, and conditional next-iteration target are arithmetic deductions from that evidence. Signed-arithmetic qualifications follow from represented operand widths and signedness.

Prior prose and reviewer feedback are not independent behavioral evidence. No dependency inspection, runtime execution, or runtime tests have been performed.
